
We thank the reviewers and address each concern below.

## A. The parallel file system layer and the baselines (R2, R3, R4) 

When the KV cache spills out of GPU HBM, how far down it spills is the point. vLLM keeps it in one node's HBM. Mooncake disaggregates it into a pool of the cluster's DRAM and SSD under a central coordinator. CASCADE goes one level deeper, to the parallel file system, the persistent and shared data-lake that the faster tiers draw from. In expert and multi-tenant serving, where shared data grows in value and the working set outgrows any node, that persistent shared store is what KV caching needs. We target diskless HPC, which offers the parallel file system but hosts no persistent service, and serving KV from it at memory speed is the hardest part of the work. The baselines follow from this layer. PDC and HDF5 are the parallel-I/O middleware that HPC applications already use to manage data on a parallel file system, so they are what CASCADE must beat at the layer it targets. 

We also compare LMCache (Disk) and LMCache (Redis) as KV-system baselines. Mooncake's Conductor is a persistent, always-on global scheduler, and a batch-scheduled system cannot host one. A job holds a time-bounded node allocation that is released on exit, with no persistent cross-job service. The exclusion is structural, not a comparison we declined. IMPRESS is single-node, and its importance-informed loading optimizes prefix reuse within one node, with no cross-node sharing or global deduplication.


## B. Novelty (R4)

The novelty is not the set of mechanisms but the coordination that binds them. To our knowledge, CASCADE is the first content-addressed, deduplicated, tiered KV cache that needs no central coordinator, which is what lets it run on diskless, batch-scheduled HPC. The insight is that a KV cache is reconstructible. A lost or stale metadata entry is only a cache miss that triggers recomputation, never a wrong read. CASCADE can therefore relax the strong consistency that general storage must enforce, and a relaxed metadata plane needs no central service. Every node keeps a local replica synchronized by off-path MPI collectives, and the data path has no blocking RPC. This is what removes the coordinator that distributed KV caches otherwise need. Mooncake does not relax this. It routes the whole system through one coordinator, the Conductor, with etcd. That recouples a disaggregated serving system, one that has already split prefill, decode, and experts into independent engines, onto a single control plane that is one bottleneck and one point of failure. 

CASCADE instead lets each node act as a peer, which matches how disaggregated serving is deployed. Each prior work lacks the capability CASCADE requires. vLLM prefix caching is single-node GPU HBM only. CloudMatrix384 is content-addressed but on a proprietary fabric with persistent services and an SSD tier. CachedAttention is single-node hierarchical caching. None coordinates a tiered HBM to DRAM to parallel file system hierarchy with zero-copy RDMA, global deduplication, and semantic eviction without a central coordinator, on a diskless HPC. That combination cuts the cache footprint by 93 to 94 percent and keeps retrieval latency flat where the storage baselines degrade.


## C. Workload generality (R1, R3, R4)

CASCADE helps two kinds of workloads through two independent mechanisms. Deduplication removes redundancy when prefixes are shared, and the tiered hierarchy extends capacity for any request that outgrows HBM, shared or not. The storage and coordination layer is workload-independent, and deduplication is an additive gain on top. Prefix sharing is the dominant pattern in production serving, through shared system prompts, RAG, and multi-turn dialogue, so it is the common case, not a niche. We still evaluate the non-shared case. OpenOrca and CNN/DailyMail contribute independent queries treated as disposable blocks alongside ShareGPT, the over-subscription study floods 90 percent of the working set with disposable suffixes, and the end-to-end hit rate is 58 to 61 percent, so a large fraction of blocks are never reused. When nothing is shared, deduplication finds no matches and adds only the hashing cost, while the tiered hierarchy still serves a single long-context request that exceeds one node's HBM by spilling to DRAM and the parallel file system. DeepCAM shows the same design reaches beyond LLM serving, to a non-redundant domain, through the Streaming path.

## D. Hot-block skew (R3) 

CASCADE does not place blocks by load, so a hot block draws concurrent first-fetches on its owning node. Demand then spreads, since each requester serves from its local copy after the first fetch, but the initial burst is real and we do not yet smooth it. Removing this residual skew is future work, through indirect hashing, in the spirit of indirect inodes, that resolves a hot BlockID to one of several replicas.

## E. Design clarifications (R2, R3, R4) 

**BlockID.** A BlockID is a SHA-256 over the cumulative token sequence from position 0 through the block boundary, so the preceding context is in the hash and different prefixes give different BlockIDs, as in vLLM. The Sec. III-B wording reads as block-local and we will correct it. Correctness rests on the end-to-end track (Sec. IV-H), which uses real model KV, not the synthetic trace-driven blocks. 

**Dedup Map and metadata.** The Dedup Map is a one-bit existence map that only skips redundant allocation, while a separate replicated Semantic Prefix Registry decides eviction protection. Algorithm 1's is_shared conflated the two, which we will correct. The Global Shard Index stores one location per BlockID, and a remote fetch adds no index entry, so there is nothing to count or invalidate.

**INT8.** INT8 is optional, since the same path supports FP16, and we will report its model-quality impact in revision.

## F. Fault tolerance and hardware generality (R3, R4) 

**Fault tolerance.** For a cache, node failure is a non-event. A lost block is a cache miss, recomputed or read from Lustre, so only cached state is lost. KV cache needs no durability, and where it is wanted CASCADE can write blocks straight to Lustre, a configuration option. We add no in-memory replication, which would undo the 93 to 94 percent footprint reduction. 

**Tiering.** A three-level GPU, DRAM, and storage hierarchy is a tiered memory hierarchy, and the term does not require an arbitrary tier count. Each storage tier uses the same eviction, write, and promotion logic, so node-local NVMe is one more POSIX tier, a configuration parameter, not a redesign. A system with NVMe is a strict superset of the diskless case we evaluate.
