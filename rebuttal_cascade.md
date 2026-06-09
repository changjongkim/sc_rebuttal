# SC26 CASCADE — Rebuttal Draft v4

> Rebuttal-first framing. Correct factual errors and defend the design.
> Style: short paper-style sentences, paper vocabulary, no semicolons, no connecting dashes, no
> double quotes. Pre-trim length.

---

We thank the reviewers for the careful reading. We address each concern below.

## A. Baselines and the diskless-HPC gap (R2, R3, R4)

The absence of Mooncake and IMPRESS is due to the target environment. The constraint is not
node-local SSDs. It is the persistent, centralized control plane these systems require. Mooncake's
global scheduler (Conductor) and metadata layer run as long-lived coordination services. This matches
the persistent HTTP and etcd services noted in our Sec. IV-A. IMPRESS assumes node-local prefix
storage.

These requirements do not fit an HPC system. Leadership-scale HPC systems are batch-scheduled, so a
persistent, cross-job coordination service cannot run on them, even where node-local NVMe exists, such
as on Frontier or El Capitan. Such storage is job-scoped, not a persistent cache pool. CASCADE targets
diskless, daemon-free operation by design. Diskless computing over a shared parallel file system is a
common and challenging HPC system. It is also the hardest case for KV caching, since there is no local
storage and data spills to the parallel file system. Solving that case is the contribution.

CASCADE needs neither a daemon nor a local disk. It coordinates through MPI collectives and uses the
parallel file system as the backing store. Where node-local NVMe is present, CASCADE maps it to the
lowest tier through POSIX, so the design extends to those systems as well. HDF5 and PDC are the
representative storage baselines that run under these constraints, alongside LMCache (Disk) and
LMCache (Redis) as KV baselines. We will make this scoping explicit in the paper.

## B. Novelty relative to the named prior works (R4)

The contribution is not any single row of Table I. It is their unified coordination, which no named
system provides in this environment. Each named work lacks a capability that CASCADE requires.

- vLLM prefix caching keeps content-addressed prefixes in one node's GPU HBM. It has no cross-node
  sharing, no DRAM or PFS tiering, and no global deduplication.
- CloudMatrix384 does implement content-addressed context caching, which we acknowledge. It does so
  as a co-designed cloud platform. Its caching service (EMS) relies on a proprietary Unified Bus
  interconnect, persistent server processes under a centralized controller, and an SSD storage tier.
  CASCADE delivers the same capability on commodity diskless HPC, with no proprietary fabric, no
  persistent service, and no local disk, coordinated only by relaxed-consistency MPI metadata.
- Mooncake uses one-sided RDMA. It also requires the Conductor daemon and a node-local SSD pool.
- CachedAttention does hierarchical caching within a single node's memory hierarchy. It has no global
  content-addressed deduplication and no relaxed-consistency distributed metadata.

CASCADE is, to our knowledge, the first system to coordinate four mechanisms together. These are a
tiered memory hierarchy, zero-copy one-sided RDMA, global content-addressed deduplication, and
semantic-aware eviction. It does so on a diskless, daemon-free, batch-scheduled HPC system.
Relaxed-consistency MPI metadata keeps the data path free of blocking RPCs. The integration is the
novelty. We will add the cited works and clarify this distinction.

## C. BlockID correctness: a factual correction (R2, R4)

This concern comes from an ambiguity in our wording, not the implementation. We agree that
Sec. III-B can be read as hashing only block-local tokens, which would give different prefixes the
same BlockID. We do not do this. A BlockID is a SHA-256 over the cumulative token sequence from
position 0 through the block boundary. The full preceding context is therefore part of the hash
input. This is the standard prefix-inclusive scheme, used by vLLM and by CloudMatrix384, which also
augments its block hash with a prefix hash. Two blocks with the same local
tokens but different prefixes hash different inputs. They receive different BlockIDs. The same BlockID
cannot occur for different prefixes. The only inaccuracy is the Sec. III-B sentence, which reads as
block-local. We will rewrite it to state the cumulative formulation. That removes the concern.

On synthetic data, the two evaluation tracks serve different purposes by design. The trace-driven
sections isolate the storage and metadata path with synthetic fixed-size blocks. This is standard
methodology for storage-layer evaluation. The end-to-end track (Sec. IV-H) runs real model KV with
the cumulative hashing above. The BlockID resolves the correct block exactly, so a hit returns the
block that was stored. The INT8 compression is a separable step with bounded error. We address it
in §G.

## D. Dedup Map and the Algorithm 1 notation (R2, R3)

We agree that a one-bit existence map cannot indicate whether a block is shared. We do not use it for that. Two distinct structures exist. Dedup Map is an existence bit per BlockID. It only skips redundant allocation on a hit. Semantic Prefix Registry is an explicit set of prefix-flagged BlockIDs, replicated across nodes. It alone decides eviction protection. The is_shared(victim) term in Algorithm 1 is a pseudocode simplification. It combined the two structures. The condition that applies is membership in the Semantic Prefix Registry. We will correct the notation. The design already separates an existence bit for deduplication from a prefix set for protection. This is what keeps the metadata at about 1 byte per block.

## E. Metadata semantics, hot blocks, and consistency (R2, R3)

- **One location or many (R2).** The Global Shard Index stores a single location per BlockID. It holds the
  owning node, tier, and offset. A remote fetch creates a local working copy. It registers no new
  index entry. There is nothing to invalidate or count. This also answers R3's duplicate-copy question.
- **Eviction scope (R2).** Protection applies at both levels. GPU to DRAM is a metadata-only move of
  protected prefixes, since the compressed shadow copy already exists. DRAM to Lustre aggregates and
  writes cold, unprotected blocks (Algorithm 1).
- **Decompression cost (R2).** It is overlapped with RDMA transfer through the Staging Buffers. The full
  GET path, including decompression, is 21 to 27 ms for a 320 MB block. It replaces roughly 1 s of
  prefill. So it is not a bottleneck.
- **Bootstrap cost (R2).** The replica is built incrementally through the 100 ms delta broadcast, not a
  blocking global build. The full replica for 10M blocks is about 1.16 GB.
- **Buffer copies (R3).** The copy between the Shadow Copy Buffer and a Staging Buffer is necessary, not
  redundant. The Shadow Copy Buffer holds the immutable compressed block. Remote nodes read it by
  one-sided RDMA. A Staging Buffer is the per-GPU workspace where that block is decompressed. Keeping
  them separate lets one block be served to remote readers and decompressed locally at the same time,
  without contention. Removing the copy would serialize the two paths.
- **Hot blocks and the 100 ms window (R3).** By design, a hot block is fetched once. It is then served
  from the requesting node's local copy. Repeated demand is meant to spread across the requesting
  nodes, not stay on one node. The initial concurrent fetch is reduced because one-sided reads need
  no remote CPU and go through the Staging Buffers. We note this is an architectural argument. We did
  not run a dedicated hot-block imbalance experiment. Load-aware placement and MoE expert skew remain
  future work. On correctness, the relaxed-consistency window is safe by design. Stale metadata is
  treated as a cache miss that triggers recomputation, never a wrong read.

## F. Fault tolerance and hardware generality (R1, R3, R4)

- **Fault tolerance (R3).** CASCADE adds no dedicated in-memory replication for high availability, by design. The shadow copies and working copies it holds are caches for access, not durability replicas. When a node fails, its cached blocks are recomputed or read from the durable Tier 3 (Lustre). Cached state is lost, but the underlying data is not. Dedicated replication would eliminate the 93 to 94 percent footprint reduction (Fig. 12) that makes terabyte-scale caching feasible. So durability is handled by Lustre. HA replication of the hot tiers is a possible extension.
- **Tiering is not falsified (R4).** The tiered-hierarchy claim holds. The lowest tier is a POSIX backend
  addressed by a path. It maps directly to node-local NVMe on Frontier or to a parallel file system,
  with no code change. The diskless testbed is the hardest case. We chose it deliberately because it
  has no local storage to fall back on. The prototype implements three tiers. We will frame the
  contribution as a logical HBM to DRAM to POSIX hierarchy that extends across storage substrates.
- **Workloads without prefix sharing (R1, R3).** Prefix sharing is the dominant pattern in production LLM
  serving, through shared system prompts, RAG, and multi-turn dialogue. It is our primary target by
  design. The non-shared case is still evaluated. OpenOrca and CNN/DailyMail contribute independent
  queries that we treat as disposable, non-shared blocks alongside ShareGPT (Sec. IV-A). The
  over-subscription study floods 90 percent of the working set with disposable suffixes (Sec. IV-G).
  The end-to-end hit rate is 58 to 61 percent (Sec. IV-H), so many blocks are never reused. With no
  sharing, deduplication finds no matches and adds only the hashing cost. The tiered hierarchy still
  extends capacity beyond HBM, which is what helps a single-user long-context request that exceeds one
  node. DeepCAM is a separate point. It shows the design extends beyond LLM serving to a
  non-redundant scientific domain, through the Streaming path (Sec. IV-I).

## G. Metrics, framework isolation, and compression (R1, R2, R4)

- **TTFT terminology (R4).** The two metrics are kept distinct and are not misleading. The trace-driven metric is defined in Sec. IV-A as retrieval latency. The standard end-to-end TTFT, including prefill, is reported separately in Sec. IV-H. We will rename the trace metric to Block Retrieval Latency to remove any ambiguity. This is a labeling change, not a change to any measurement. 
- **vLLM and PyTorch (R2, R4).** We respectfully disagree that the effects cannot be separated. We do not claim our PyTorch runtime beats vLLM's fused kernels. Fig. 13b breaks TTFT down into the RDMA retrieval path and the prefill it replaces. This isolates CASCADE's data-layer contribution, independent of the runtime. Plain PyTorch is the conservative choice. It attributes the gain to retrieval, not to kernel optimizations. Decode TPOT and scheduler integration belong to the serving engine. They are the natural next step once CASCADE is integrated into one. 
- **Why vLLM-APC ON leads at small scale (R1).** In Fig. 13a, vLLM-APC ON is lower at small node counts. Its fused PagedAttention keeps the prefix KV resident in GPU HBM, which our plain PyTorch runtime does not. This advantage does not persist. As the working set exceeds single-node HBM, APC evicts and falls back to full prefill. CASCADE holds its KV across the DRAM and Lustre tiers and keeps TTFT flat (Sec. IV-H). The comparison shows that distributed caching capacity scales, not single-node kernel speed. 
- **INT8 compression (R4).** Sec. III-C applies INT8 to all blocks and cites prior work reporting that this quantization preserves inference accuracy. The same data path also supports uncompressed FP16, so compression is separable from the core contribution. We will add a direct model-quality measurement in revision as further evidence. 
- **Why CASCADE scales past the storage baselines (R1).** CASCADE resolves a block in a 2.0 microsecond local index lookup. It then reads the block with one-sided RDMA that needs no remote CPU. The baselines instead pay per-block metadata-server operations or central TCP serialization. Both costs grow with node count. This is the source of the flat scaling in Sec. IV-B and IV-C.

## H. Scale and statistical reporting (R4)

- **Evaluated scale and the abstract (R4).** The phrase about thousands of GPUs describes the target problem scale, not the evaluated configuration. The evaluation runs on 256 GPUs on a top-20 system, and we will reword the abstract to make this clear. The MPI cost is bounded by design. Each update is a 116-byte delta, batched into one MPI_Allgatherv every 100 ms off the critical path, with a replica of about 1.16 GB for 10M blocks. The synchronized volume grows with the number of updated blocks per interval, not the node count, so the collective is not the bottleneck at this scale. 
- **Statistical reporting and Fig. 7 (R4).** The 128-request workload is specific to the strong-scaling experiment in Sec. IV-B, not the tail-latency figure. The Fig. 7 runs issue 300 to 500 reads per rank, and the reported percentile aggregates these across all ranks, up to roughly 19,000 samples at 64 nodes, not 128. We agree that the submission did not report variance or per-experiment sample sizes, and we will add them. The conclusions do not depend on percentile precision. The gaps are one to two orders of magnitude, for example 16.9 times lower retrieval latency and a 469 times lower P99.9 than HDF5 at 64 nodes, far beyond any run-to-run variance. So the qualitative result holds. CASCADE removes the metadata and serialization bottlenecks.
