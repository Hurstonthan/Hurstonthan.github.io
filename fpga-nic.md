---
layout: page
title: FPGA TCP Offload NIC
permalink: /fpga-nic.html
---

## FPGA TCP Offload NIC

| Metric | Value |
|--------|-------|
| Device | Xilinx Kintex-7 325T |
| Latency | **< 450 ns** RTT (wire-to-wire) |
| Throughput | 10 GbE line-rate |


### Highlights
* Full hardware SYN / ACK handshake  
* Sliding-window retransmit with duplicate-ACK detection  
* Parallel CRC-32 (64-bit datapath, 4-cycle)  
* Uses custom low-jitter PCS / PMA for deterministic latency  

### Block-Diagram
*(insert diagram or description here)*

### Source
Code on GitHub → <https://github.com/Hurstonthan/fpga-tcp-nic>
