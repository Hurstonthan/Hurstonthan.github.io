---
layout: page
title: DDR4 DRAM Memory Controller
permalink: /ddr4-controller.html
---

## Summary

- **Top-Level Design** — Modular DDR4 controller for the accelerated matrix processor. It separates initialization, command sequencing, timing, address mapping, data transfer, and signal generation.  
- **Scheduler Buffer** — Proposed for future non-blocking operation to reorder requests and reduce timing overhead.  
- **FSM Controller** — Sequences activate, read, write, precharge, and refresh while meeting DDR4 timing limits.  
- **Issue Command Module** — Drives ACT_n, RAS_n, CAS_n, WE_n and address/bank lines with one-cycle commands and Deselect gaps.  
- **Address Mapping Policy** — Ra–R–B–BG–C mapping to increase parallelism and row-buffer hits.  
- **Timing Control** — Counter-based timing for ACT, READ, WRITE, PRECHARGE, and REFRESH; maintains average tREFI.  
- **Row Policy** — Open-row policy to favor spatial locality and reduce access latency.  
- **Implementation** — RTL verified with Micron DDR4 models; write-centering and burst handling implemented.  
- **Results** — Verified power-up, initialization, read and write cycles, and multi-chip data assembly.  
- **Proposed Architecture** — Modular structure eases updates for other DDR standards and optimized-version non-blocking upgrades.

![System Overview]({{ "/doc/ddr5/dram_top.png" | relative_url }})
<p align="center"><em>Figure 1 — System-level overview.</em></p>

---

## Top-Level Design

The controller connects AMP0 to off-chip DDR4. It includes:
- **Control Unit** for initialization, command FSM, timing, refresh, and address mapping.  
- **Data Transfer Unit** for double-data-rate reads and writes.  
- **Signal Generator** for DDR4-compliant command and address signaling.

<!-- ![Top Level]({{ "/doc/ddr5/ddr4-top-level.png" | relative_url }})
<p align="center"><em>Figure 1 — Top-level controller organization.</em></p> -->

---

## Scheduler Buffer

A scheduler buffer was planned for a non-blocking version. It would collect requests, reorder them to reduce timing gaps, and return status when complete. The current controller is blocking, but the design leaves room to add the scheduler later to communicate with cross-bar module and non-blocking memory requests.

<!-- ![Scheduler Buffer]({{ "/doc/ddr5/ddr4-scheduler.png" | relative_url }})
<p align="center"><em>Figure 2 — Scheduler buffer concept for future non-blocking mode.</em></p> -->

---

## FSM Controller

The command FSM manages the sequence:
- **Idle** (banks precharged) → **Activate** (row open) → **Read/Write** → **Precharge** (if row change or refresh) → **Refresh** → **Idle**.  
- It uses timing-complete flags (e.g., tACT_done, tRD_done, tWR_done, tRP_done) and a refresh request signal.  
- It tracks which row is open per bank to avoid unnecessary activates.

![Command FSM]({{ "/doc/ddr5/dram_fsm.png" | relative_url }})
<p align="center"><em>Figure 3 — FSM for activate/read/write/precharge/refresh.</em></p>

---

## Issue Command Module

The issue module converts FSM states and mapped addresses into DDR4 signals. Each valid command is driven for one clock. A Deselect command is inserted between valid commands as required. Address and bank lines are used as either command or address pins depending on ACT_n and command type.

<!-- ![Signal Generation]({{ "/doc/ddr5/ddr4-signal-gen.png" | relative_url }})
<p align="center"><em>Figure 4 — One-cycle command issue with Deselect spacing.</em></p> -->

---

## Address Mapping Policy

The controller maps the system address to DDR4 fields as **Ra–R–B–BG–C** to increase parallelism and row hits.  
- **Ranks** are the most significant bits to reduce frequent rank switching.  
- **Row** bits are higher to avoid frequent row opens/closes.  
- **Bank Group/Bank** bits follow rows to interleave across groups and banks.  
- **Column** bits are least significant so sequential addresses target the same open row.  
- For x4/x8 devices, two bank-group bits are placed to promote interleaving; x16 uses one bank-group bit.

![Address Mapping]({{ "/doc/ddr5/dram_addr_map.png" | relative_url }})
<p align="center"><em>Figure 5 — Address breakdown (Ra–R–B–BG–C) for parallel access.</em></p>

---

## Timing Control

Timing is counter-based per FSM state:
- **Activate**: load **tRCD**; signal tACT_done when complete.  
- **Read**: load **tAL + tCL + tBURST**; signal tRD_done.  
- **Write**: load **tAL + tCWL + tBURST**; signal tWR_done.  
- **Precharge**: load **tRP**; signal tRP_done.  
- **Refresh**: schedule by average **tREFI** (with limited push/pull); load **tRFC** during refresh and signal tREF_done.

<!-- ![Timing Control]({{ "/doc/ddr5/ddr4-timing-control.png" | relative_url }})
<p align="center"><em>Figure 6 — Counter-based timing for each command phase.</em></p> -->

---

## Row Policy

The controller uses an **open-row** policy. After an access, the row stays open to serve more hits to the same row. This lowers latency for workloads with spatial locality. The trade-off is higher power and extra precharge time when switching rows. A closed-row policy is not used in this version.

![Row Policy]({{ "/doc/ddr5/row_open.png" | relative_url }})
<p align="center"><em>Figure 7 — Open-row policy to favor locality.</em></p>

---

## Implementation

The RTL was verified with the Micron DDR4 model in Questasim. Key items:
- Full power-up and initialization sequence, including MRS programming and ZQ calibration.  
- Data transfer unit supports burst length 8 with write centering (sampling on the opposite clock edge) and a two-cycle preamble.  
- Write masking via **DM_n** prevents unwanted overwrites within a burst.  
- A four-chip x8 setup forms a 32-bit data path for demonstration.

<!-- ![Implementation]({{ "/doc/ddr5/ddr4-implementation.png" | relative_url }})
<p align="center"><em>Figure 8 — Testbench and model-based verification.</em></p> -->

---

## Results

- **Initialization** — The device completed reset, clock start, CKE enable, mode register sets (MR3→MR6→MR5→MR4→MR3’→MR2→MR1→MR0), and a long ZQ calibration before entering normal operation.  
  ![Init Result]({{ "/doc/ddr5/dram_init.png" | relative_url }})
  <p align="center"><em>Figure 8 — Initialization progress.</em></p>

- **Write Cycle** — After ACT, a write was issued. Data were masked to the target burst with **DM_n**. The controller then precharged the bank.  
  ![Write Result]({{ "/doc/ddr5/dram_writing_cycle.png" | relative_url }})
  <p align="center"><em>Figure 9 — Writing cycle.</em></p>

- **Read Cycle** — A read to the same address returned data from four chips (x8 each). The words were assembled into a 32-bit value and delivered on the bus.  
  ![Read Result]({{ "/doc/ddr5/dram_reading.png" | relative_url }})
  <p align="center"><em>Figure 10 — Reading cycle.</em></p>

---

## Proposed Architecture

The modular design allows changes to timing tables and the signal generator to target other DDR generations. A non-blocking version can add the scheduler buffer and a split-transaction bus. For strict process limits on clock speed, a two-phase technique (e.g., Johnson counter) may be used to meet double-data-rate timing without doubling the clock.

![Proposed Architecture]({{ "/doc/ddr5/dram_prototype.png" | relative_url }})
<p align="center"><em>Figure 11 — Modular architecture with paths to non-blocking and other DDRs.</em></p>
