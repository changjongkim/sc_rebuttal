
We thank the reviewers and address each concern below.

## A. The parallel file system layer and the baselines (R2, R3, R4) 

When the KV cache spills out of GPU HBM, what matters is how far down it spills. Existing KV cache systems stop at node-local storage, the SSD that Mooncake and IMPRESS rely on. That approach requires SSDs on the nodes, which a diskless system does not have, and the data it holds does not persist. CASCADE goes one level deeper, to the parallel file system, a persistent, high-capacity pool, the data-lake that the faster tiers draw from. In expert and multi-tenant serving, shared data grows in value and the working set outgrows any node, so KV caching needs a persistent, high-capacity store that node-local SSDs cannot provide. Serving KV from it while masking the storage latency is the hardest part of the work. The baselines follow from this layer. PDC and HDF5 are the parallel-I/O middleware that HPC applications already use to manage data on a parallel file system, so they are what CASCADE must beat at the layer it targets. We also compare LMCache (Disk) and LMCache (Redis) as KV-system baselines. Mooncake and IMPRESS never reach this layer. We tried to add Mooncake by launching its Conductor as a separate batch job, but a persistent coordinator is non-executable under batch scheduling. IMPRESS is single-node, and its importance-informed loading optimizes prefix reuse within one node, with no cross-node sharing or global deduplication.


## B. Novelty (R4)

The novelty is not the set of mechanisms but the coordination that binds them. CASCADE is the first content-addressed, deduplicated, tiered KV cache that needs no central coordinator, which is what lets it run on diskless, batch-scheduled HPC. The insight is that a KV cache is reconstructible. A lost or stale metadata entry is only a cache miss that triggers recomputation, never a wrong read. CASCADE can therefore relax the strong consistency that general storage must enforce, and a relaxed metadata plane needs no central service. Every node keeps a local replica synchronized by off-path MPI collectives, and the data path has no blocking RPC. This is what removes the coordinator that distributed KV caches otherwise need. Mooncake does not relax this. It routes the whole system through one coordinator, the Conductor, with etcd. That recouples a disaggregated serving system, one that has already split prefill, decode, and experts into independent engines, onto a single control plane that is one bottleneck and one point of failure. CASCADE instead lets each node act as a peer, which matches how disaggregated serving is deployed. Each prior work lacks the capability CASCADE requires. vLLM prefix caching is single-node GPU HBM only. CloudMatrix384 is content-addressed but on a proprietary fabric with persistent services and an SSD tier. CachedAttention is single-node hierarchical caching. None coordinates a tiered HBM to DRAM to parallel file system hierarchy with zero-copy RDMA, global deduplication, and semantic eviction without a central coordinator, on a diskless HPC. That combination cuts the cache footprint by 93 to 94 percent and keeps retrieval latency flat where the storage baselines degrade.

## C. Correctness and notation (R2, R4)

**BlockID.** A BlockID is a SHA-256 over the cumulative token sequence from position 0 through the block boundary, so the full preceding context is in the hash input, as in vLLM and CloudMatrix384. Different prefixes give different BlockIDs. The Sec. III-B wording reads as block-local and we will correct it. The trace-driven sections use synthetic blocks to isolate the storage path, and the end-to-end track (Sec. IV-H) runs real model KV with this hashing, so the correctness claim rests on the latter. 

**Dedup Map.** A one-bit map cannot express sharedness, and we do not use it for that. The Dedup Map only skips redundant allocation on a hit, while the Semantic Prefix Registry, a replicated set of prefix-flagged BlockIDs, alone decides eviction protection. The is_shared term in Algorithm 1 conflated the two, and we will correct the notation. 

**INT8.** INT8 quantization follows cited prior work that reports preserved accuracy, and it is optional since the same path supports FP16. We will add a direct model-quality measurement in revision.

## D. Metadata, consistency, and fault tolerance (R2, R3)

**Metadata.** The Global Shard Index stores one location per BlockID, and a remote fetch makes a working copy that adds no index entry to count or invalidate. The relaxed-consistency window is safe by construction, since stale metadata is a cache miss that triggers recomputation, never a wrong read. 

**Fault tolerance.** For a cache, node failure is a non-event. A lost block is a cache miss, recomputed or read from Lustre, so only cached state is lost, not data. KV cache needs no durability, which is why we cache in the memory tiers. Where durability is wanted, CASCADE can write blocks straight to Lustre, a configuration option, not a redesign. We add no dedicated in-memory replication, which would eliminate the 93 to 94 percent footprint reduction (Fig. 12).

## E. Workloads, tiering, and hardware generality (R1, R3, R4)

**Single-turn workloads.** Prefix sharing is the dominant production pattern and our target, and we still evaluate the non-shared case. OpenOrca and CNN/DailyMail are independent queries treated as disposable blocks, and the end-to-end hit rate is 58 to 61 percent. With no sharing, deduplication is a no-op and tiering still extends capacity beyond HBM. DeepCAM extends to a non-redundant scientific domain via the Streaming path. 

**Tier count.** A three-level GPU, DRAM, and storage hierarchy is a tiered memory hierarchy, and the term does not require an arbitrary tier count. Each storage tier uses the same eviction, write, and promotion logic, so node-local NVMe is one more POSIX tier, a configuration parameter, not a redesign. The diskless testbed is the hardest case, and a system with NVMe is a strict superset. 

**Skew.** CASCADE does not place blocks by load, so a hot block draws concurrent first-fetches on its owner. Demand spreads once each requester holds a local copy, and removing the residual skew is future work, through a level of indirection like indirect inodes that maps a hot BlockID to several replicas.

## F. Metrics and scale (R1, R2, R4)

**Framework (R2, R4).** R4 is right that the absolute CASCADE-versus-vLLM TTFT in Fig. 13a mixes caching and runtime, and we do not rest the contribution on it. The gain is isolated in Fig. 13b within one PyTorch run, where retrieval replaces 1s of prefill with a 21 to 27 ms RDMA read, and the scaling difference is structural, since vLLM-APC fills HBM and falls back to full prefill while CASCADE spills to DRAM and Lustre. We will reframe Sec. IV-H and rename the trace metric to Block Retrieval Latency, distinct from end-to-end TTFT.

**Scale (R1, R4).** 256 GPUs is the maximum allocation we could obtain, in a real run. Scaling is not gated by the metadata, which is off the data path, since a block resolves in a 2.0 microsecond local lookup and a one-sided RDMA read rather than per-block metadata-server calls that grow with node count. We will reword the abstract so the evaluated scale is unambiguous.

**Statistics (R4).** We agree variance was not reported and will add it with repeated trials. The tail-latency figure is not 128 samples but 300 to 500 reads per rank, roughly 19,000 at 64 nodes, and the gaps are one to two orders of magnitude, far beyond variance.

