---
layout: page
title: Dual-Core Pipelined CPU
permalink: /in-order-cpu.html
---

## Summary

- **Top Level** — Two in-order, five-stage pipelines (IF, ID, EX, MEM, WB). Data hazards are resolved by a forwarding unit; remaining load–use cases stall via a hazard unit. The pipeline control was extended to recognize **LW.R** / **SW.C** for atomic sequences.
- **Branch Prediction** — Dynamic 2-bit saturating-counter predictor (per-PC entry) with flush on misprediction.  
- **D-Cache (32-bit system address)** — 128 B, 2-way set-associative, 2-word blocks, write-back + write-allocate, dirty bit, LRU; per-line **MSI** coherence state.
- **I-Cache** — Direct-mapped, 1-word blocks; report figure shows a 16 B configuration that streams instructions to the datapath.
- **Coherence & Atomicity** — Snooping bus with invalidate/transfer actions, per-line MSI, and **LR/SC-style** atomicity via **reservation set** and **valid bit**; bus controller prioritizes writes and coordinates snoops.
- **Results** — Merge-sort benchmark (single/dual-thread) used to extract CPI, Fmax, execution time (Iron’s Law); up to ~4.3× speedup with pipelining, caches, and coherence versus single-cycle baseline.

![System Overview]({{ "/doc/dual_core/dual_core_top.png" | relative_url }})
<p align="center"><em>Figure S.1 — Dual-core microarchitecture overview.</em></p>

---

## Top-Level Pipeline (Five Stages and Hazard Handling)

Each core implements a five-stage pipeline:

1) **IF** — Instruction fetch and next-PC selection.  
2) **ID** — Decode, register read, immediate generation, branch target calculation.  
3) **EX** — ALU operations, branch resolution, effective address calculation.  
4) **MEM** — D-cache access.  
5) **WB** — Register writeback.

**Data hazards.** The forwarding unit supplies EX operands from EX/MEM and MEM/WB; a **load–use** interlock inserts a single bubble when the loaded value is needed by the immediately following instruction. For **LW.R / SW.C**, the hazard unit adds selects and control bits to steer `dmemaddr` and ensure correct interlocks.

**Control hazards.** The predictor provides the speculative next PC in IF; on misprediction detected in EX, the pipeline flushes younger stages.

<!-- ![Pipeline and Hazards]({{ "/doc/dual_core/dc-pipeline-hazards.png" | relative_url }})
<p align="center"><em>Figure 1 — Five-stage pipeline with forwarding and load–use stall.</em></p> -->

---

## Branch Prediction (2-Bit Saturating Counter)

The predictor uses a per-PC **2-bit saturating counter**:

- `00` strongly not-taken → fall-through; `11` strongly taken → predicted target.  
- Counter increments on taken, decrements on not-taken (saturating).  
- Mispredictions flush IF/ID and redirect the PC.

<!-- ![Branch Prediction]({{ "/doc/dual_core/dc-branch-2bit.png" | relative_url }})
<p align="center"><em>Figure 2 — Two-bit BHT predictor and recovery.</em></p> -->

---

## Data Cache (32-Bit System Address)

**Organization.** 128 B capacity, **2-way** set-associative, **2-word (8 B)** blocks; write-back + write-allocate; **dirty** and **LRU** per set; each line holds an **MSI** state.

**Access flow.** Tag compare on both ways; read hits return data in MEM; write hits set dirty. Read misses allocate/fill; write misses allocate and may write back evicted dirty lines.

**Coherence hooks.** D-cache permits store hits **only in M state**; snoop operations are processed in designated states; dirty bookkeeping and whole-cache flush at halt are specified in the design notes.

![D-Cache]({{ "/doc/dual_core/dual_core_dcache.png" | relative_url }})
<p align="center"><em>Figure 1 — D-cache arrays, comparators, LRU, and write-back path.</em></p>

---

## Instruction Cache

Direct-mapped, **1-word blocks**; the report figure instantiates **16 B** total and continuously supplies instructions to the datapath.

<!-- ![I-Cache]({{ "/doc/dual_core/dc-icache.png" | relative_url }})
<p align="center"><em>Figure 4 — I-cache lookup and refill.</em></p> -->

---

## Coherence and Atomic Writes

**Protocol.** Per-line **MSI** is implemented in each D-cache. On entering a **snoop** phase, the cache compares `snoop_address` against both ways’ tags. If a snoop **hit** finds a **dirty (M)** line, a **transfer** is signaled (`cctrans`) and the line is written back before transitioning to **S**; otherwise an explicit **invalidate** (`ccinv`). Local data writes assert `ccwrite`. This ensures only one writer (M owner) holds the most recent copy, with updates propagated over the bus via `cctrans/ccwrite/ccinv/ccsnoopaddr`.

**Bus controller.** The coherence controller (shared bus) prioritizes **writes over reads**, arbitrates simultaneous requests with **LRU**, and coordinates snoops: (a) issue **invalidation** on conflicting writes; (b) **source data from peer cache** when the peer holds the line in **M**, using `cctrans` to identify dirty ownership; otherwise fetch from RAM; and (c) a **SNOPWAIT1** stage aligns both peers and the controller at the same snoop cycle.

**Atomicity (LR/SC-style).** The design introduces **LW.R** (load-reserved) and **SW.C** (store-conditional) with a **reservation set** and **valid bit** that hold the lock address. On **LW.R**, the control raises `atomic` and records the address; on **SW.C**, if `dmemaddr` matches the reservation, the set is invalidated and the result is returned (report notes a `0/1` result convention), and **any write by another core to the same address invalidates the reservation**. The hazard unit and datapath add selects for these opcodes.

![Coherence and Atomics]({{ "/doc/dual_core/dual_core_msi.png" | relative_url }})
<p align="center"><em>Figure 2 — MSI states, snoop paths, and LR/SC-style reservation.</em></p>

---

## Results Report

**Method.** We used single-thread and dual-thread **merge sort** to compare single-cycle, pipelined (no caches), pipelined (with caches), and multicore (with caches). We measured **Fmax**, **cycles**, and computed **CPI** and **execution time** (Iron’s Law).

**Findings.** Pipelining + caches + coherence delivered up to ~**4.3×** speedup at higher memory latency; benefits depend on hit rate, misprediction rate, and residual load–use stalls.

![Results]({{ "/doc/dual_core/dual_core_result.png" | relative_url }})
<p align="center"><em>Figure 3 — Fmax frequency for dual-cores architecture.</em></p>
