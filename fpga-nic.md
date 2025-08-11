---
layout: page
title: FPGA network Offload NIC
permalink: /fpga-nic.html
---

## FPGA Network Offload NIC

| Metric | Value |
|--------|-------|
| Device | Xilinx Kintex-7 325T |
| Latency | **< 450 ns** RTT (wire-to-wire) |
| Throughput | 10 GbE line-rate |

---

## Summary

- **AXI4-Stream Paths** — RX and TX run on AXIS 64-bit AXIS stream, cut-through framing, and clean handshakes (VALID/READY) to upper pipelines.
- **MAC Layer** — 10 GbE PCS/PMA interface (64b datapath, 8b control), parallel CRC-32, padding to minimum Ethernet frame, 802.3x PAUSE generate.
- **IP Layer** — IPv4 with header checksum, TCP/UDP support,.
- **TCP Layer** — Receiver, Transmitter, and Flow Logic with RFC 793+ behaviors (OOO handling, duplicate trimming, fast retransmit with duplicate ACKs).
- **Payload FIFO RX** — BRAM-based OOO buffer delivering in-order bytes to apps, with read-pointer tracking cache.
- **FIFO TX** — BRAM-based in-order buffer, base checksum accumulation, TX cache for segment pointers & fast retransmit.

![System Overview]({{ "/doc/network_top.png" | relative_url }})
<p align="center"><em>Figure S.1 — System-level overview.</em></p>


## Block Diagram

*(insert diagram or description here)*

![Block Diagram]({{ "/doc/block-diagram.png" | relative_url }})
<p align="center"><em>Figure B.1 — Top-level NIC block diagram.</em></p>

---

## AXI4-Stream (AXIS) RX/TX Paths

The NIC exposes **AXIS** at both TX and RX datapath. Frames are received on RX (XGMII→AXIS) and sent on TX (AXIS→XGMII). Control uses **VALID/READY**; framing uses **LAST**;

![AXIS Overview]({{ "/doc/axis-overview.png" | relative_url }})
<p align="center"><em>Figure A.1 — AXIS bridging at RX/TX boundaries.</em></p>

**RX Path (XGMII → AXIS)**  
- Start-of-frame detected from XGMII control; CRC verified in parallel.  
- Payload and headers stream out on **TDATA[63:0]**; **TLAST** asserted on final xgmii data.  
- **TKEEP[7:0]** marks valid bytes on the last cycle (handles non-aligned frame tails).  
- Cut-through mode: data can exit before full-frame CRC completes (configurable).

**TX Path (AXIS → XGMII)**  
- Application/TCP hands data via **AXIS TX**; **LAST** define segment boundaries.  
- Hardware pads payload to meet 46-byte Ethernet minimum when needed.  
- CRC-32 appended.  
- Back-pressure: deassert **READY/VALID** on internal congestion; upstream stalls cleanly.

---

## MAC Layer

Supports a 10 GbE interface from the AMD PCS/PMA with a 64-bit data path and an 8-bit byte control signal.  
Implements parallel CRC computation (64-bit input, 32-bit output).  
Supports zero-padding to ensure the minimum Ethernet frame size.  
Implements back-pressure handling by generating PAUSE frames.

![MAC Layer]({{ "/doc/mac-layer.png" | relative_url }})
<p align="center"><em>Figure M.1 — MAC datapath, CRC, PAUSE flow.</em></p>

**Details**
- 64-bit XGMII data + 8-bit control alignment at line rate (156.25 MHz transmitting clock).  
- CRC-32 computed in parallel to avoid serialization stalls.  
- Padding logic ensures ≥46 B Ethernet payload when needed.  
- PAUSE frame generation on ingress/egress pressure; AXIS **TREADY** deassertion propagates safely.

---

## IP Layer

Supports IPv4, including header checksum calculation.  
Supports TCP and UDP protocols.  

![IP Layer]({{ "/doc/ip-layer.png" | relative_url }})
<p align="center"><em>Figure I.1 — IP parsing pipeline and checksum unit.</em></p>

**Details**
- IPv4 header checksum folding and finalize in-line.  
- Protocol demux for TCP/UDP with minimal added latency.  
- Stateless IPv6 header parse to feed upper layers.  
- Optional 802.1Q VLAN tag extraction & pass-through via AXIS sideband.

---

## TCP Layer

The TCP layer consists of three main modules — **TCP Receiver**, **TCP Transmitter**, and **TCP Flow Logic** — along with supporting modules.

![TCP Layer]({{ "/doc/tcp-layer.png" | relative_url }})
<p align="center"><em>Figure T.1 — TCP RX/TX/Flow control orchestration.</em></p>

### TCP Receiver
Parses and extracts the sequence number, acknowledgment number, checksum, receive window, and control flags from incoming TCP segments. Communicates with TCP Flow Logic to process packets in compliance with RFC 793+, ignoring duplicate data, handling out-of-order packets, and trimming duplicate bytes.

### TCP Transmitter
Constructs TCP segments, computes TCP checksums, and transmits TCP headers and payloads to the IP layer. Communicates with TCP Flow Logic to ensure valid header generation and to handle fast retransmission logic. Segments are streamed over **AXIS TX** with **TLAST/TKEEP** semantics.

### TCP Flow Logic
Acts as the controller for the TCP layer, coordinating between the TCP Receiver, TCP Transmitter, and FIFO modules. Ensures that application modules receive in-order, valid packets, and that outgoing packets are valid. Implements window-based flow control by adjusting the advertised receive window in response to back-pressure (also controlling **TREADY** on AXIS).

**Details**
- Hardware duplicate-ACK detection triggers fast retransmit.  
- Out-of-order acceptance with in-order delivery to applications.  
- Window updates reflect RX buffer pressure to peers.  
- Per-connection state designed for deterministic timing.

---

## Payload FIFO RX

BRAM-based out-of-order FIFO that stores received TCP payloads.  
Delivers in-order packets to application modules (e.g., SoupBinTCP) by tracking the number of application bytes read (`seq_rd_trk`).  
Communicates with TCP Flow Logic to manage a small cache array that stores the read pointer for the next in-order byte.

![Payload FIFO RX]({{ "/doc/payload-fifo-rx.png" | relative_url }})
<p align="center"><em>Figure R.1 — RX FIFO with OOO accept / in-order release.</em></p>

**Details**
- Segment placement by sequence number; duplicate/overlap trimming.  
- Read-pointer cache assists low-latency in-order drain to apps via **AXIS RX**.  
- Back-pressure signals propagate to flow/window control and AXIS **TREADY**.

---

## FIFO TX

BRAM-based in-order FIFO that stores payloads from application modules or SoupBinTCP.  
Calculates a base checksum sum in parallel as data is written for TCP checksum computation.  
On each transmission, stores the pointer range of the transmitted segment along with its base checksum sum in a TX cache.  
On fast retransmission requests from TCP Flow Logic, retrieves the earliest unacknowledged segment pointers and checksum sums from the TX cache and retransmits the data.  
When a new acknowledgment number is received, updates the acknowledged byte count and removes the corresponding segments from the TX cache.

![FIFO TX]({{ "/doc/fifo-tx.png" | relative_url }})
<p align="center"><em>Figure X.1 — TX FIFO with base-sum accumulation and TX cache.</em></p>

**Details**
- Base checksum accumulation amortizes per-segment TCP checksum finalize.  
- TX cache accelerates selective/fast retransmit without recompute.  
- Clean AXIS **TLAST** segmentation; **TKEEP** masks short tails; **TREADY** honors downstream pressure.

---

## Source

Code on GitHub → <https://github.com/Hurstonthan/fpga-tcp-nic>
