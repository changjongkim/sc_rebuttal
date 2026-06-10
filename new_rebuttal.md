
We thank the reviewers and address each concern below.

## A. The parallel file system layer and the baselines (R2, R3, R4) 

When the KV cache spills out of GPU HBM, what matters is how far down it spills. Existing KV cache systems stop at node-local storage, the SSD that Mooncake and IMPRESS rely on. That tier is bounded by one node's capacity and is not shared across nodes, even when serving is disaggregated. CASCADE goes one level deeper, to the parallel file system, a shared and high-capacity pool, the data-lake that the faster tiers draw from. In expert and multi-tenant serving, where shared data grows in value and the working set outgrows any node, this removes both limits. Serving KV from it at memory speed is the hardest part of the work. The baselines follow from this layer. PDC and HDF5 are the parallel-I/O middleware that HPC applications already use to manage data on a parallel file system, so they are what CASCADE must beat at the layer it targets. We also compare LMCache (Disk) and LMCache (Redis) as KV-system baselines. Mooncake and IMPRESS never reach this layer. We tried to add Mooncake by launching its Conductor as a separate batch job, but a persistent coordinator is non-executable under batch scheduling. IMPRESS is single-node, and its importance-informed loading optimizes prefix reuse within one node, with no cross-node sharing or global deduplication.

## B. Novelty (R4)

The novelty is not the set of mechanisms but the coordination that binds them. CASCADE is the first content-addressed, deduplicated, tiered KV cache that needs no central coordinator, which is what lets it run on diskless, batch-scheduled HPC. The enabling insight is that a KV cache is reconstructible. A lost or stale metadata entry is only a cache miss that triggers recomputation, never a wrong answer. CASCADE can therefore relax the strong consistency that general storage must enforce, and a relaxed metadata plane needs no central service. Every node keeps a local replica synchronized by off-path MPI collectives, and the data path has no blocking RPC. This is why content-addressed deduplication and tiering can run with no daemon, where a persistent control plane cannot run at all. Mooncake does not relax this. It routes the whole system through one coordinator, the Conductor with etcd, which is the wrong abstraction twice over. It cannot run on batch HPC, and it recouples a disaggregated serving system that has already split prefill, decode, and experts into independent engines. A single coordinator forces those engines onto one control plane, one bottleneck, and one point of failure. CASCADE instead lets each node act as a peer, which matches how disaggregated serving is deployed. Each prior work lacks the capability CASCADE requires. vLLM prefix caching is single-node GPU HBM only. CloudMatrix384 is content-addressed but on a proprietary fabric with persistent services and an SSD tier. CachedAttention is single-node hierarchical caching. None coordinates a tiered HBM to DRAM to parallel file system hierarchy with zero-copy RDMA, global deduplication, and semantic eviction without a central coordinator, on diskless HPC. That combination cuts the cache footprint by 93 to 94 percent and keeps retrieval latency flat where the storage baselines degrade. 

## C. Correctness and notation (R2, R4)

**BlockID.** A BlockID is a SHA-256 over the cumulative token sequence from position 0 through the
block boundary, so the full preceding context is in the hash input, as in vLLM and CloudMatrix384.
Different prefixes give different BlockIDs. The Sec. III-B wording reads as
block-local and we will correct it.

**Dedup Map.** A one-bit map cannot express sharedness, and we do not use it for that. The Dedup Map
only skips redundant allocation on a hit, while the Semantic Prefix Registry, a replicated set of
prefix-flagged BlockIDs, alone decides eviction protection. The is_shared term in Algorithm 1
conflated the two, and we will correct the notation. INT8 quantization follows cited prior work and is optional, with FP16 supported.

## D. Metadata, consistency, and fault tolerance (R2, R3)

**Metadata.** The Global Shard Index stores one location per BlockID, and a remote fetch makes a working copy that adds no index entry to count or invalidate. The
relaxed-consistency window is safe by construction, since stale metadata is a cache miss that triggers
recomputation, never a wrong read.

**Fault tolerance.** For a cache, node failure is a non-event. A lost block is a cache miss,
recomputed or read from Lustre, so only cached state is lost, not data. KV cache needs no durability, which is why we cache in memory. Where durability is wanted, CASCADE can
write blocks straight to Lustre, a configuration option, not a redesign. We add no dedicated
in-memory replication, which would eliminate the 93 to 94 percent footprint reduction (Fig. 12).

## E. Workloads, tiering, and hardware generality (R1, R3, R4)

**Single-turn workloads.** Prefix sharing is the dominant production pattern and our primary target,
and we still evaluate the non-shared case. OpenOrca and CNN/DailyMail are independent queries treated
as disposable blocks, and the end-to-end hit rate is 58 to 61 percent. With no sharing, deduplication
is a no-op and tiering still extends capacity beyond HBM. DeepCAM extends the design to a
non-redundant scientific domain via the Streaming path.

**Tier count.** A three-level GPU, DRAM, and storage hierarchy is a tiered memory hierarchy, and the
term does not require an arbitrary tier count. Each storage tier uses the same eviction, write, and
promotion logic, so node-local NVMe is one more POSIX tier, a configuration parameter, not a redesign.
The diskless testbed is the hardest case, and a system with NVMe is a strict superset.

**Skew.** Hot blocks spread across requesters after the first fetch, and removing residual skew is
future work, through a level of indirection like indirect inodes that maps a hot BlockID to several
replicas.

## F. Metrics and scale (R1, R2, R4)

**TTFT.** The two metrics are distinct. The trace metric is defined as retrieval latency in Sec. IV-A,
and the end-to-end TTFT is reported separately in Sec. IV-H. We will rename the trace metric to Block
Retrieval Latency.

**vLLM and PyTorch.** R4 is right that the absolute CASCADE-versus-vLLM TTFT in Fig. 13a mixes caching
and runtime, and we do not rest the contribution on it. The caching gain is isolated in Fig. 13b,
within one PyTorch run, where retrieval replaces 1 s of prefill with a 21 to 27 ms RDMA read, with no
framework variable. The scaling difference is structural, since vLLM-APC fills HBM and falls back to
full prefill while CASCADE spills to DRAM and Lustre. We will reframe Sec. IV-H accordingly.

**Scale.** CASCADE resolves a block in a 2.0 microsecond local lookup and reads it by one-sided RDMA,
while the baselines pay per-block metadata-server operations that grow with node count. 256 GPUs is the maximum allocation we could obtain, a real run at that
scale, and the metadata is off the data path so it does not gate scaling. We will reword the abstract
so the evaluated 256-GPU scale is unambiguous.

**Statistics.** We agree variance was not reported and will add it with repeated trials. The
128-request workload is strong-scaling only, while the tail-latency figure aggregates 300 to 500 reads
per rank to roughly 19,000 samples at 64 nodes. The gaps are one to two orders of magnitude, far beyond variance.
