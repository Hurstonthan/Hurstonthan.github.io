---
layout: page
title: In-Order Computer Architecture
permalink: /in-order-cpu.html
---

## In-Order Computer Architecture (RISC-V)

| Pipeline Stage | Function |
|----------------|----------|
| IF | 2-cycle fetch with branch predictor |
| ID | Register read, hazard detect |
| EX | ALU, multiplier, CSR |
| MEM | Data cache (1 cycle) |
| WB | Write-back |

### Highlights
* Forwarding & hazard unit – no NOPs in steady state  
* Static branch predictor (2-bit) reaches 93 % accuracy on CoreMark  
* Synthesized at 300 MHz on Kintex-7, 25 k LUTs  
* Formal proof with SymbiYosys for ISA compliance

### Next Steps
* Add simple superscalar issue for ≥ 1.6 IPC  
* Integrate with your DDR5 controller for SoC demo
