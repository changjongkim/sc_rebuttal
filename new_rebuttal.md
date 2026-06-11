
We thank the reviewers and respond below.

## A. Baselines and the parallel file system (R2, R3, R4)

When the KV cache spills out of GPU HBM, how far down it spills is the point. vLLM keeps it in one node's HBM. Mooncake disaggregates it into a cluster pool of DRAM and SSD under a central coordinator. CASCADE goes one level deeper, to the parallel file system, the shared data-lake that the faster tiers draw from. In expert and multi-tenant serving, shared data grows in value and the working set outgrows any node, so this shared, high-capacity store is what KV caching needs. We target diskless HPC, which offers the parallel file system but hosts no persistent service. The baselines follow from this layer. PDC and HDF5 are the parallel-I/O middleware HPC applications already use on a parallel file system, the baselines at the layer CASCADE targets.

We also compare LMCache (Disk) and LMCache (Redis) as KV-system baselines. Mooncake's Conductor is a persistent global scheduler a batch-scheduled system cannot host, since a job holds a time-bounded allocation released on exit with no cross-job service. The exclusion is structural, not a comparison we declined. IMPRESS is single-node, optimizing prefix reuse within one node with no cross-node sharing or global deduplication.

## B. Novelty (R4)

The novelty is the coordination that binds the mechanisms. To our knowledge, CASCADE is the first content-addressed, deduplicated, tiered KV cache that needs no central coordinator, which lets it run on diskless, batch-scheduled HPC. The insight is that a KV cache is reconstructible. A lost or stale entry is only a cache miss, never a wrong read. CASCADE can therefore relax the strong consistency that general storage must enforce, and a relaxed metadata plane needs no central service. Every node keeps a local replica synchronized by off-path MPI collectives, with no blocking RPC on the data path. This removes the coordinator that distributed KV caches otherwise need. Mooncake does not relax this, routing the whole system through one coordinator, the Conductor with etcd. That recouples a disaggregated serving system, already split into independent prefill, decode, and expert engines, onto one control plane, a bottleneck and single point of failure.

Each prior work lacks the capability CASCADE requires. vLLM prefix caching is single-node GPU HBM only. CloudMatrix384 is content-addressed but on a proprietary fabric with persistent services and SSD. CachedAttention is single-node hierarchical caching. None coordinates a tiered HBM to DRAM to parallel file system hierarchy with zero-copy RDMA, global deduplication, and semantic eviction without a central coordinator, on diskless HPC. That combination cuts the cache footprint by 93 to 94 percent and keeps retrieval latency flat where the storage baselines degrade.

## C. Workload generality (R1, R3, R4)

CASCADE helps two kinds of workloads through two independent mechanisms. Deduplication removes redundancy when prefixes are shared, and the tiered hierarchy extends capacity for any request that outgrows HBM, shared or not. The storage and coordination layer is workload-independent, with deduplication an additive gain. Prefix sharing is the dominant production pattern, through shared system prompts, RAG, and multi-turn dialogue, the common case, not a niche. We still evaluate the non-shared case. OpenOrca and CNN/DailyMail are independent queries treated as disposable blocks, the over-subscription study floods 90 percent with disposable suffixes, and the hit rate is 58 to 61 percent. When nothing is shared, deduplication is a no-op and the tiered hierarchy still serves a single long-context request beyond one node's HBM, spilling to DRAM and the file system. DeepCAM extends this to a non-redundant scientific domain.

## D. Hot-block skew (R3)

CASCADE does not place blocks by load, so a hot block draws concurrent first-fetches on its owning node. Demand spreads after the first fetch, when each requester serves from its local copy, but the initial burst is real and we do not yet smooth it. Removing this residual skew is future work, through indirect hashing, in the spirit of indirect inodes, that resolves a hot BlockID to one of several replicas.

## E. Design clarifications (R2, R3, R4)

**BlockID.** A BlockID is a SHA-256 over the cumulative token sequence from position 0, so the preceding context is in the hash and different prefixes give different BlockIDs, as in vLLM. The Sec. III-B wording reads as block-local and we will correct it. The end-to-end evaluation (Sec. IV-H) runs this on real model KV, not synthetic blocks.

**Dedup Map and metadata.** The Dedup Map is a one-bit existence map for skipping redundant allocation, while a replicated Semantic Prefix Registry decides eviction protection. Algorithm 1's is_shared conflated the two, which we will correct. The Global Shard Index stores one location per BlockID, and a remote fetch adds no entry to count or invalidate.

**INT8.** INT8 is optional, with FP16 supported, and we will report its quality impact in revision.

**Framework.** The trace-driven metric we labeled TTFT measures block retrieval, and we will rename it to Block Retrieval Latency. The standard end-to-end TTFT is the one in Fig. 13. R4 is right that the absolute CASCADE vs. vLLM comparison there mixes caching and runtime, which we do not rest on. The gain is isolated in Fig. 13b, one PyTorch run where retrieval replaces 1 s of prefill with a 21 to 27 ms RDMA read. The tail-latency figure uses 300 to 500 reads per rank, not 128, with gaps far beyond variance.

## F. Fault tolerance and tiering (R3, R4)

**Fault tolerance.** For a cache, node failure is a non-event. A lost block is a cache miss, recomputed or read from Lustre, so only cached state is lost. KV cache needs no durability, and CASCADE can optionally write blocks straight to Lustre. We add no in-memory replication, which would undo the 93 to 94 percent footprint reduction.

**Tiering.** A three-level GPU, DRAM, and storage hierarchy is a tiered memory hierarchy, and the term does not require an arbitrary tier count. Each storage tier uses the same eviction, write, and promotion logic, so node-local NVMe is one more POSIX tier, a configuration parameter, not a redesign. The diskless testbed is the hardest case, making systems with NVMe a strict superset.
