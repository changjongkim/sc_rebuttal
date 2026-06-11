
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

## C. Correctness and notation (R2, R4)

**BlockID.** A BlockID is a SHA-256 over the cumulative token sequence from position 0 through the block boundary, so the full preceding context is in the hash input, as in vLLM and CloudMatrix384. Different prefixes give different BlockIDs. The Sec. III-B wording reads as block-local and we will correct it. The trace-driven sections use synthetic blocks to isolate the storage path, and the end-to-end track (Sec. IV-H) runs real model KV with this hashing, so the correctness claim rests on the latter. 

**Dedup Map.** A one-bit map cannot express sharedness, and we do not use it for that. The Dedup Map only skips redundant allocation on a hit, while the Semantic Prefix Registry, a replicated set of prefix-flagged BlockIDs, alone decides eviction protection. The is_shared term in Algorithm 1 conflated the two, and we will correct the notation. 

**INT8.** INT8 compression follows cited prior work that reports preserved accuracy, and it is optional since the same path supports FP16. We will add a direct model-quality measurement in revision process.

## D. Metadata, consistency, and fault tolerance (R2, R3)

**Metadata.** The Global Shard Index stores one location per BlockID, and a remote fetch makes a working copy that adds no index entry to count or invalidate. The relaxed-consistency window is safe by construction, since stale metadata is a cache miss that triggers recomputation, never a wrong read. 

**Fault tolerance.** For a cache, node failure is a non-event. A lost block is a cache miss, recomputed or read from Lustre, so only cached state is lost, not data. KV cache needs no durability, which is why we cache in the memory tiers. Where durability is wanted, CASCADE can write blocks straight to Lustre, a configuration option, not a redesign. We add no dedicated in-memory replication, which would eliminate the 93 to 94 percent footprint reduction (Fig. 12).

## E. Workloads, tiering, and hardware generality (R1, R3, R4)

**Tier count.** A three-level GPU, DRAM, and storage hierarchy is a tiered memory hierarchy, and the term does not require an arbitrary tier count. Each storage tier uses the same eviction, write, and promotion logic, so node-local NVMe is one more POSIX tier, a configuration parameter, not a redesign. The diskless testbed is the hardest case, and a system with NVMe is a strict superset. 

**Skew.** CASCADE does not place blocks by load, so a hot block draws concurrent first-fetches on its owner. Demand spreads once each requester holds a local copy, and removing the residual skew is future work, through a level of indirection like indirect inodes that maps a hot BlockID to several replicas.

## F. Metrics and scale (R1, R2, R4)

**Framework (R2, R4).** R4 is right that the absolute CASCADE-versus-vLLM TTFT in Fig. 13a mixes caching and runtime, and we do not rest the contribution on it. The gain is isolated in Fig. 13b within one PyTorch run, where retrieval replaces 1 s of prefill with a 21 to 27 ms RDMA read, and the scaling difference is structural, since vLLM-APC fills HBM and falls back to full prefill while CASCADE spills to DRAM and Lustre. We will reframe Sec. IV-H and rename the trace metric to Block Retrieval Latency, distinct from end-to-end TTFT. 

**Scale (R1, R4).** 256 GPUs is the maximum allocation we could obtain, a real run. Scaling is not gated by the metadata, which is off the data path. We will reword the abstract so the evaluated scale is unambiguous. 

**Statistics (R4).** We agree variance was not reported and will add it with repeated trials. The tail-latency figure is not 128 samples but 300 to 500 reads per rank, roughly 19,000 at 64 nodes, and the gaps are one to two orders of magnitude, far beyond variance.
