---
layout: page
title: Dual-Core Pipelined CPU
permalink: /in-order-cpu.html
---

## Summary

- **Top Level** — Two in-order, five-stage pipelines (IF, ID, EX, MEM, WB). Data hazards are resolved by a forwarding unit; remaining load–use cases stall via a hazard unit. The pipeline control was extended to recognize **LW.R** / **SW.C** for atomic sequences. [^1]  
- **Branch Prediction** — Dynamic 2-bit saturating-counter predictor (per-PC entry) with flush on misprediction.  
- **D-Cache (32-bit system address)** — 128 B, 2-way set-associative, 2-word blocks, write-back + write-allocate, dirty bit, LRU; per-line **MSI** coherence state. [^2]  
- **I-Cache** — Direct-mapped, 1-word blocks; report figure shows a 16 B configuration that streams instructions to the datapath. [^3]  
- **Coherence & Atomicity** — Snooping bus with invalidate/transfer actions, per-line MSI, and **LR/SC-style** atomicity via **reservation set** and **valid bit**; bus controller prioritizes writes and coordinates snoops. [^4] [^5] [^6]  
- **Results** — Merge-sort benchmark (single/dual-thread) used to extract CPI, Fmax, execution time (Iron’s Law); up to ~4.3× speedup with pipelining, caches, and coherence versus single-cycle baseline. [^7] [^8] [^9]

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

**Data hazards.** The forwarding unit supplies EX operands from EX/MEM and MEM/WB; a **load–use** interlock inserts a single bubble when the loaded value is needed by the immediately following instruction. For **LW.R / SW.C**, the hazard unit adds selects and control bits to steer `dmemaddr` and ensure correct interlocks. [^10]

**Control hazards.** The predictor provides the speculative next PC in IF; on misprediction detected in EX, the pipeline flushes younger stages.

<!-- ![Pipeline and Hazards]({{ "/doc/dual_core/dc-pipeline-hazards.png" | relative_url }})
<p align="center"><em>Figure 1 — Five-stage pipeline with forwarding and load–use stall.</em></p> -->

---

## Branch Prediction (2-Bit Saturating Counter)

The predictor
