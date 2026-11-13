# Uncovering the Memory Hierarchy of Modern NVIDIA GPUs for Accurate Simulation

Ziwen Wang ${}^1$ , Jiapeng Zhou ${}^1$ , Changxi Liu ${}^{1, *}$ , Dongsheng Wang ${}^2$ , Yun Chen ${}^{1, *} \; {}^1$ The Hong Kong University of Science and Technology (Guangzhou), ${}^2$ Tsinghua University \{zwang578, jzhou418\}@connect.hkust-gz.edu.cn, \{changxiliu, yunchen\}@hkust-gz.edu.cn, wds@tsinghua.edu.cn *Corresponding authors

Abstract-Emerging artificial intelligence workloads, particularly Large Language Models (LLMs), place unprecedented demands on GPU memory systems. Consequently, architectural innovations increasingly target the memory system to alleviate these bottlenecks. However, existing works still rely on outdated memory hierarchy models that fail to capture the complexities of modern hardware. This fidelity gap leads to inaccurate characterization and could misguide optimization strategies.

To bridge this gap, we present GemAtlas, the first systematic reverse engineering of the memory hierarchy in modern NVIDIA GPUs. By developing fine-grained SASS-level benchmarking, we expose critical, previously undocumented micro-architectural details across different cache levels, including cache set associativity, slice-routing and set-indexing hash functions, replacement policies, coherence behavior, and interconnect topologies.

We integrate these discovered micro-architecture details into the state-of-the-art GPU simulation framework. The final results demonstrate that GemAtlas reduces the mean absolute percentage error (MAPE) from 31.69% to 22.24% on average across four GPUs. Specifically, GemAtlas reduces the MAPE from 34.04% to 23.98% for real-world LLM operators.

## I. INTRODUCTION

In recent years, GPUs have evolved from graphics accelerators into general-purpose computing platforms and are now widely used for throughput-oriented workloads, including bioinformatics [7], [38], molecular dynamics [31], [34], reacting-flow simulation [44], and deep learning [8], [37], [42]. As GPU computing capabilities continue to advance, improvements in the memory system have not kept pace, making memory performance an increasingly important bottleneck [15].

More recently, LLM training and inference have become the primary workloads on modern GPUs. As model sizes and context lengths increase, the memory footprint of model parameters and the KV cache imposes high bandwidth and capacity demands on the memory subsystem [9], [29]. Consequently, overall performance of GPU system has become increasingly sensitive to memory hierarchy behavior and data path efficiency.

However, the memory hierarchy of modern commercial GPUs remains largely opaque at the microarchitectural level. Existing reverse-engineering studies on modern GPUs have focused on SM-core internals, providing detailed analyses of scheduling and execution behavior [18]. In contrast, reverse engineering of the memory hierarchy remains fragmented and coarse-grained [1], [24], [28], [35], [51], [54], making it difficult to explain memory-dependent performance behavior on real hardware and perform fine-grained performance optimization.

To address this gap, we present GemAtlas, a unified reverse-engineering and modeling framework dedicated to the memory hierarchy of modern GPUs. GemAtlas characterizes key memory-hierarchy behaviors at cache-set granularity across the L1, L2 data cache, and constant-memory hierarchies (L0, L1, and L1.5). Table 1 summarizes our specific reverse-engineering contributions across these levels, detailing newly exposed set associativity, topologies, mapping policies, and replacement mechanisms. To validate our findings and evaluate their impact, we incorporate them into AccelSim, a widely used GPU simulator [18], [26]. Evaluation on a benchmark suite spanning traditional GPU benchmarks and real-world LLM operators shows substantially improved fidelity, reducing the average MAPE over four GPUs from 31.69% to 22.24%, and from 34.04% to 23.98% for the LLM operators, with particularly large gains for memory-sensitive workloads, whose MAPE decreases from 36.33% to 20.61%. In summary, GemAtlas makes the following contributions:

- We reverse-engineer the L1 data cache/shared memory microarchitecture on H20, H100, A800, and RTX 4090 and disclose its sliced organization, slice routing and cache set indexing, tag-data asymmetry, and replacement policy.

- We reconstruct the complete topology of the distributed Hopper L2 cache across partition, Frame Buffer Partitions (FBP) and slice levels; derive the memory-interleave hash; expose coherence/replacement behavior; and introduce an efficient hash-guided eviction-set construction method for modern GPUs.

- We systematically dissect the constant memory hierarchy across L0, L1, and GPC-level L1.5 cache, recovering hierarchy organization, cache-set hash indexing rules, and replacement policy.

- We develop a benchmark suite, GemAtlas-Bench, that extends GPU simulation beyond traditional workloads to include real-world LLM operators drawn from vLLM [27] and PyTorch [43].

- We integrate the recovered mechanisms into AccelSim [26], a widely used GPU simulator, and validate the model against four GPUs (H20, H100, A800, and RTX 4090) using GemAtlas-Bench, substantially improving accuracy for LLM operators and memory-sensitive workloads.

TABLE I

CONTRIBUTION MAP OF REVERSE-ENGINEERED MECHANISMS ON H20 AND H100 ACROSS CACHE LEVELS.

<table><tr><td colspan="2">Reverse-Engineered Mechanism</td><td>L1 D-Cache/SMEM</td><td>L2 Cache</td><td>L0 C-Cache</td><td>L1 C-Cache</td><td>GCC (L1.5)</td></tr><tr><td rowspan="3">Structure</td><td>Hierarchy Topology</td><td>✓ N</td><td>✓</td><td>-</td><td>-</td><td>✘</td></tr><tr><td>Capacity</td><td>✓ I</td><td>✘</td><td>✓ N</td><td>✓</td><td>✓</td></tr><tr><td>Geometry (line/set/way)</td><td>✓ I</td><td>✓</td><td>✓ N</td><td>✓</td><td>✓</td></tr><tr><td rowspan="4">Mapping & Policy</td><td>Physical Routing (slice) Hash Function</td><td>✓ N</td><td>✓</td><td></td><td>-</td><td>✘</td></tr><tr><td>Cache Set Index Hash Function</td><td>✓ N</td><td>✘</td><td>✓ N</td><td>✓</td><td>✓N</td></tr><tr><td>Cache Set Replacement Policy</td><td>✓ N</td><td>✓</td><td>✓ N</td><td>✓N</td><td>✘</td></tr><tr><td>Cache-Set-Level Cross-Partition Coherence</td><td>-</td><td>✓N</td><td>-</td><td>-</td><td>-</td></tr><tr><td>Validation &</td><td>Latency & Data-Path Characterization</td><td>✓</td><td>✓ $\mathbf{I}$</td><td>✓ I</td><td>✓</td><td>✓</td></tr><tr><td>Utility</td><td>Eviction-set Construction Support</td><td>✓N</td><td>✓N</td><td>✓ N</td><td>✓N</td><td>✘</td></tr><tr><td></td><td>Legend: $\;\rho^{\mathrm{N}}$ New (first reverse-engineered) ✓ Improved over prior work</td><td>✓ Not covered</td><td>- Not applicable</td><td></td><td></td><td></td></tr></table>

## II. BACKGROUND AND MOTIVATION

GPU Architecture and Memory Hierarchy. Modern NVIDIA GPUs are organized hierarchically into Graphics Processing Clusters (GPCs), Streaming Multiprocessors (SMs), and SM-subpartitions (SMSPs) [32], [40]. In the CUDA programming model, a kernel is decomposed into thread blocks, blocks are scheduled onto SMs, and warps are the basic scheduling unit within an SMSP. The on-chip memory hierarchy consists of three parts. Each SM has a unified L1 cache that integrates data caching, shared memory (SMEM), and texture functionality [40]. A dedicated constant-memory hierarchy provides access paths for read-only data through the Indexed Constant Cache (IDC), Data Constant Cache (DCC), Immediate Constant Cache (IMC), and GPC Constant Cache (GCC) [41]. All SMs access a shared L2 cache through the on-chip network (NoC). This L2 is distributed across partitions, each connected to a subset of GPCs [24], [33]. Each partition contains multiple Frame Buffer Partitions (FBPs) that interface with the memory controllers directly, and each FBP groups multiple independently addressed L2 slices.

Undocumented Memory Hierarchy Details. Modern GPU workloads, particularly LLMs, are increasingly bottlenecked by memory hierarchy behavior [15], and architecture-specific optimizations that explicitly exploit it have delivered substantial performance gains [11], [12]. Newer GPU features, such as thread block clusters with multicast, L2 cache residency control, and the Tensor Memory Accelerator (TMA), also build on this hierarchy. Understanding the memory hierarchy is therefore a prerequisite both for optimizing these workloads and for studying and modeling these features. Yet the memory hierarchy remains opaque. Official documentation describes only coarse-grained properties, such as cache capacities [40]. Prior reverse-engineering studies go further and measure finer-grained properties, such as line sizes, latencies, and the associativity of some caches [1], [21]-[24], [35], [51], [54]. However, for the large, sliced caches of recent GPUs, set-level mechanisms, such as address mapping and replacement policies, remain unknown. As a result, developers cannot reliably predict the performance impact of high-level optimizations, making performance tuning inherently empirical.

In addition, the lack of microarchitectural understanding limits architecture research. Existing GPU simulators [5], [18], [26] rely on simplified or outdated memory-hierarchy models and have been evaluated primarily on traditional benchmark suites. Although some evaluations include GEMM kernels, modern LLM workloads comprise a much broader range of operators with diverse memory behavior, leaving their performance simulation unexplored. Our evaluation shows that existing simulators are inaccurate on these operators:

On the real-world LLM operators, the newest Accel-Sim GPU simulator [18], [26] has 39.91% MAPE, with MAPE reaching 56.66% for the 11 memory-sensitive LLM operators. Consequently, the fidelity of existing simulators on modern GPU workloads remains uncertain, undermining the reliability of architecture exploration.

Together, these limitations motivate systematic reverse engineering of the modern GPU memory hierarchy and its accurate modeling in GPU simulators. By exposing the underlying mechanisms, we aim to bridge this knowledge gap and establish a high-fidelity foundation for systematic performance optimization and architectural analysis.

## III. REVERSE ENGINEERING METHODOLOGY

### A.The GemAtlas Framework

GemAtlas is a fully automated framework that performs reverse engineering end to end and feeds the recovered parameters directly into simulation. It consists of three components: (1) a SASS microbenchmark suite that implements every probe in Sections IVI-VI as a parameterized template and assembles it for the target GPU; (2) an analysis pipeline that runs the suite and applies the inference steps of Sections IV-VI to produce a parameter table for each cache level; and (3) an extended AccelSim whose memory hierarchy is configured directly from this table (Section VII). For a previously unseen GPU, the user only specifies the target architecture, and probing, inference, and simulator configuration then run without manual intervention.

Test Platform. We evaluate GemAtlas on NVIDIA H20 and H100 GPUs, both based on the Hopper microarchitecture, as well as A800 (Ampere) and RTX 4090 (Ada), using CUDA 12.8 and cuAssembler [10]. We use H20 as the primary platform for presenting results and report the results on the other GPUs in Section VI-C. GemAtlas is hardware-agnostic and can be readily extended to Blackwell and future architectures with support for their corresponding SASS ISAs.

## B. SASS-Level Microbenchmarking

CUDA provides a high-level programming abstraction for NVIDIA GPUs, where applications are written in a C-like language and compiled into PTX, a virtual ISA, before being translated into SASS, the native ISA executed by the hardware [18]. However, neither CUDA nor PTX offers sufficient control for microarchitectural characterization. Compiler optimizations in both the offline backend (ptxas) and the Just-In-Time (JIT) compiler can reorder instructions and alter register allocation, obscuring true latency and invalidating carefully constructed access patterns [16]. As a result, deterministic analysis requires direct manipulation of SASS. Modern NVIDIA GPUs also rely on a compiler-driven scoreboard mechanism rather than dynamic hardware scheduling to manage memory dependencies. SASS instructions carry explicit control codes that manage dependency barriers and dictate when subsequent instructions may issue [10], [18].

Direct SASS programming enables two key capabilities for microarchitectural characterization. First, explicit control of scoreboarding allows us to serialize dependent operations and measure memory latency with cycle-level precision. Second, fixed instruction streams ensure deterministic code footprints, which are critical for isolating cache behaviors. For example, in the GCC study (Section VI-B), we place instruction lines into a chosen GCC set with a chain of branches to induce instruction-driven cache evictions. Because NVIDIA does not provide an official SASS assembler, we build our microbench-marks upon the open-source cuAssembler [10]. We extend its Ampere (SM86) support to Hopper (SM90) by updating the ELF parser and packer for the newer cubin format [39], thereby enabling direct SASS execution and microbenchmark construction on Hopper GPUs.

## C. Case Study: L1 Eviction Set Construction

An eviction set is a group of addresses that map to the same cache set, forming the basis for recovering cache indexing and replacement behavior. As a case study to validate the effectiveness of our SASS-level microbenchmarking framework, we construct L1 eviction sets using the Group Elimination algorithm [53], but implemented entirely at the SASS level. The algorithm iteratively prunes a candidate pool while preserving its ability to evict a target line. Each iteration dynamically generates a SASS sequence that accesses the target address $T$ followed by the candidate pool. Using this workflow, we recover stable L1 eviction sets. For example, the eviction set for target address $0 \times 0$ is $\{ 0 \times 13880,0 \times 20\mathrm{\;d}80,0 \times 33500$ , $0 \times 4bb00\}$ . Figure 1 shows the latency of re-accessing $0 \times 0$ as these lines are accessed one by one: $0 \times 0$ remains an L1 hit (31 cycles) after three of them and becomes a miss after the fourth, whereas lines from another set never evict it. The minimal cardinality of four indicates 4-way associativity and provides the basis for the L1 analyses in the next section.

![Fig. 1. Latency of re-accessing $0 \times 0$ on $\mathrm{H}20$ after accessing $k$ lines from its eviction set (same set) or from the eviction set of $0 \times 80$ (different set). Each point is the median of 400 runs.](images/fig01.jpg)

Fig. 1. Latency of re-accessing $0 \times 0$ on $\mathrm{H}20$ after accessing $k$ lines from its eviction set (same set) or from the eviction set of $0 \times 80$ (different set). Each point is the median of 400 runs.

```
Listing 1. SASS code for an eviction test kernel.
```

<table><tr><td></td><td>[B0-----:R-:W-:-:S04]</td><td>NOP;</td></tr><tr><td></td><td>2 [B0-----:R-:W0:-:S05]</td><td>LDG.ERZ, [R2.64 + T] ;</td></tr><tr><td>3</td><td>[B0-----:R-:W0:-:S05]</td><td>LDG.ERZ, [R2.64 + CANDIDATE_1] ;</td></tr><tr><td></td><td>[B0-----:R-:W0:-:S05]</td><td>LDG.E RZ, [R2.64 + CANDIDATE_2] ;</td></tr><tr><td></td><td>[B0-----:R-:W0:-:S05]</td><td>LDG.ERZ, [R2.64 + CANDIDATE_3] ;</td></tr><tr><td></td><td>[B0-----:R-:W0:-:S05]</td><td>LDG.ERZ, [R2.64 + CANDIDATE_4] ;</td></tr><tr><td></td><td>[B0-----:R-:W-:-:S04]</td><td>S2R R6, SR_CLOCKLO;</td></tr><tr><td>8</td><td>[B------:R-:W0:-:S05]</td><td>LDG.ERZ, [R2.64 + T] ;</td></tr><tr><td></td><td>[B0-----:R-:W1:-:S04]</td><td>S2R R7, SR_CLOCKLO;</td></tr><tr><td>10</td><td>[B-1-----:R-:W2:-:S04]</td><td>IADD3 R6, R7, -R6, RZ ;</td></tr></table>

Listing [1] shows a representative eviction-test kernel. The kernel operates in three phases: (1) prime, loading $T$ into the L1 data cache; (2) evict, accessing candidate addresses to create set contention; and (3) probe, measuring the latency of re-accessing $T$ using SR_CLOCKLO and IADD3 to classify hits and misses. This execution order is enforced by the scoreboard dependency chain encoded in the bracketed SASS control fields. During traversal, each LDG. E allocates a dependency via W0, while the following instruction waits on it via B0, thereby serializing the loads. The probe path uses the same dependency chain, ensuring that the timing instructions execute only after the re-access to $T$ completes, and their operands are ready. This explicit control yields cycle-accurate hit/miss timing. We use the same SASS-level methodology to construct the eviction sets needed for reverse engineering the L2 data cache and the constant-memory hierarchy in the following sections.

### IV.L1 DATA CACHE

Modern NVIDIA GPUs use a unified memory structure that dynamically partitions capacity among the L1 data cache, shared memory (SMEM), and texture cache. In this section, we use SASS microbenchmarks to recover the organization of the L1 data cache within this structure. We find that the L1 consists of four slices of 128 4-way sets each, indexed by an XOR hash of the virtual address. It keeps a fixed tag array while its data array shrinks with larger SMEM allocations, and it uses a Segmented LRU (SLRU) replacement policy.

## A. Physical Organization and Tag-Data Asymmetry

We begin by characterizing three baseline L1 properties using our SASS-level methodology: line size, associativity, and hit latency. We observe a $128\mathrm{\;B}$ cache line grouped into four 32 B sectors, 4-way associativity (confirmed by a minimal eviction-set cardinality of four in Section III-C), and a 31-cycle hit latency measured with the SASS probe in Listing II.

TABLE II

PHYSICAL L1 DATA ARRAY CAPACITIES UNDER VARYING SHARED MEMORY (SMEM) CONFIGURATIONS.

<table><tr><td>Total SMEM</td><td>Total L1 Size</td><td>Slice Capacity</td><td>Data-Line Capacity</td></tr><tr><td>8 KiB</td><td>241 KiB</td><td>60.25 KiB</td><td>482</td></tr><tr><td>16 KiB</td><td>233 KiB</td><td>58.25 KiB</td><td>466</td></tr><tr><td>32 KiB</td><td>217 KiB (Default)</td><td>54.25 KiB</td><td>434</td></tr><tr><td>64 KiB</td><td>185 KiB</td><td>46.25 KiB</td><td>370</td></tr><tr><td>100 KiB</td><td>149 KiB</td><td>37.25 KiB</td><td>298</td></tr><tr><td>132 KiB</td><td>117 KiB</td><td>29.25 KiB</td><td>234</td></tr><tr><td>164 KiB</td><td>85 KiB</td><td>21.25 KiB</td><td>170</td></tr><tr><td>196 KiB</td><td>53 KiB</td><td>13.25 KiB</td><td>106</td></tr><tr><td>228 KiB</td><td>21 KiB</td><td>5.25 KiB</td><td>42</td></tr></table>

Accurate Capacity Measurement. Prior studies typically estimate L1 capacity using CUDA-level pointer-chasing mi-crobenchmarks [36], [51]. In contrast, we measure L1 capacity at cache-line granularity using the SASS-level eviction sets derived in Section III-C. For each of the nine SMEM configurations, we fill each cache set until the first capacity-driven eviction and determine the data-array capacity from the number of resident lines. As shown in Table II, the combined L1 data cache and SMEM capacity is $249\mathrm{KiB}$ in every configuration, $7\mathrm{KiB}$ below the $256\mathrm{KiB}$ unified capacity reported in official documentation [40]. Notably, under the default configuration, where the kernel requests no SMEM, the effective L1 data cache capacity is only $217\mathrm{KiB}$ , revealing an implicit $32\mathrm{KiB}$ reservation for SMEM. We discuss a possible cause of the $7\mathrm{KiB}$ gap in Section VIII.

The 4-Slice Topology. During per-set cache filling, evictions occur before the aggregate L1 capacity is reached. Moreover, the evicted lines always come from the same one-quarter of the filled sets, for example, 108 of 432 under the default configuration. This indicates that the L1 is divided into four slices, each holding one-quarter of the sets with its own tag and data arrays (Fig. 2(a)). Once the data array of one slice is full, a new line in that slice evicts a line from another set of the same slice, while the other slices still have free space. We call this a cross-set eviction and explain it with the tag-data asymmetry (Fig. 2(b)) later in this subsection. Across all SMEM configurations, the four slices have equal capacity, reported in Table II in KiB and as the number of $128\mathrm{\;B}$ cache lines per slice.

Address Indexing and XOR Hash. We then recover the L1 indexing logic that maps each virtual address to a slice and an in-slice cache set. Using eviction-set probes at a 128 B cache-line stride, we find that the 512 cache lines within a $64\mathrm{KiB}$ address span map one-to-one onto 512 distinct slice/set pairs, that is, 128 sets in each of the four slices. Each address is therefore mapped by a 9-bit logical index, consisting of a 2- bit slice index and a 7-bit in-slice set index.

![Fig. 2. Microarchitectural organization of the L1 unified cache.](images/fig02.jpg)

Fig. 2. Microarchitectural organization of the L1 unified cache.

![Fig. 3. Bit-mapping heatmap of the L1 hash-routing function.](images/fig03.jpg)

Fig. 3. Bit-mapping heatmap of the L1 hash-routing function.

Using bit-flipping analysis [14], we determine how virtual-address bits generate this 9-bit logical index. The recovered mapping is generated by XOR reductions over selected virtual-address bits drawn from $b_7$ to $b_{36}$ . Bits $b_0 - b_6$ do not participate because they encode the 128 B cache-line offset, and we do not consider bits above $b_{36}$ , which lie outside the $96\mathrm{GiB}$ address range of H20. Figure 3 summarizes the recovered bit mapping as a heatmap, where rows $I_0 - I_6$ correspond to the in-slice set index and rows $S_0$ and $S_1$ correspond to the slice index. For example, the mapping for cache set index bit $I_0$ is

$$
I_0 = b_7 \oplus b_{11} \oplus b_{15} \oplus b_{22} \oplus b_{25} \oplus b_{26} \oplus b_{27}
$$

$$
\oplus b_{30} \oplus b_{31} \oplus b_{33} \oplus b_{34} \oplus b_{36}.
$$

To validate this mapping, we apply it to the measured eviction sets of all 512 sets (15,873 addresses): every address maps to the slice and set of its target. For example, $0 \times 0$ and its eviction set in Figure 1 all map to slice 0 and set 0 .

Hardware Asymmetry: Tag Array vs. Data Array. The tag array follows this fixed logical index: its 512 4-way sets with 128 B lines cover $256\mathrm{KiB}$ (Fig. 2(b)). We recover identical XOR mappings under all SMEM configurations, so changing the SMEM allocation does not alter this organization. The data array, however, is smaller. Even in the configuration with the largest L1, each slice holds only 482 lines (60.25 KiB; Table II), fewer than the 512 lines needed to fill its 128 4-way sets, and it shrinks further as more of the unified capacity is allocated to SMEM. We refer to this mismatch between a fixed tag array and a smaller data array as the L1 tag-data asymmetry and discuss it further in Section VIII.

This asymmetry leads to three allocation cases. (1) When the target set is full (regardless of whether the data array of the slice is full), the incoming line replaces a line in the same set under the in-set replacement policy (Section IV-B). (2) When the target set has a free tag entry and the slice still has free data capacity, the line is allocated normally. (3) The key case arises when the target set has a free tag entry but the data array of the slice is full, a situation that cannot occur in a conventional cache. In this case, the hardware performs a cross-set eviction within the slice: it first selects a victim set from the sets in the slice, then evicts a line from that set using the local replacement policy, and finally allocates the new line to the target set. To determine how the victim set is selected, we repeat the per-set filling experiment 13,500 times under the default configuration and record the victim sets in each run. The victim set changes from run to run, and its selection probability depends on the fill order: among the 108 sets in the slice, the first 27 filled are selected about 2.5 times as often as the last 27. All nine SMEM configurations show the same trend. We leave the exact selection rule to future work.

### B.L1 Cache Set Replacement Policy

Our measurements show that the L1 data cache employs a Segmented LRU (SLRU) policy [13] rather than conventional unified LRU. Each set contains Probationary and Protected segments: new lines enter the Probationary Segment, while re-accesses promote them to the Protected Segment. Figure 4 summarizes the inferred state transitions.

Listing 2. SASS microbenchmark used to deduce the SLRU mechanisms.

<table><tr><td>2</td><td>[B0-----:R-:W0:-:S05]</td><td>LDG.E</td><td>RZ,</td><td>[R2.64 ADDR_A];</td></tr><tr><td>3</td><td>[B0-----:R-:W0:-:S05]</td><td>LDG.E</td><td>RZ,</td><td></td></tr><tr><td>4</td><td>[B0-----:R-:W0:-:S05]</td><td>LDG.E</td><td>RZ,</td><td>[R2.64 + ADDR_B];</td></tr><tr><td>5</td><td>[B0-----:R-:W0:-:S05]</td><td></td><td></td><td>LDG.ERZ, [R2.64 + ADDR_B]; //</td></tr><tr><td>6</td><td>[B0-----:R-:W0:-:S05]</td><td></td><td></td><td>LDG.ERZ, [R2.64 + ADDR_C]; //</td></tr><tr><td>7</td><td>[B0-----:R-:W0:-:S05]</td><td></td><td></td><td>DG.ERZ, [R2.64 + ADDR_C]; //</td></tr><tr><td>8</td><td colspan="4">// Phase 2: Streaming</td></tr><tr><td>9</td><td>[B0-----:R-:W0:-:S05]</td><td></td><td></td><td>LDG.ERZ, [R2.64 + ADDR_X]</td></tr><tr><td>10</td><td>D $\lbrack\text{ B }0 - - - - - - - - - - - - - - - - 0\rbrack$</td><td>LDG.E</td><td></td><td></td></tr><tr><td>11</td><td>[B0___ $= R - : W0 : - : S05$ ]</td><td></td><td></td><td>RZ, [R2.64 + ADD</td></tr><tr><td colspan="5">12 // Phase 3: Probing</td></tr><tr><td>13</td><td>[B0______: R-: W-: - : S04]</td><td colspan="3">S2R R6, SR_CLOCKLO;</td></tr><tr><td>14</td><td>[B______: R-: W0: - : S05]</td><td colspan="3">LDG.ERZ, [R2.64 + TARGET_T];</td></tr><tr><td>15</td><td>[B0______: R-: W1: - : S04]</td><td colspan="3">S2R R7, SR_CLOCKLO;</td></tr><tr><td>16</td><td>[B-1----: R-: W2: -: S04]</td><td colspan="3">IADD3 R6, R7, -R6, RZ;</td></tr></table>

We infer these transitions with the SASS microbenchmark in Listing 2, which selectively re-accesses target lines $(A - C)$ , streams new lines $(X - Z)$ into the same set to trigger eviction, and times a final probe to determine target residency.

Streaming-only baseline (Figure 4(a)). With all optional re-accesses disabled, the access stream is $A, B, C, X, Y, Z$ within one set. We observe that $Y$ and $Z$ evict $A$ and $B$ , respectively, consistent with LRU eviction within the Probationary Segment when no line is promoted.

Stepwise promotion into Protected (Figure 4(b, c)). We next introduce re-accesses progressively. When only $A$ is re-accessed (Figure 4(b)), $Y$ and $Z$ evict $B$ and $C$ while $A$ remains resident, showing that a re-accessed line is promoted into the Protected Segment and gains higher retention priority. When $B$ is also re-accessed (Figure 4(c)), $A$ and $B$ both remain resident while $C$ and $X$ are evicted, indicating that two promoted lines can coexist in the Protected Segment.

Priority exchange on protected overflow (Figure 4(d)). Starting from the state in Figure 4(c), we additionally re-access $C$ , creating a third promoted line and triggering protected overflow. Under subsequent streaming pressure, we observe that $Y$ evicts $X$ and then $Z$ evicts $A$ . This implies that $C$ is promoted into the Protected Segment, the LRU line in the Protected Segment $(A)$ is demoted into the Probationary Segment, and the demoted line is inserted at Probationary-MRU, since $A$ survives one insertion longer than $X$ . Because the third promotion causes demotion rather than expansion, the Protected Segment can hold at most two lines.

Immediate promotion on re-access. We further test whether promotion depends on timing or on access count. (1) Inserting a delay of up to about two million cycles before or after the re-access of $A$ does not change the outcomes in Figure 4(b, c), so a line is promoted as soon as it is re-accessed, and its promotion does not expire. (2) Re-accessing $A$ up to four times gives the same outcome as once, so extra accesses do not affect promotion. Moreover, in the overflow test of Figure 4(d), $A$ is still demoted when accessed four times and $B$ and $C$ only twice, whereas ordering by access count would demote $B$ . Hence, the Protected Segment is ordered by recency, not by access count.

Final replacement policy. Taken together, these observations indicate the following SLRU policy: (1) new lines are inserted into the Probationary Segment; (2) a re-access promotes a line into the Protected Segment, where lines have higher retention priority; (3) the two segments maintain separate recency orders, with LRU eviction in the Probationary Segment and, on overflow, the LRU line of the Protected Segment demoted to the MRU position of the Probationary Segment; and (4) the Protected Segment holds at most two lines, and the remaining ways belong to the Probationary Segment.

![Fig. 4. Inferred state transitions of the L1 SLRU policy. Evictions occur exclusively within the Probationary Segment (yellow blocks).](images/fig04.jpg)

Fig. 4. Inferred state transitions of the L1 SLRU policy. Evictions occur exclusively within the Probationary Segment (yellow blocks).

## V. Distributed L2 Cache

Modern NVIDIA GPUs (e.g., A100, H100, and B200) organize the L2 cache into two partitions, each connected to a subset of GPCs through the on-chip interconnect [24], [33]. In this section, we use SASS microbenchmarks to recover this distributed L2 on H20. We find that its 96 slices are grouped into 12 FBPs across the two partitions and are selected by an XOR hash with a 128-to-96 remapping. Using slice-constrained eviction sets built with this hash, we further characterize replacement and set-level cross-partition coherence.

![Fig. 5. L2 Cache Topology.](images/fig05.jpg)

Fig. 5. L2 Cache Topology.

## A. Partition-Level Topology and SM Affinity

Mapping SMs to GPCs and Partitions. We read both mappings directly from special registers with the SASS S2R instruction. For the SM-to-GPC mapping, SR_AFFINITY returns the GPC ID of the executing SM. Alternatively, SMs can be grouped by SR_CLOCK, which, unlike the GPU-wide SR_GLOBALTIMER, is shared only by the SMs of one GPC (e.g., two GPCs differ by 2,315,338 cycles). Both methods yield the same grouping. For the GPC-to-partition mapping, SR_SW_SCRATCH returns the same value for all SMs of a partition: 0xA0000000 for 42 SMs and 0x80000000 for the other 36. The resulting topology is asymmetric: Partition0 connects GPC0, GPC1, GPC2, and GPC4, whereas Partition1 connects GPC3, GPC5, and GPC6 (Figure 5, which also shows the SMs of GPC4).

SM-to-Partition Latency. Using the SASS probe in Listing 1, we measure the latency of L2 hits and HBM accesses through the local partition, which connects to the probing SM's GPC, and through the remote one. For L2 hits, we first warm the target line in the designated partition; for HBM accesses, we force an L2 miss. Local and remote L2 hits take $\sim 276$ and $\sim 446$ cycles, and local and remote HBM accesses take $\sim 530 - 570$ and $\sim 700 - 750$ cycles, respectively. Both therefore incur a similar $\sim 170$ -cycle cross-partition penalty.

### B.L2 Slice and FBP Organization

Determining the Number of Physical L2 Slices. The number of active physical L2 slices on H20 is determined using a contention-based microbenchmark derived from CuAssem-bler [10]. Global-memory atomic operations execute at the slice where the target cache line is located. Concurrent atomicAdd operations to two addresses in the same slice therefore contend and increase the runtime, whereas addresses in different slices execute nearly in parallel. Runtime thus indicates whether an address pair maps to the same L2 slice.

On H20, we generate 1,024 addresses spanning the range 0x00000 to 0x1FF80 (128 KiB window) with stride 0x80 (one cache line), and perform pairwise atomic contention tests over all ordered address pairs $(1024 \times 1024)$ , treating high-latency pairs as same-slice collisions. Let $N_{\text{ collide }}$ denote the number of pairs classified as collisions. If the addresses are distributed uniformly over $S$ slices, the number of same-slice ordered pairs is

$$
N_{\text{ collide }} = 1024^2/S,\;S = 1024^2/N_{\text{ collide }}.
$$

We measure $N_{\text{ collide }} = 12,288$ , which gives a slice count of $S = 85.333$ on H20. This non-integer value is inconsistent with a direct physical indexing scheme. We infer a two-stage remapping design: a 7-bit logical slice space (128 outcomes) followed by remapping onto 96 physical slices. Specifically, 64 slices are uniquely mapped (one logical index each), while the remaining 32 slices are shared by two logical indices. Under this model, the probability of same-slice collisions is $64 \times \left( 1/128\right)^2 + 32 \times \left( 2/128\right)^2 = 3/256$ , which implies an apparent slice count of $1/(3/256) = 85.333$ , exactly matching our measurement. We therefore conclude that H20 has 96 active physical L2 slices, with 48 slices per partition and $640\mathrm{KiB}$ per slice.

Identifying FBP Groupings via Cross-SM Correlation. Slice-access latency alone cannot reliably identify FBP groupings because it also depends on each slice's position within the FBP relative to the XBAR, so slices from different FBPs have overlapping latency ranges [24]. Bandwidth contention does not reveal them either, because each memory-side controller sustains more than $300\mathrm{GiB}/\mathrm{s}$ in our measurements, which our probes cannot saturate. Instead, we identify FBPs from how slice latency varies across SMs, since slices in the same FBP show similar variation. Using the atomic-contention microbenchmark described earlier, we isolate individual slices and measure access latency from each of the 42 SMs in one partition to each of its 48 local slices. After mean-centering per SM to remove systematic offsets, we obtain a $48 \times 42$ relative-latency matrix (Figure 6(a)). We then compute pairwise Pearson correlations between slices and derive the corresponding correlation-distance matrix, shown in Figure 6(b). The resulting structure reveals six distinct clusters, each containing eight slices, which is further confirmed by standard clustering analysis. We therefore conclude that each partition consists of 6 FBPs with 8 slices each, yielding a total of 12 FBPs and 96 slices across the GPU.

Cross-Partition FBP Correspondence. To characterize the cross-partition FBP topology, a fixed SM measures the mean latency to each FBP for remote L2 hits and for the same cache lines after coherence brings them into the home partition. The two sets of latencies have the same rank order across the six FBPs and are highly correlated (Pearson correlation $> 0.95$ ), differing by a nearly constant offset of $\sim 178$ cycles, close to the $\sim 170$ -cycle cross-partition penalty measured in Section V-A. Crossing partitions thus adds an almost uniform penalty without changing the relative SM-to-FBP distances, which implies a one-to-one correspondence between the six FBPs of the two partitions ( $P0$ and $P1$ ). Figure 5 shows this mirror topology as $\mathrm{FBP}_X(\mathrm{P}0) \leftrightarrow \mathrm{FBP}_X(\mathrm{P}1)$ , where the bidirectional arrows denote the cross-partition coherence relationship between matched FBPs.

![Fig. 6. (a) Mean-centered SM-to-slice relative-latency heatmap within one partition. (b) Slice correlation-distance heatmap derived from the latency vectors in (a), where distance is defined as $1 - r$ and $r$ is the Pearson correlation coefficient. SM labels are display indices reordered according to clustering results (not physical SM IDs).](images/fig06.jpg)

Fig. 6. (a) Mean-centered SM-to-slice relative-latency heatmap within one partition. (b) Slice correlation-distance heatmap derived from the latency vectors in (a), where distance is defined as $1 - r$ and $r$ is the Pearson correlation coefficient. SM labels are display indices reordered according to clustering results (not physical SM IDs).

## C. Complete Memory Interleave Hash Recovery

Using bit-flip analysis [14], we recover the complete routing hash that maps each memory address to its partition, FBP, and slice. Because NVIDIA L2 indexing uses physical addresses while user programs observe only virtual addresses, fully recovering the indexing hash is impossible. We therefore restrict our recovery to a $128\mathrm{MiB}$ virtual address window (the low 27 bits), which is sufficient for our subsequent eviction-set construction on the 60 MiB H20 L2 (Section V-D). Modern NVIDIA GPUs commonly allocate memory using 2 MiB large pages, whose offset bits (bits 0-20) are identical in virtual and physical addresses, so the recovered hash on these bits directly reflects the hardware routing. We therefore report only this portion of the routing hash. The recovered routing equations are:

$$
\begin{gathered}p = PA\lbrack12\rbrack \oplus PA\lbrack13\rbrack \oplus PA\lbrack15\rbrack \oplus PA\lbrack17\rbrack \oplus PA\lbrack19\rbrack \\ f_0 = PA\lbrack8\rbrack \oplus PA\lbrack12\rbrack \oplus PA\lbrack16\rbrack \oplus PA\lbrack18\rbrack \oplus PA\lbrack20\rbrack \\ f_1 = PA\lbrack10\rbrack \oplus PA\lbrack13\rbrack \oplus PA\lbrack16\rbrack \oplus PA\lbrack17\rbrack \oplus PA\lbrack19\rbrack \oplus PA\lbrack20\rbrack \\ f_2 = PA\lbrack11\rbrack \oplus PA\lbrack14\rbrack \oplus PA\lbrack18\rbrack \oplus PA\lbrack19\rbrack \\ s_0 = PA\lbrack8\rbrack \oplus PA\lbrack16\rbrack \oplus PA\lbrack17\rbrack \\ s_1 = PA\lbrack10\rbrack \oplus PA\lbrack16\rbrack \\ s_2 = PA\lbrack11\rbrack \oplus PA\lbrack16\rbrack \oplus PA\lbrack17\rbrack\end{gathered}\tag{1}
$$

These equations define a 7-bit logical routing space consisting of one partition bit $p$ , a 3-bit logical FBP index $f_{\log} = (f_{2}f_{1}f_0) \in \{ 0,\ldots ,7\}$ , and a 3-bit slice index $s = (s_{2}s_{1}s_0) \in \{ 0,\ldots ,7\}$ . The FBP and slice equations are shared by both partitions, while $p$ selects the target partition. Together, they yield $2 \times 8 \times 8 = 128$ logical destinations. However, $\mathrm{H}20$ has only $2 \times 6 \times 8 = 96$ active physical slices (Section V-B), so the 128 logical destinations must be remapped onto them. We recover this remap by locating the physical slice of every line in a $2\mathrm{MiB}$ page with atomic contention (Section V-B) and comparing it with Eq. (1). The remap preserves the partition bit $p$ . In each partition, six logical FBPs map one-to-one onto the six physical FBPs, and the other two are folded onto two of them with slice index $s \oplus 4$ . For the first $128\mathrm{KiB}$ of a page, this gives

$$
(f_{\text{ phys }}, s_{\text{ phys }}) = \left\{ \begin{array}{ll} (f_{\text{ log }}, s) , & f_{\text{ log }} < 6, \\  (f_{\text{ log }} \oplus 6, s \oplus 4) , & f_{\text{ log }} \geq 6. \end{array}\right.\tag{2}
$$

In Eq. (2), logical FBPs 6 and 7 are folded onto physical FBPs 0 and 1, which therefore receive twice as many lines as the other physical FBPs. In each subsequent $128\mathrm{KiB}$ , the folded logical FBPs land on the next pair of physical FBPs, cycling through the three pairs $0 - 1,2 - 3$ , and 4-5. This modulo-3 rotation balances the load: each physical pair receives the folded logical FBPs in exactly one of every three consecutive $128\mathrm{KiB}$ , so all physical FBPs receive the same number of lines over 384 KiB. Eq. (1) and the remap reproduce exactly the 12,288 colliding pairs measured in Section V-B $(S = 85.333)$ .

## D. Slice-Constrained Eviction-Set Construction

Prior L2 eviction-set (ES) construction techniques [53], [57] fail on modern, distributed L2 architectures such as H20. The large capacity of these caches (e.g., 60 MiB) forces traditional group elimination algorithms to filter very large candidate pools, accumulating noise that discards true ES members. Furthermore, undocumented cross-partition coherence traffic introduces interference into the measurements.

To overcome these challenges, we introduce a slice-constrained group-elimination method. Because only addresses mapping to the same set in the same slice contend, we use our recovered slice hash (Section V-C) to restrict the candidate pool to the target slice. We also perform profiling from an SM in the same partition, avoiding cross-partition coherence interference. Within this reduced pool, group elimination iteratively discards candidate groups that are unnecessary to evict the target, leaving an ES for the target set. This slice constraint aggressively prunes the search space by $93 \times$ : in our $120\mathrm{MiB}$ candidate pool, only 10,542 of 983,040 lines fall in the target slice. Successful construction of ESs also provides validation of the recovered slice hash, since stable ESs require a correct slice mapping.

From the resulting ESs, we infer that each H20 L2 cache set is 16-way associative and follows a standard LRU replacement policy. Specifically, exactly 16 conflicting accesses evict a line, even if it was accessed ten times before, ruling out SLRU. Moreover, when the 16 lines of a set are re-accessed in random orders (900 trials), a new line always evicts the least recently used one, ruling out pseudo-LRU.

## E. Cross-Partition Coherence at Cache-Set Granularity

Using the slice-constrained ESs constructed in Section V-D, we first access all lines in an ES from an SM in the home partition, placing them in that partition and verifying that they contend for a single cache set. We then access the same lines from an SM connected to the remote partition to create coherent copies. These copies continue to contend for a single cache set, showing that coherence preserves this cache set indexing rather than distributing the lines across multiple sets. Moreover, all 16 ES lines can be coherently replicated into the remote partition and simultaneously occupy this set. Thus, each cache set can hold any mixture of lines mapped to that partition and coherent copies from the peer partition, with no additional per-set capacity limit beyond the standard 16-way associativity.

Our measurements further indicate that corresponding cache sets in the two partitions maintain independent LRU states. Evicting a line from one set does not change the LRU order or trigger an eviction in the corresponding set of the other partition. During a cross-partition coherence event, the event behaves as a normal cache hit in the home partition and promotes the target line to the most recently used (MRU) position. In the remote partition, the same event behaves as the insertion of a new cache line: the coherent copy is inserted at the MRU position, and if the set is full, its current LRU line is evicted.

## VI. CONSTANT MEMORY HIERARCHY

In modern GPU workloads, constant memory is widely used to store network weights, scaling factors, and kernel parameters. Official documentation [41] names several constant-cache components (GCC, IDC, DCC, and IMC) but does not describe their organization. In this section, we use SASS microbenchmarks to reconstruct this hierarchy. We find three levels: L0 and L1 constant caches in each SM, both with 2 KiB, 4-way sets and Tree-PLRU replacement, and a 128 KiB GCC that is shared by the SMs of each GPC and also holds instructions.

### A.The Intra-SM Hierarchy: L0 and L1 Constant Caches

At the SM level, official documentation distinguishes three constant-data access paths, each associated with a different class of SASS instruction [41]. Immediate operands encoded in instructions (e.g., MOV) use the Immediate Constant Cache (IMC) path, ULDC uses the Data Constant Cache (DCC) path, and standard LDC uses the Indexed Constant Cache (IDC) path. We measure their cache hit latencies as 2, 15, and 25 cycles, respectively.

DCC and IMC Share a Single Physical Cache. Cross-instruction probing shows that MOV and ULDC share a single cache: warming a constant address with MOV makes a subsequent ULDC access hit, and capacity evictions through MOV also evict data brought in by ULDC, whereas cross-probing between LDC and either MOV or ULDC misses in both directions. Thus, the IMC and DCC are physically unified, while the IDC is separate. We attribute the latency difference between MOV and ULDC to their distinct data paths: ULDC routes data through the uniform-register path, whereas MOV uses direct operand fetch.

Cache Placement and Structural Parameters. We identify the physical placement of the IDC through SASS-level cross-warp probing: when one warp warms a cache line using LDC, other warps on different sub-partitions within the same SM still observe a hit, indicating that the IDC is shared at the SM level. We therefore classify it as the L1 Constant Cache. In contrast, the physical placement of the IMC/DCC cache cannot be directly determined due to limitations in the current SASS assembler tool. The documentation is also inconsistent: it places the IMC at the SM-subpartition level and the DCC at the SM level, even though our measurements show that the two are physically unified. Given the 2-cycle hit latency of the IMC, we conjecture that this shared cache resides at the SM-subpartition level and treat it as the L0 Constant Cache.

Using SASS-level microbenchmarks, we characterize the structural parameters of the two intra-SM constant caches. The L0 and L1 constant caches exhibit identical structural parameters: each has a $2\mathrm{KiB}$ capacity, a $64\mathrm{\;B}$ line size,4-way associativity, and 8 sets. In both caches, set indexing follows the same linear function, set_index = (offset/64) (mod 8).

Listing 3. Microbenchmark for reverse-engineering the replacement policy.

<table><tr><td></td><td>fill of the 4-way set</td></tr><tr><td>[B0-----:R-:W0:-:S05]</td><td>LDC R8, c[0x3] [ADDR_A] ;</td></tr><tr><td>[B0-----:R-:W0:-:S05]</td><td>LDC R8, c[0x3] [ADDR_B] ;</td></tr><tr><td>[B0-----:R-:W0:-:S05]</td><td>LDC R8, c[0x3] [ADDR_C] ;</td></tr><tr><td>[B0-----:R-:W0:-:S05]</td><td>LDC R8, c[0x3] [ADDR_D] ;</td></tr><tr><td>6 // Phase 2: Selectively</td><td></td></tr><tr><td>7 [B0-----:R-:W0:-:S05]</td><td>LDC R8, c[0x3] [ADDR_X] ;</td></tr><tr><td>[B0-----:R-:W0:-:S05]</td><td></td></tr></table>

Tree-PLRU Replacement Policy. To determine the cache set replacement policy, we use a SASS microbenchmark (see Listing [3]). The benchmark first fills all four ways of a single set with unique addresses $(A, B, C\text{ , and }D)$ . It then accesses these lines in different orders (the touch pattern) before inserting a fifth address $(E)$ to force an eviction. By measuring which address survives, we infer the underlying state machine. If the hardware employed a standard LRU policy, touching $A$ in Phase 2 would leave $B$ as the oldest resident line, causing $B$ to be evicted when $E$ is allocated. However, our timing results show that touching $A$ consistently causes $C$ to be evicted. Through testing various access sequences, we determined that both the L0 and L1 constant caches implement an identical Tree-based Pseudo-LRU (Tree-PLRU) policy.

The observed behavior is consistent with a 3-bit binary-tree state machine for a 4-way set. One root bit selects between two-way groups, and the corresponding child bit selects a victim within the chosen group. On each access, the corresponding bits flip to point away from the accessed path, thereby determining the pseudo-LRU eviction target for the next allocation.

### B.The GPC-Level Unified Cache (GCC)

When a constant load misses the SM-level caches, we observe a 57-cycle latency. This indicates a hit in the GPC Constant Cache (GCC), a structure that unifies constant data and instructions [41]. Cross-SM microbenchmarks verify that an address fetched by one SM results in a hit in the GCC for any other SM residing within the same Graphics Processing Cluster (GPC). This confirms that the GCC is physically located within the GPC and is shared across all its constituent SMs. We therefore classify it as the L1.5 Cache.

Extracting GCC Characteristics via Instruction Insertion. We first probe address offsets after an L1 eviction, revealing a 256 B GCC line size. Set-conflict measurements then establish the indexing function as set_index $= (\text{ address }/256)$ (mod 64), yielding 64 sets.

However, measuring the associativity and total capacity is challenging. Because the CUDA programming model limits user-accessible constant memory to $64\mathrm{KiB}$ , it is impossible to overflow the GCC using only constant data, so MT4G cannot measure its capacity on H100 [51]. To solve this limitation, we follow prior work [21], [22], [54] and leverage the unified nature of the GCC by using instructions to force data evictions. Specifically, a SASS microbenchmark first loads $c$ constant lines that map to one GCC set and then executes $k$ instruction lines that map to the same set, using a chain of branches with a $16\mathrm{KiB}$ stride. A timed probe then checks whether each constant line is still in the GCC.

With up to eight lines in the set, no constant line is evicted, even when all eight are constant lines. With nine lines, for example eight constant lines and one instruction line, one constant line is evicted in every run. Because this bound does not depend on which line the replacement policy evicts, the GCC is 8-way set-associative, with a capacity of $64 \times 8 \times 256\mathrm{\;B} = 128\mathrm{KiB}$ , much larger than the SM-level constant caches.

Non-Standard Replacement Policy. In mixed instruction and data microbenchmarks, the GCC replacement behavior deviates from standard LRU, FIFO, or Tree-PLRU policies: after eight constant lines fill a set, the next insertion does not evict the first filled line, and re-accessing any line other than this victim does not change the victim. We leave a detailed characterization of the GCC replacement policy to future work.

## C. Reverse Engineering Other GPUs

Sections IVI-VI recover the memory hierarchy of H20. To show that GemAtlas is hardware-agnostic, we apply the same microbenchmarks to H100 (Hopper), A800 (Ampere), and RTX 4090 (Ada). As Table III shows, all four GPUs share the organization we recover on H20 at every cache level and differ only in parameters, such as the L1 associativity and the number of L2 slices. The only structural difference is that RTX 4090 has a single L2 partition. Its hash still folds eight logical FBPs onto six physical FBPs, but the folding target rotates every $32\mathrm{KiB}$ instead of every $128\mathrm{KiB}$ .

## VII. MODELING AND EVALUATION

To evaluate the architectural impact of our reverse-engineered memory hierarchy, we integrate the recovered mechanisms into AccelSim and validate the simulation results against real hardware. Specifically, we use AccelSim v1.3.0 [26] as extended by the open-source release of Huerta et al. [18].

## A. Supporting Real-World LLM Benchmarks

Prior cycle-level GPU simulators have been evaluated primarily on traditional benchmark suites and CUTLASS GEMM workloads [5], [18], [26]. However, modern LLM workloads extend far beyond GEMM, and performance simulation of their diverse operators has received little attention. To broaden this coverage, we construct GemAtlas-Bench with 25 traditional GPU workloads widely used in prior work [5], [18], [26] and 28 LLM operators. These operators span attention [52], normalization and activation [4], [17], [49], [55], KV-cache management, token processing, quantization, and MoE. We take their implementations from vLLM [27], PyTorch [43], and FlashAttention-3 [48], with tensor dimensions configured based on the Qwen3-30B-A3B model [46]. The complete benchmark suite is listed in Table VI.

TABLE III

REVERSE-ENGINEERED MEMORY-HIERARCHY PARAMETERS ON FOUR GPUs.

<table><tr><td></td><td>Parameter</td><td>H20</td><td>H100</td><td>A800</td><td>RTX 4090</td></tr><tr><td rowspan="4">L1D</td><td>Slices×sets×ways</td><td>$4 \times 128 \times 4$</td><td>$4 \times 128 \times 4$</td><td>$4 \times 64 \times 6$</td><td>$4 \times 64 \times 4$</td></tr><tr><td>L1D+SMEM (KiB)</td><td>249/256</td><td>249/256</td><td>185/192</td><td>121/128</td></tr><tr><td>Hit latency (cyc.)</td><td>31</td><td>31</td><td>31</td><td>32</td></tr><tr><td>Index, replacement</td><td colspan="4">XOR, SLRU</td></tr><tr><td rowspan="6">L2</td><td>Part. $\times$ FBPs $\times$ slices</td><td>$2 \times 6 \times 8$</td><td>$2 \times 5 \times 8$</td><td>$2 \times 5 \times 4$</td><td>$1 \times 6 \times 6$</td></tr><tr><td>GPCs per partition</td><td>4,3</td><td>4,4</td><td>4,3</td><td>-</td></tr><tr><td>FBP remap</td><td>8→6</td><td>8→5</td><td>8→5</td><td>8→6</td></tr><tr><td>Slice remap</td><td>128→96</td><td>128→80</td><td>64→40</td><td>64→36</td></tr><tr><td>Ways, replacement</td><td colspan="4">16, LRU</td></tr><tr><td>Coherence</td><td colspan="4">Set-level -</td></tr><tr><td rowspan="2">Const.</td><td>L0C / L1C</td><td colspan="4">2 KiB, 8×4, Tree-PLRU</td></tr><tr><td>GCC sets $\times$ ways</td><td colspan="3">64×8</td><td></td></tr></table>

## B. Evaluation Methodology

Hardware measurement. We measure the execution cycles of each workload with Nsight Compute. Nsight Compute locks the GPU clocks and flushes all caches before each kernel, and the simulator likewise starts each kernel with cold caches. We repeat each measurement five times and use the average.

Baselines. The simulator released with [18] does not cover the GPUs we evaluate. We therefore define a baseline (BL) with their RTX A6000 model, the closest of these models to our GPUs, and set its SM count, clock frequency, and shared-memory size to those of each target GPU (e.g., 78 SMs for H20). All other parameters keep their RTX A6000 values, such as the 6 MiB L2 cache (Table IV). We then improve it (denoted as BL-Impv) by further aligning its memory parameters with those documented by NVIDIA for each GPU, such as the $60\mathrm{MiB}$ L2 cache and the $4\mathrm{\;TB}/\mathrm{s}$ memory bandwidth of H20. BL-Impv includes none of our reverse-engineered mechanisms, so any accuracy gain over BL-Impv comes from our findings.

Configurations. To isolate the contribution of each recovered cache level, we incrementally extend BL-Impv with three configurations. The L1D configuration replaces the L1 data cache of BL-Impv with our recovered model, L1D+L2D further replaces the L2, and GemAtlas further adds the recovered constant-cache hierarchy, so it includes all the mechanisms we recover. Table IV lists the parameters of each configuration.

TABLE IV

MEMORY-HIERARCHY PARAMETERS OF THE SIMULATED CONFIGURATIONS FOR H20.

<table><tr><td></td><td>Parameter</td><td>BL</td><td>BL-Impv</td><td>GemAtlas</td></tr><tr><td rowspan="3">L1D</td><td>Slices×sets×ways</td><td>1×4×256</td><td>1×4×512</td><td>$4 \times 128 \times 4$</td></tr><tr><td>Capacity</td><td>128 KiB</td><td>256 KiB</td><td>249 KiB-SMEM</td></tr><tr><td>Index, replacement</td><td>Linear, LRU</td><td>Linear, LRU</td><td>XOR, SLRU</td></tr><tr><td rowspan="5">L2</td><td>Slices</td><td>48</td><td>48</td><td>96</td></tr><tr><td>Partitions</td><td>1</td><td>1</td><td>2, GPC affinity</td></tr><tr><td>Capacity</td><td>6MiB</td><td>60 MiB</td><td>60MiB</td></tr><tr><td>Slice hash</td><td>IPOLY</td><td>IPOLY</td><td>Sec. V-C</td></tr><tr><td>DRAM bandwidth</td><td>2 TB/s</td><td>4 TB/s</td><td>4 TB/s</td></tr><tr><td rowspan="3">Const.</td><td>L0C / L1C size</td><td>0.5 / 2 KiB</td><td>0.5 / 2 KiB</td><td>2 / 2 KiB</td></tr><tr><td>L0C / L1C policy</td><td>LRU</td><td>LRU</td><td>Tree-PLRU</td></tr><tr><td>GCC</td><td>-</td><td>-</td><td>$64 \times 8$ per GPC</td></tr></table>

## C. Performance Accuracy

We simulate all 53 workloads under BL, BL-Impv, and the three configurations L1D, L1D+L2D, and GemAtlas. Table VI reports hardware cycles and per-workload cycle errors. Overall MAPE is 36.83% with BL. Applying the documented memory parameters reduces it to 27.58% with BL-Impv, and our recovered models further reduce it to 27.36% with L1D, 20.62% with L1D+L2D, and 20.04% with GemAtlas. Each incremental model improves aggregate accuracy, with distributed L2 modeling providing the largest reduction.

Compared with BL-Impv, GemAtlas reduces MAPE for both traditional benchmarks, from 28.69% to 19.31%, and LLM operators, from 26.59% to 20.69% (Figure 7). To relate accuracy gains to memory sensitivity, we adapt Alsop et al.'s workload classification [3]: a workload is memory-sensitive if its simulated cycles change by at least 5% between adjacent configurations from BL-Impv to GemAtlas. This classification identifies 26 sensitive workloads, marked with an asterisk in Table VI, and 27 insensitive workloads. As Figure 7 shows, compared with BL-Impv, GemAtlas reduces MAPE from 33.46% to 19.12% for sensitive workloads, but only from 21.91% to 20.91% for insensitive workloads. This contrast indicates that the simplified memory hierarchy in BL-Impv is a major source of error for sensitive workloads, while the recovered model corrects much of this error. The 20.91% MAPE that remains for insensitive workloads, whose cycles change little across configurations, therefore comes mainly from parts of the simulator other than the memory hierarchy.

Finally, GemAtlas raises the Pearson correlation with hardware from 0.896 with BL and 0.921 with BL-Impv to 0.974 (Figure 8), and increases the number of workloads within the $\pm 30\%$ error bound from 34 and 37 to 42, demonstrating stronger global agreement with hardware.

Other GPUs. We repeat the evaluation on H100, A800, and RTX 4090, building each configuration from the documented memory parameters of the GPU and our measurements in Sections IV-VI. As Table V shows, GemAtlas is more accurate than BL-Impv on every GPU, reducing MAPE from 37.7% to 21.3% on H100, from 32.5% to 28.0% on A800, and from 29.0% to 19.7% on RTX 4090. On average, GemAtlas reduces MAPE from 34.04% to 23.98% for the LLM operators and from 36.33% to 20.61% for the memory-sensitive workloads.

TABLE V

MAPE (%) ON FOUR GPUS.

<table><tr><td>GPU</td><td>BL-Impv</td><td>L1D</td><td>L1D+L2D</td><td>GemAtlas</td></tr><tr><td>H20</td><td>27.6</td><td>27.4</td><td>20.6</td><td>20.0</td></tr><tr><td>H100</td><td>37.7</td><td>37.4</td><td>21.9</td><td>21.3</td></tr><tr><td>A800</td><td>32.5</td><td>32.3</td><td>29.3</td><td>28.0</td></tr><tr><td>RTX 4090</td><td>29.0</td><td>28.5</td><td>20.2</td><td>19.7</td></tr><tr><td>Average</td><td>31.7</td><td>31.4</td><td>23.0</td><td>22.2</td></tr></table>

![Fig. 7. Mean absolute percentage error (MAPE) on H20 workload groups.](images/fig07.jpg)

Fig. 7. Mean absolute percentage error (MAPE) on H20 workload groups.

## D. Impact of L1D Modeling

Replacing the L1D of BL-Impv with the recovered H20 L1D changes overall MAPE only slightly, from 27.58% to 27.36%. Hardware L1 hit rates measured with Nsight Compute explain why: 22 of the 53 workloads hit in the L1 for fewer than 10% of their global loads, and for 21 others both BL-Impv and L1D already match the hardware hit rate within 2 percentage points. In 20 of these 21 workloads, more than 99% of the re-accessed sectors in the traces are re-accessed after fewer than $64\mathrm{KiB}$ of other data, a quarter of the L1 capacity, so both L1 organizations retain them. Only polybench-gesummv depends on the L1 organization. Its warps access memory with a $16\mathrm{KiB}$ stride and re-access data after 64 to $128\mathrm{KiB}$ . The linear index of BL-Impv maps these accesses to the same set and causes set conflicts, whereas the recovered XOR index reduces these conflicts, so its hit rate rises from 86.10% to 90.22%. Its cycle error also drops from $- 19.25\%$ to $- 15.47\%$ .

## E. Impact of Distributed L2 Modeling

Distributed L2D modeling provides the largest incremental accuracy improvement, reducing overall MAPE from 27.36% with L1D to 20.62% with L1D+L2D. Starting from the L1D configuration, we add three components of the recovered L2 model step by step. (1) The slice count replaces the 48 slices of BL-Impv with the 96 slices recovered in Section V-B. (2) The FBP/slice routing hash replaces the IPOLY hash and selects the partition, FBP, and slice of each address. (3) The partition model adds the two partitions, the GPC-to-partition mapping, the measured local and cross-partition latencies, and cross-partition coherence, which together form the partitioned L2 of H20 (Sections V-A and V-E). We do not split this component further because a model with only some of these mechanisms does not correspond to real hardware. The slice count reduces MAPE to 22.14%, the largest reduction among the three components. With only 48 slices, the simulated L2 serves fewer requests in parallel, so L1D overestimates execution cycles by 14.30% on average, compared with 6.87% with 96 slices. Adding routing hash then lowers MAPE slightly to 21.83%, and the partition model finally reduces the MAPE to 20.62% by accounting for cross-partition accesses. In parboil-stencil, for example, 46% of the L2 reads come from SMs in the other partition, and modeling their latency and replication reduces the error from -29.3% to -10.9%.

![Fig. 8. Pearson correlation between simulated and hardware execution cycles on H20 under BL, BL-Impv, and GemAtlas.](images/fig08.jpg)

Fig. 8. Pearson correlation between simulated and hardware execution cycles on H20 under BL, BL-Impv, and GemAtlas.

TABLE VI

PER-WORKLOAD H20 HARDWARE CYCLES AND SIMULATION ERRORS (%). ASTERISKS MARK MEMORY-SENSITIVE WORKLOADS.

<table><tr><td>Workload</td><td>HW cyc.</td><td>BL</td><td>BL-Impv</td><td>L1D</td><td>L1D+L2D</td><td>GemAtlas</td></tr><tr><td colspan="7">Traditional Benchmarks</td></tr><tr><td>rodinia-pathfinder</td><td>36726</td><td>-14.98</td><td>-15.21</td><td>-15.23</td><td>-14.50</td><td>-13.67</td></tr><tr><td>rodinia-hotspot</td><td>72095</td><td>+25.63</td><td>+26.81</td><td>+27.06</td><td>+23.28</td><td>+22.05</td></tr><tr><td>rodinia-bfs</td><td>137208</td><td>-15.07</td><td>-13.99</td><td>-14.03</td><td>-14.30</td><td>-14.40</td></tr><tr><td>rodinia-nn</td><td>27681</td><td>-11.11</td><td>-0.92</td><td>-1.12</td><td>-7.54</td><td>-7.81</td></tr><tr><td>rodinia-srad-v2</td><td>34740</td><td>+2.07</td><td>+7.38</td><td>+7.96</td><td>+4.72</td><td>+4.28</td></tr><tr><td>rodinia-streamcluster</td><td>1162650</td><td>-10.12</td><td>-5.43</td><td>-5.77</td><td>-14.63</td><td>-14.85</td></tr><tr><td>rodinia-backprop</td><td>15598</td><td>+23.44</td><td>+27.54</td><td>+26.84</td><td>+34.41</td><td>+33.85</td></tr><tr><td>rodinia-lud</td><td>209332</td><td>-17.48</td><td>-16.28</td><td>-16.31</td><td>-19.19</td><td>-19.01</td></tr><tr><td>rodinia-heartwall*</td><td>11871</td><td>-7.35</td><td>-7.90</td><td>-7.49</td><td>-4.51</td><td>-9.82</td></tr><tr><td>rodinia-nw</td><td>150396</td><td>+28.41</td><td>+28.49</td><td>+28.49</td><td>+28.44</td><td>+28.53</td></tr><tr><td>parboil-bfs</td><td>8056</td><td>-37.70</td><td>-37.60</td><td>-37.60</td><td>-39.09</td><td>-38.93</td></tr><tr><td>parboil-mri-q</td><td>349190</td><td>+17.44</td><td>+31.99</td><td>+32.11</td><td>+18.70</td><td>+2.24</td></tr><tr><td>parboil-histo</td><td>3403026</td><td>-35.05</td><td>-33.21</td><td>-33.08</td><td>-38.55</td><td>-38.50</td></tr><tr><td>parboil-stencil</td><td>6551610</td><td>-22.47</td><td>-14.30</td><td>-14.30</td><td>-10.87</td><td>-8.01</td></tr><tr><td>parboil-spmv</td><td>1619775</td><td>+96.04</td><td>+37.98</td><td>+37.54</td><td>-0.99</td><td>-1.24</td></tr><tr><td>parboil-lbm</td><td>2322409</td><td>+162.46</td><td>+145.20</td><td>+145.81</td><td>+7.08</td><td>+7.05</td></tr><tr><td>polybench-atax</td><td>4114959</td><td>-23.51</td><td>-23.37</td><td>-22.88</td><td>-15.52</td><td>-15.76</td></tr><tr><td>polybench-bicg</td><td>4116584</td><td>-23.40</td><td>-23.67</td><td>-22.58</td><td>-15.59</td><td>-16.00</td></tr><tr><td>polybench-mvt</td><td>4093132</td><td>-22.68</td><td>-22.94</td><td>-22.06</td><td>-15.13</td><td>-14.71</td></tr><tr><td>polybench-2dconv</td><td>182170</td><td>+81.69</td><td>+77.54</td><td>+78.07</td><td>+67.57</td><td>+67.93</td></tr><tr><td>polybench-3dconv</td><td>1465403</td><td>-10.98</td><td>-7.57</td><td>-7.63</td><td>-14.87</td><td>-14.79</td></tr><tr><td>polybench-gemm</td><td>515826</td><td>+68.74</td><td>+38.89</td><td>+38.77</td><td>+25.69</td><td>+16.09</td></tr><tr><td>polybench-gesummv</td><td>2708664</td><td>+16.58</td><td>-19.25</td><td>-15.47</td><td>-14.99</td><td>-15.05</td></tr><tr><td>lonestar-bfs-wla-rmat20</td><td>4930879</td><td>+26.55</td><td>+19.77</td><td>+21.45</td><td>+23.62</td><td>+30.10</td></tr><tr><td>pannotia-fw_block-512</td><td>1356172</td><td>+33.77</td><td>+33.92</td><td>+34.21</td><td>+28.11</td><td>+27.98</td></tr><tr><td>Traditional MAPE</td><td>-</td><td>33.39</td><td>28.69</td><td>28.55</td><td>20.08</td><td>19.31</td></tr><tr><td colspan="7">LLM Operators</td></tr><tr><td>vllm-rmsnorm</td><td>12556</td><td>+128.27</td><td>+101.91</td><td>+103.21</td><td>+97.69</td><td>+93.95</td></tr><tr><td>vllm-fused-add-rmsnorm</td><td>16583</td><td>+106.81</td><td>+45.90</td><td>+44.82</td><td>+26.26</td><td>+24.91</td></tr><tr><td>vllm-layernorm</td><td>118510</td><td>+46.37</td><td>+21.23</td><td>+21.57</td><td>+9.79</td><td>+9.37</td></tr><tr><td>torch-layernorm</td><td>19537</td><td>+9.50</td><td>+8.20</td><td>+9.03</td><td>+8.65</td><td>+8.44</td></tr><tr><td>vllm-silu-mul</td><td>81949</td><td>+18.20</td><td>+14.93</td><td>+15.60</td><td>+16.84</td><td>+17.23</td></tr><tr><td>torch-swiglu</td><td>111998</td><td>+18.77</td><td>+6.48</td><td>+6.39</td><td>+6.12</td><td>+6.44</td></tr><tr><td>torch-ffn-gate</td><td>798534</td><td>-61.46</td><td>-63.08</td><td>-63.07</td><td>-62.97</td><td>-62.91</td></tr><tr><td>torch-embedding</td><td>12017</td><td>+61.48</td><td>+23.67</td><td>+22.90</td><td>+0.88</td><td>-2.86</td></tr><tr><td>torch-softmax-vocab</td><td>93853</td><td>+20.85</td><td>+2.39</td><td>+5.68</td><td>+19.43</td><td>+19.24</td></tr><tr><td>torch-topk-vocab</td><td>201720</td><td>+16.32</td><td>+11.28</td><td>+11.06</td><td>+1.36</td><td>+2.12</td></tr><tr><td>torch-gelu</td><td>30833</td><td>+48.12</td><td>+10.88</td><td>+10.69</td><td>+1.72</td><td>+1.68</td></tr><tr><td>torch-rope</td><td>161187</td><td>-6.06</td><td>-0.70</td><td>-0.51</td><td>-6.78</td><td>-6.92</td></tr><tr><td>vllm-paged-attn-v2</td><td>130354</td><td>+83.63</td><td>+81.43</td><td>+73.22</td><td>+60.40</td><td>+59.78</td></tr><tr><td>flash3-mqa-kvcache</td><td>55781</td><td>+45.35</td><td>+31.71</td><td>+31.71</td><td>+33.28</td><td>+31.90</td></tr><tr><td>flash3-packgqa-kvcache</td><td>55699</td><td>+46.04</td><td>+32.63</td><td>+32.73</td><td>+32.54</td><td>+32.49</td></tr><tr><td>vllm-reshape-cache</td><td>9224</td><td>+23.69</td><td>+17.61</td><td>+16.90</td><td>+15.80</td><td>+14.60</td></tr><tr><td>vllm-reshape-cache-flash</td><td>7915</td><td>+5.19</td><td>+3.10</td><td>+3.01</td><td>+1.97</td><td>+2.21</td></tr><tr><td>vllm-token-quant-fp8</td><td>15169</td><td>+28.12</td><td>+29.72</td><td>+22.01</td><td>+28.47</td><td>+21.76</td></tr><tr><td>vllm-token-group-quant</td><td>15685</td><td>+33.04</td><td>+15.04</td><td>+15.07</td><td>+15.39</td><td>+12.98</td></tr><tr><td>vllm-silu-mul-fp8</td><td>145461</td><td>+22.28</td><td>+11.73</td><td>+11.16</td><td>+11.90</td><td>+12.68</td></tr><tr><td>vllm-2d-silu-mul-fp8</td><td>144148</td><td>+23.35</td><td>+12.87</td><td>+13.41</td><td>+13.37</td><td>+13.30</td></tr><tr><td>vllm-fused-topk</td><td>13052</td><td>+4.68</td><td>+4.60</td><td>+5.14</td><td>+4.51</td><td>+6.31</td></tr><tr><td>torch-residual-add</td><td>11038</td><td>+120.19</td><td>+72.01</td><td>+73.87</td><td>+17.64</td><td>+15.24</td></tr><tr><td>vllm-moe-align-block</td><td>14670</td><td>+2.95</td><td>+0.95</td><td>-0.32</td><td>-1.21</td><td>-0.26</td></tr><tr><td>vllm-moe-permute</td><td>42660</td><td>+85.31</td><td>+66.95</td><td>+67.01</td><td>+40.91</td><td>+42.60</td></tr><tr><td>vllm-moe-unpermute</td><td>13834</td><td>+19.38</td><td>+6.68</td><td>+9.66</td><td>+6.90</td><td>+9.38</td></tr><tr><td>vllm-grouped-gemm</td><td>655735</td><td>-15.83</td><td>-23.13</td><td>-23.13</td><td>-24.04</td><td>-23.81</td></tr><tr><td>vllm-cutlass-moe-fp8</td><td>658249</td><td>-16.21</td><td>-23.60</td><td>-23.49</td><td>-23.98</td><td>-23.90</td></tr><tr><td>LLM MAPE</td><td>-</td><td>39.91</td><td>26.59</td><td>26.30</td><td>21.10</td><td>20.69</td></tr><tr><td>Overall MAPE</td><td>-</td><td>36.83</td><td>27.58</td><td>27.36</td><td>20.62</td><td>20.04</td></tr></table>

## F. Impact of Constant-Memory Modeling

Adding the recovered constant-memory hierarchy reduces overall MAPE from 20.62% to 20.04%. The gain is small because constant-memory accesses exceed 1% of L2 accesses in only 9 of the 53 workloads. Nevertheless, GemAtlas reduces the constant-memory requests reaching the L2 in 52 of the 53 workloads, by 71.15% in total. In parboil-mri-q, which has the highest constant-memory share (6.16%), the GCC reduces these requests by 95.35% and the cycle error from +18.70% to +2.24%.

## VIII. DISCUSSION

Beyond improving simulation accuracy, the mechanisms we recover provide architectural insights for both software optimization and hardware design.

L2-aware data placement. Eq. (1) gives the partition of every address, so software can place the data that each SM reads in the SM's local partition. This avoids both the extra latency of cross-partition accesses (Section V-A) and the L2 capacity taken by their coherent copies (Section V-E). We apply this placement to the vLLM paged-attention decode kernel on H20. It removes 99% of the cross-partition requests, lets more KV data fit in the L2, and speeds up the kernel by $1.10 - 1.12 \times$ over the same kernel without this placement. The hash also suggests placement at the FBP level: because of the rotating remap in Eq. (2), the FBP load is balanced only over three consecutive 128 KiB ranges, so a hot region of about $128\mathrm{KiB}$ could be spread over $384\mathrm{KiB}$ to use all FBPs evenly.

Decoupled L1 tags and data. The recovered L1 decouples its tag and data arrays, which appears to let it switch capacity between the L1 and SMEM. The tag array is fixed, so the L1 keeps the same XOR index under every SMEM configuration, while the data array shrinks as the SMEM allocation grows (Section IV-A). The data array can therefore fill up before the tags do, and the L1 needs cross-set eviction to make room, similar to the global data replacement of the V-Way cache [45]. Cross-set eviction starts when the L1 data and SMEM together hold $249\mathrm{KiB},7\mathrm{KiB}$ below the $256\mathrm{KiB}$ unified capacity (Table II). We conjecture that this $7\mathrm{KiB}$ is a buffer that holds incoming lines when cross-set eviction cannot keep up with heavy fill traffic. Because data entries are allocated independently of the tags, we suggest a possible design that may improve performance: one tag owns several $128\mathrm{\;B}$ data entries as a larger line, and all of these entries are released when the tag is evicted. Data with spatial locality could then use larger lines while other data keeps 128 B lines, without adding tags. We leave the evaluation of this design to future work.

## IX. RELATED WORK

Prior work has studied GPU caches and TLBs from GT200 to A100 [21]-[23], [25], [30], [36], [54], [56] and GPU NoCs [2], [24], and, more recently, Hopper-era features, automated topology discovery, and the L2/DRAM NUMA topology [33], [35], [51]. These studies provide component-level visibility but do not reconstruct a unified, cache-set-level hierarchy. We next compare our methods and architectural findings with these studies.

Our probes build on group elimination [53], bit-flip analysis [14], and GCC eviction with instruction fetches [21], [22], [54]. Our SASS probes measure latency more accurately than the CUDA/PTX benchmarks of prior work [24], [33], [35], [51]: our probes time only the load, whereas these benchmarks also time cold misses or dependent instructions. We also introduce five new methods: (1) SASS-level eviction sets for the L1 and constant caches, (2) L1 capacity measurement at cache-line granularity instead of CUDA pointer chasing [36], [51], (3) reading the SM-to-GPC and GPC-to-partition mappings from special registers instead of inferring them from latencies and thread-block clusters [23], [24], [33], (4) grouping L2 slices into FBPs by latency correlation, and (5) slice-constrained group elimination, to our knowledge the first method to reliably and efficiently construct eviction sets on partitioned GPU L2 caches, where prior techniques fail [53], [57].

Several of our architectural findings refine or extend prior measurements, and Table 1 marks them as improved (I). For the L1, we improve three prior results. First, Jia et al. find the L1 capacity of Volta and Turing 7 KiB below the nominal size and conjecture that a new replacement policy causes this gap [21], [22], whereas our line-by-line probe finds the same gap on H20 before any set holds more than four lines, which rules out replacement within a set (Section VIII). Second, prior work reports the L1 associativity only for Fermi [36], and to our knowledge we are the first to measure it on a modern GPU (4-way on H20). Third, the benchmarks of Luo et al. and MT4G yield L1 hit latencies of 41 and 38 cycles on H20 [35], [51], whereas our SASS probe measures a more accurate 31 cycles. For the L2, we improve two prior results. First, Jia et al. report a 16-way L2 on Volta and Turing [21], [22], and to our knowledge we are the first to measure the associativity of a partitioned L2 (16-way on H20). Second, Mei and Chu conclude from aperiodic miss patterns that the Fermi L2 does not use LRU, and note that its address mapping is irregular [36]. Our eviction sets access a single set, which separates the replacement policy from the address mapping, and show that H20 uses LRU. For the constant caches, we improve three prior results. First, prior work reports a $2\mathrm{KiB}$ , 4-way L1 constant cache with 64 B lines and 8 sets on GT200, Volta, and Turing [21], [22], [54], but only its capacity and line size on H100 [51], whereas we find the same geometry on H20 together with its linear set index. Second, prior work measures a single hit latency per constant-cache level [21]- [23], [51], [54], whereas we find that the latency also depends on the instruction: MOV and ULDC both hit in the L0 but take 2 and 15 cycles. Third, Jia et al. measure the GCC capacity on Volta, Turing, and A100 [21]-[23], and MT4G cannot measure it beyond $64\mathrm{KiB}$ on H100 because CUDA limits constant memory to $64\mathrm{KiB}$ [51]. Evicting GCC lines with instruction fetches [21], [22], [54], we fill a single set with both constant and instruction lines and find 64 sets of 8 ways (128 KiB).

The remaining findings, marked as new (N) in Table 1, have not been reported before to our knowledge. For the L1, they are the four slices, the XOR set index, the tag-data asymmetry that causes cross-set evictions, and the SLRU policy. These findings also explain two observations of Jia et al. on Volta and Turing [21], [22]: the four L1 sets they report appear to be the four slices, and the random evictions they observe appear to be cross-set evictions. For the L2, they are the partition/FBP/slice hash with its 128-to-96 remap and the set-level coherence, whereas prior work observes the address interleaving between the two partitions and characterizes coherence at cache-line granularity [23], [33]. For the constant caches, they are the L0 shared by MOV and ULDC with its geometry and index, the GCC index, and the Tree-PLRU policy of the L0 and L1, whereas prior work identifies the L0 [18] and finds only that the L1 does not use LRU [21], [22].

These memory-system gaps also affect GPU simulators such as GPGPU-Sim [5], Accel-Sim [25], [26], gem5 and its GPU models [6], [20], [47], and MGPUSim [50], and recent work revisits their hardware assumptions [18], [19]. However, existing simulators lack hardware-faithful slice routing, set indexing, replacement, and cross-partition coherence.

## X. CONCLUSION

In this paper, we present GemAtlas, a systematic reverse engineering study of the memory hierarchy of modern NVIDIA GPUs. We develop a SASS-level microbenchmarking suite to recover undocumented microarchitectural behavior. We characterize the L1 data cache in depth (including sliced organization, XOR-based routing, tag/data asymmetry, and segmented replacement), reconstruct the L2 partition/slice/FBP hierarchy, recover address hash and remap logic, expose cache-set granularity cross-partition coherence and replacement behavior, and introduce an efficient hash-guided slice-constrained, eviction-set construction method for modern GPUs. We further characterize the constant-memory hierarchy, covering its fine-grained geometry and replacement policy. Finally, incorporating these findings into a cycle-level GPU simulator reduces the average MAPE over four GPUs from 34.04% to 23.98% for real-world LLM operators, from 36.33% to 20.61% for memory-sensitive workloads, and from 31.69% to 22.24% overall.

## REFERENCES

[1] H. Abdelkhalik, Y. Arafa, N. Santhi, and A.-H. A. Badawy, "Demys-tifying the nvidia ampere architecture through microbenchmarking and instruction-level analysis," in 2022 IEEE High Performance Extreme Computing Conference (HPEC), 2022, pp. 1-8.

[2] J. Ahn, J. Kim, H. Kasan, L. Delshadtehrani, W. Song, A. Joshi, and J. Kim, "Network-on-chip microarchitecture-based covert channel in gpus," in MICRO-54: 54th Annual IEEE/ACM International Symposium on Microarchitecture, ser. MICRO '21. New York, NY, USA: Association for Computing Machinery, 2021, p. 565-577. [Online]. Available: https://doi.org/10.1145/3466752.3480093

[3] J. Alsop, X. Zhang, T. T. Yeh, B. M. Beckmann, M. D. Sinclair, S. Bharadwaj, A. Dutu, A. Gutierrez, O. Kayiran, M. LeBeane, B. Potter, and S. Puthoor, "Optimizing GPU cache policies for MI workloads," in 2019 IEEE International Symposium on Workload Characterization (IISWC), 2019, pp. 243-248.

[4] J. L. Ba, J. R. Kiros, and G. E. Hinton, "Layer normalization," arXiv preprint arXiv:1607.06450, 2016.

[5] A. Bakhoda, G. L. Yuan, W. W. L. Fung, H. Wong, and T. M. Aamodt, "Analyzing cuda workloads using a detailed gpu simulator," in 2009 IEEE International Symposium on Performance Analysis of Systems and Software, 2009, pp. 163-174.

[6] N. Binkert, B. Beckmann, G. Black, S. K. Reinhardt, A. Saidi, A. Basu, J. Hestness, D. R. Hower, T. Krishna, S. Sardashti, R. Sen, K. Sewell, M. Shoaib, N. Vaish, M. D. Hill, and D. A. Wood, "The gem5 simulator," SIGARCH Comput. Archit. News, vol. 39, no. 2, p. 1-7, Aug. 2011. [Online]. Available: https://doi.org/10.1145/2024716.2024718

[7] A. Bustamam, K. Burrage, and N. A. Hamilton, "Fast parallel markov clustering in bioinformatics using massively parallel computing on gpu with cuda and ellpack-r sparse format," IEEE/ACM Transactions on Computational Biology and Bioinformatics, vol. 9, no. 3, pp. 679-692, 2012.

[8] S. Chetlur, C. Woolley, P. Vandermersch, J. M. Cohen, J. Tran, B. Catanzaro, and E. Shelhamer, "cudnn: Efficient primitives for deep learning," ArXiv, vol. abs/1410.0759, 2014. [Online]. Available: https://api.semanticscholar.org/CorpusID:12330432

[9] R. Chowdhury, "I/o for llm inference: A survey of storage and memory bottlenecks," Research Square, 2026, preprint. [Online]. Available: https://doi.org/10.21203/rs.3.rs-9036613/v1

[10] Cloudcores, "Cuassembler: An unofficial cuda assembler, for all generations of sass," 2022, https://github.com/cloudcores/CuAssembler.[Online].Available: https://github.com/cloudcores/CuAssembler

[11] Y. Dong, Y. Miao, W. Li, X. Zheng, C. Wang, J. Wu, and F. Lyu, "Accelerating llm inference throughput via asynchronous kv cache prefetching," in Proceedings of the AAAI Conference on Artificial Intelligence, vol. 40, no. 25, 2026, pp. 20844-20851.

[12] Y. Fu, E. Bolotin, A. Jaleel, G. Dalal, S. Mannor, J. Subag, N. Korem, M. Behar, and D. Nellans, "Autoscratch: Ml-optimized cache management for inference-oriented gpus," Proceedings of Machine Learning and Systems, vol. 5, pp. 495-512, 2023.

[13] H. Gao and C. Wilkerson, "A dueling segmented lru replacement algorithm with adaptive bypassing," 2010. [Online]. Available: https://api.semanticscholar.org/CorpusID:18813408

[14] L. Gerlach, S. Schwarz, N. Faros, and M. Schwarz, " Efficient and Generic Microarchitectural Hash-Function Recovery ," in 2024 IEEE Symposium on Security and Privacy (SP). Los Alamitos, CA, USA: IEEE Computer Society, May 2024, pp. 3661-3678. [Online]. Available: https://doi.ieeecomputersociety.org/10.1109/SP54263.2024.00028

[15] A. Gholami, Z. Yao, S. Kim, C. Hooper, M. W. Mahoney, and K. Keutzer, "Ai and memory wall," IEEE Micro.

[16] A. Gutierrez, B. M. Beckmann, A. Dutu, J. Gross, M. LeBeane, J. Kalamatianos, O. Kayiran, M. Poremba, B. Potter, S. Puthoor, M. D. Sinclair, M. Wyse, J. Yin, X. Zhang, A. Jain, and T. Rogers, "Lost in abstraction: Pitfalls of analyzing gpus at the intermediate language level," in 2018 IEEE International Symposium on High Performance Computer Architecture (HPCA), 2018, pp. 608-619.

[17] D. Hendrycks and K. Gimpel, "Gaussian error linear units (gelus)," arXiv preprint arXiv:1606.08415, 2016.

[18] R. Huerta, M. A. Shoushtary, J.-L. Cruz, and A. Gonzalez, "Dissecting and modeling the architecture of modern gpu cores," in Proceedings of the 58th IEEE/ACM International Symposium on Microarchitecture, ser. MICRO '25. New York, NY, USA: Association for Computing Machinery, 2025, p. 369-384. [Online]. Available: https://doi.org/10.1145/3725843.3756041

[19] R. Huerta, M. A. Shoushtary, and A. Gonz'alez, "Analyzing and improving hardware modeling of accel-sim," ArXiv, vol. abs/2401.10082, 2024. [Online]. Available: https://api.semanticscholar.org/CorpusID:267035102

[20] C. Jamieson and A. Chandrashekar, "gem5 GPU Accuracy Profiler (GAP)," in 4th gem5 Users Workshop, 2022. [Online]. Available: https://par.nsf.gov/biblio/10359089

[21] Z. Jia, M. Maggioni, J. K. Smith, and D. P. Scarpazza, "Dissecting the nvidia turing t4 gpu via microbenchmarking," ArXiv, vol. abs/1903.07486, 2019. [Online]. Available: https://api.semanticscholar.org/CorpusID:81981887

[22] Z. Jia, M. Maggioni, B. Staiger, and D. P. Scarpazza, "Dissecting the nvidia volta gpu architecture via microbenchmarking," arXiv preprint arXiv:1804.06826, 2018.

[23] Z. Jia, P. V. Sandt, M. Maggioni, J. Smith, and D. Scarpazza, "Dissecting the ampere gpu architecture through microbenchmarking," NVIDIA GTC 2021, 2021.

[24] Z. Jin, C. Rocca, J. Kim, H. Kasan, M. Rhu, A. Bakhoda, T. M. Aamodt, and J. Kim, "Uncovering real gpu noc characteristics: Implications on interconnect architecture," in Proceedings of the 2024 57th IEEE/ACM International Symposium on Microarchitecture, ser. MICRO '24. IEEE Press, 2024, p. 885-898. [Online]. Available: https://doi.org/10.1109/MICRO61859.2024.00070

[25] M. Khairy, A. Jain, T. M. Aamodt, and T. G. Rogers, "Exploring modern gpu memory system design challenges through accurate modeling," ArXiv, vol. abs/1810.07269, 2018. [Online]. Available: https://api.semanticscholar.org/CorpusID:52987962

[26] M. Khairy, Z. Shen, T. M. Aamodt, and T. G. Rogers, "Accel-sim: an extensible simulation framework for validated gpu modeling," in Proceedings of the ACM/IEEE 47th Annual International Symposium on Computer Architecture, ser. ISCA '20. IEEE Press, 2020, p. 473-486. [Online]. Available: https://doi.org/10.1109/ISCA45697.2020.00047

[27] W. Kwon, Z. Li, S. Zhuang, Y. Sheng, L. Zheng, C. H. Yu, J. Gonzalez, H. Zhang, and I. Stoica, "Efficient memory management for large language model serving with pagedattention," in Proceedings of the 29th symposium on operating systems principles, 2023, pp. 611-626.

[28] A. Lashgar, E. Salehi, and A. Baniasadi, "A case study in reverse engineering gpgpus: Outstanding memory handling resources," SIGARCH Comput. Archit. News, vol. 43, no. 4, p. 15-21, Apr. 2016. [Online]. Available: https://doi.org/10.1145/2927964.2927968

[29] B. Li, Y. Jiang, V. Gadepally, and D. Tiwari, "Llm inference serving: Survey of recent advances and opportunities," in 2024 IEEE High Performance Extreme Computing Conference (HPEC). IEEE, 2024, pp. 1-8.

[30] C. Li, Y. Yang, H. Dai, S. Yan, F. Mueller, and H. Zhou, "Understanding the tradeoffs between software-managed vs. hardware-managed caches in gpus," in 2014 IEEE International Symposium on Performance Analysis of Systems and Software (ISPASS). IEEE, 2014.

[31] J. L. X. Li, Z. Mo, and L. Guo, "Revealing chemical reactions of coal pyrolysis with gpu-enabled reaxff molecular dynamics and chemin-formatics analysis," 2015, molecular Simulation 41, 1-3 (2015), 13-27.

[32] E. Lindholm, J. R. Nickolls, S. F. Oberman, and J. Montrym, "Nvidia tesla: A unified graphics and computing architecture," IEEE Micro, vol. 28, 2008. [Online]. Available: https://api.semanticscholar.org/CorpusID:2793450

[33] C. Liu, Y. Chen, and T. E. Carlson, "DGNA: Dissecting GPU NUMA architecture through microbenchmarking and data analysis," arXiv preprint arXiv:2607.19922, 2026.

[34] W. Liu, B. Schmidt, G. Voss, and W. Müller-Wittig, "Accelerating molecular dynamics simulations using graphics processing units with cuda," 2008, computer Physics Communications 179, 9 (2008), 634-641.

[35] W. Luo, R. Fan, Z. Li, D. Du, Q. Wang, and X. Chu, "Benchmarking and dissecting the nvidia hopper gpu architecture," in 2024 IEEE International Parallel and Distributed Processing Symposium (IPDPS). IEEE, 2024, pp. 656-667.

[36] X. Mei and X. Chu, "Dissecting GPU memory hierarchy through microbenchmarking," IEEE Transactions on Parallel and Distributed Systems, vol. 28, no. 1, pp. 72-86, 2017.

[37] S. Narang and G. Diamos, "Deepbench: Benchmarking deep learning op- erations on different hardware," 2016, https://github.com/baidu-research/DeepBench.[Online].Available: https://github.com/baidu-research/DeepBench

[38] M. S. Nobile, P. Cazzaniga, A. Tangherloni, and D. Besozzi, "Graphics processing units in bioinformatics, computational biology and systems biology," Briefings in bioinformatics, vol. 18, no. 5, pp. 870-885, 2017.

[39] NVIDIA, "Cuda binary utilities documentation," https://docs.nvidia.com/cuda/cuda-binary-utilities/. [Online]. Available: https://docs.nvidia.com/

[40] NVIDIA, "Nvidia h100 tensor core gpu architecture," 2022, technical Report. NVIDIA.

[41] NVIDIA, NVIDIA Nsight Compute Profiling Guide, 2026, version 2026.1.1. [Online]. Available: https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html

[42] OpenAI, "Gpt-4 technical report," arXiv preprint arXiv:2303.08774, 2023.

[43] A. Paszke, S. Gross, F. Massa, A. Lerer, J. Bradbury, G. Chanan, T. Killeen, Z. Lin, N. Gimelshein, L. Antiga, A. Desmaison, A. Kopf, E. Yang, Z. DeVito, M. Raison, A. Tejani, S. Chilamkurthy, B. Steiner, L. Fang, J. Bai, and S. Chintala, "Pytorch: An imperative style, high-performance deep learning library," Advances in neural information processing systems, vol. 32, 2019.

[44] F. E. H. Pérez, N. Mukhadiyev, X. Xu, A. Sow, B. J. Lee, R. Sankaran, and H. G. Im, "Direct numerical simulations of reacting flows with detailed chemistry using many-core/gpu acceleration," 2018, computers & Fluids 173 (2018), 73-79.

[45] M. K. Qureshi, D. Thompson, and Y. N. Patt, "The V-Way cache: Demand-based associativity via global replacement," in Proceedings of the 32nd International Symposium on Computer Architecture (ISCA), 2005, pp. 544-555.

[46] Qwen Team, "Qwen3-30B-A3B," ModelScope model repository, 2025. [Online]. Available: https://modelscope.cn/models/Qwen/Qwen3-30B-A3B

[47] V. Ramadas, D. Kouchekinia, N. Osuji, and M. D. Sinclair, "Closing the gap: Improving the accuracy of gem5's gpu models." [Online]. Available: https://par.nsf.gov/biblio/10468163

[48] J. Shah, G. Bikshandi, Y. Zhang, V. Thakkar, P. Ramani, and T. Dao, "Flashattention-3: Fast and accurate attention with asynchrony and low-precision," Advances in Neural Information Processing Systems, vol. 37, pp. 68 658-68 685, 2024.

[49] N. Shazeer, "Glu variants improve transformer," arXiv preprint arXiv:2002.05202, 2020.

[50] Y. Sun, T. Baruah, S. A. Mojumder, S. Dong, X. Gong, S. Treadway, Y. Bao, S. Hance, C. McCardwell, V. Zhao, H. Barclay, A. K. Ziabari, Z. Chen, R. Ubal, J. L. Abellán, J. Kim, A. Joshi, and D. Kaeli, "Mgpusim: enabling multi-gpu performance modeling and optimization," in Proceedings of the 46th International Symposium on Computer Architecture, ser. ISCA '19. New York, NY, USA: Association for Computing Machinery, 2019, p. 197-209. [Online]. Available: https://doi.org/10.1145/3307650.3322230

[51] S. Vanecek, M. W. Mußbacher, D. Größler, U. Saroliya, and M. Schulz, "Mt4g: A tool for reliable auto-discovery of nvidia and amd gpu compute and memory topologies," in Proceedings of the SC '25 Workshops of the International Conference for High Performance Computing, Networking, Storage and Analysis, ser. SC Workshops '25. New York, NY, USA: Association for Computing Machinery, 2025, p. 1574-1586. [Online]. Available: https://doi.org/10.1145/3731599.3767518

[52] A. Vaswani, N. Shazeer, N. Parmar, J. Uszkoreit, L. Jones, A. N. Gomez, Ł. Kaiser, and I. Polosukhin, "Attention is all you need," Advances in neural information processing systems, vol. 30, 2017.

[53] P. Vila, B. Köpf, and J. F. Morales, "Theory and practice of finding eviction sets," in 2019 IEEE Symposium on Security and Privacy (SP), 2019, pp. 39-54.

[54] H. Wong, M.-M. Papadopoulou, M. Sadooghi-Alvandi, and A. Moshovos, "Demystifying gpu microarchitecture through microbenchmarking," in 2010 IEEE International Symposium on Performance Analysis of Systems & Software (ISPASS), 2010, pp. 235-246.

[55] B. Zhang and R. Sennrich, "Root mean square layer normalization," Advances in neural information processing systems, vol. 32, 2019.

[56] Z. Zhang, T. Allen, F. Yao, X. Gao, and R. Ge, "Tunnels for bootlegging: Fully reverse-engineering gpu tlbs for challenging isolation guarantees of nvidia mig," in Proceedings of the 2023 ACM SIGSAC Conference on Computer and Communications Security, ser. CCS '23. New York, NY, USA: Association for Computing Machinery, 2023, p. 960-974. [Online]. Available: https://doi.org/10.1145/3576915.3616672

[57] Z. Zhang, K. Cai, Y. Guo, F. Yao, and X. Gao, "Invalidate+Compare: A Timer-Free GPU cache attack primitive," in 33rd USENIX Security Symposium (USENIX Security 24). Philadelphia, PA: USENIX Association, Aug. 2024, pp. 2101-2118. [Online]. Available: https://www.usenix.org/conference/ usenixsecurity24/presentation/zhang-zhenkai