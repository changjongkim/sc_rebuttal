
We thank the reviewers and address each concern below.

## A. The parallel file system layer and the baselines (R2, R3, R4)

When the KV cache spills out of GPU HBM, how far down it spills is the point. Existing KV cache systems
stop at node-local storage, a fast local layer. CASCADE goes one level deeper, to the parallel file
system, which is shared and persistent across the cluster. In multi-tenant and expert serving, shared
data grows in value, node-local storage is capacity-limited and not shared, and
shareability requires the data to persist in one shared place. The parallel file system is that shared
data-lake, and serving KV from it at memory speed is the hardest part of the work.

The baselines follow from this layer. PDC and HDF5 are the HPC parallel-I/O frameworks that manage
data at the parallel file system layer CASCADE targets, so they are the meaningful comparison, with
LMCache (Disk) and LMCache (Redis) as KV-system baselines. Mooncake and IMPRESS never reach this
layer, operating only at node-local storage. They also cannot run here, since Mooncake needs a
persistent Conductor with etcd that batch-scheduled HPC cannot host, and IMPRESS is single-node on a
local SSD that a diskless node lacks.

## B. Novelty (R4)

The novelty is a decentralized, daemon-free, content-addressed KV cache that reaches the parallel file
system data-lake on commodity HPC. The central choice is decentralized coordination. Disaggregated
expert-LLM serving separates prefill, decode, and experts, so the cache layer should match, with no
single coordinator. CASCADE uses relaxed-consistency MPI metadata, where every node holds a local
replica and the data path has no blocking RPC and no central service. Mooncake instead unifies the
system under one coordinator, the Conductor with etcd, which is wrong for HPC and a bottleneck for
disaggregated serving.

Each prior work also lacks a capability CASCADE requires. vLLM prefix caching is single-node GPU HBM
only. CloudMatrix384 is content-addressed but on a proprietary fabric with persistent services and an
SSD tier. CachedAttention is single-node hierarchical caching. None coordinate a tiered HBM to DRAM to
parallel file system hierarchy with zero-copy RDMA, global deduplication, and semantic eviction,
decentralized, on diskless HPC. The integration is the novelty.

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
