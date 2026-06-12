
We thank the reviewers and respond below.

## A. Baselines and the PFS (R2, R3, R4)

When the KV cache spills out of GPU HBM, how far down it spills dictates the architectural constraints. vLLM keeps it in one node's HBM. Mooncake disaggregates it into a cluster pool of DRAM and SSD. CASCADE goes one level deeper, to the parallel file system, the shared data-lake that the faster tiers draw from. In expert and multi-tenant serving, shared data grows in value and the working set outgrows any node, so this shared, high-capacity store is what KV caching needs. The baselines follow from this layer. PDC and HDF5 are the parallel-I/O middleware HPC applications already use on a parallel file system, and LMCache (Disk) and LMCache (Redis) are the production KV-cache systems. 

The harder constraint is the platform. A batch-scheduled HPC system grants a job a time-bounded allocation released on exit, so no persistent cross-job service can run, and coordination must be daemon-free. Mooncake's Conductor is an always-on global scheduler that a batch allocation cannot host, so the exclusion is structural, not a comparison we declined.

## B. Novelty (R4)

The novelty is the daemon-free coordination that binds the mechanisms. To our knowledge, CASCADE is the first content-addressed, deduplicated, tiered KV cache that needs no central coordinator, which lets it run on batch-scheduled HPC. The insight is that a KV cache is reconstructible. A lost or stale entry is only a cache miss, never a wrong read. CASCADE can therefore relax the strong consistency that general storage must enforce, and a relaxed metadata plane needs no central service. Every node keeps a local replica synchronized by off-path MPI collectives, with no blocking RPC on the data path. This removes the coordinator that distributed KV caches otherwise need.
Each prior work lacks a capability CASCADE requires.

- Mooncake uses one-sided RDMA but routes every request through a central, persistent Conductor, which recouples disaggregated serving into one unscalable control plane and cannot run on batch-scheduled HPC.
- vLLM prefix caching is single-node GPU HBM only.
- CloudMatrix384 is content-addressed and deduplicated but only on a proprietary fabric, coordinated by a centralized control plane and resident per-node services.
- CachedAttention is a single-node hierarchical KV cache for multi-turn conversations, with no cross-node sharing or global deduplication.
- IMPRESS loads important prefix KV across a single node's GPU, CPU, and local SSD, with no cross-node sharing or global deduplication.

CASCADE alone coordinates a tiered HBM to DRAM to PFS hierarchy with zero-copy RDMA, global deduplication, and semantic eviction without a central coordinator, on batch-scheduled HPC. That combination cuts the cache footprint by 93 to 94 percent and keeps retrieval latency flat where the storage baselines degrade.


## C. Workload generality (R1, R3, R4)

CASCADE helps two kinds of workloads through two independent mechanisms. Deduplication removes redundancy when prefixes are shared, and the tiered hierarchy extends capacity for any request that outgrows HBM, shared or not. The storage and coordination layer is workload-independent, with deduplication an additive gain. Prefix sharing is the dominant production pattern, through shared system prompts, RAG, and multi-turn dialogue, the common case, not a niche. We still evaluate the non-shared case. OpenOrca and CNN/DailyMail are independent queries treated as disposable blocks, the over-subscription study floods 90 percent with disposable suffixes, and the hit rate is 58 to 61 percent. When nothing is shared, deduplication is a no-op and the tiered hierarchy still serves a single long-context request beyond one node's HBM, spilling to DRAM and the file system. DeepCAM extends this to a non-redundant scientific domain.

## D. Hot-block skew (R3)

CASCADE does not place blocks by load, so a hot block draws concurrent first-fetches on its owning node. Demand spreads after the first fetch, when each requester serves from its local copy, but the initial burst is real and we do not yet smooth it. Removing this residual skew is future work, through indirect hashing, in the spirit of indirect inodes, that resolves a hot BlockID to one of several replicas.

## E. Design clarifications (R2, R3, R4)

**BlockID.** A BlockID is a SHA-256 over the cumulative token sequence from position 0, so the preceding context is in the hash and different prefixes give different BlockIDs, as in vLLM. The Sec. III-B wording reads as block-local and we will correct it. The end-to-end evaluation (Sec. IV-H) runs this on real model KV, not synthetic blocks.

**Dedup Map and metadata.** The Dedup Map is a one-bit existence map for skipping redundant allocation, while a replicated Semantic Prefix Registry decides eviction protection. Algorithm 1's is_shared conflated the two, which we will correct. The Global Shard Index stores one location per BlockID, and a remote fetch adds no entry to count or invalidate.

**INT8.** INT8 is optional, with FP16 supported, and we will report its quality impact in revision.

**Framework.** The trace-driven metric we labeled TTFT measures block retrieval, and we will rename it to Block Retrieval Latency. The standard end-to-end TTFT is the one in Fig. 13. R4 is right that the absolute CASCADE vs. vLLM comparison there mixes caching and runtime, on which we do not rely. The gain is isolated in Fig. 13b, one PyTorch run where retrieval replaces 1 second of prefill with a 21 to 27 ms RDMA read. The tail-latency figure uses 300 to 500 reads per rank, not 128, with gaps far beyond variance.

## F. Fault tolerance and tiering (R3, R4)

**Fault tolerance.** We design CASCADE as a cache and treat its data as temporary and non-persistent, so node failure is a non-event. A lost block is a cache miss that the system recomputes or reads from the durable Lustre tier, and only cached state is lost, never the underlying data. We add no in-memory replication, which would undo the 93 to 94 percent footprint reduction, though CASCADE can write blocks straight to Lustre when durability is wanted.

**Tiering.** While CASCADE is implemented for three tiers, this is not a design decision, just an implementation. Our primary contribution lies in efficient data movement and access, and this contribution can be easily extended to additional data tiers with standard APIs such as POSIX which we already use. We acknowledge the hardcoded implementation, but this does not fundamentally limit the design. The diskless testbed is the hardest case, making systems with NVMe a strict superset.
