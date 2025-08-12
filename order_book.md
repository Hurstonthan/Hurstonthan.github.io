---
layout: page
title: Limit Order Book (FPGA Matching Engine)
permalink: /order_book.html
---

## Summary

- **Snapshot** — Price–time matching, OUCH/SoupBinTCP, L2/L3 depth.
- **Reference Architecture** — CloudEx-style exchange pipeline (gateways → admission/risk → global **sequencer** → matching engine per shard → market data publisher) [CloudEx].  
- **Implementation Choice** — **SystemVerilog** RTL with micro-pipelined stages and per-price FIFOs (L3) plus level aggregates (L2). Protocols: OUCH over SoupBinTCP at ingress [OUCH-5.0] [SoupBinTCP-4.0].

<!-- ![Summary]({{ "/doc/lob-summary.png" | relative_url }}) -->

---

## Industrial Reference Architecture (CloudEx)

We adopt the **CloudEx** layout:  
(1) **Gateways** handle participant links and buffering;  
(2) **Admission/Risk** validates orders;  
(3) a single-writer **Sequencer** imposes strict total order;  
(4) the **Matching Engine** applies the ordered stream to symbol shards;  
(5) the engine **publishes market data** (trades, L2/L3 snapshots).  
This separation of connectivity, ordering, matching, and dissemination underpins fairness and reproducibility [CloudEx].

<!-- ![CloudEx Architecture]({{ "/doc/lob-architecture.png" | relative_url }}) -->

---

## Protocols & Ingress (OUCH over SoupBinTCP)

- **Order entry:** OUCH fixed-length messages for new/replace/cancel and execution reports [OUCH-5.0].  
- **Session/transport:** OUCH commonly runs over **SoupBinTCP**, which delivers sequenced, ordered messages with recovery across reconnects [SoupBinTCP-4.0].  
These choices match widely used exchange connectivity and align with the CloudEx gateway role [CloudEx].

<!-- ![Protocols]({{ "/doc/lob-protocols.png" | relative_url }}) -->

---

## Matching Policy (Price–Time Priority)

Continuous trading uses **price–time priority (FIFO within price)**: best price first; within the same price, earlier orders match before later ones. This is the baseline algorithm at major venues (see CME overview and rules) [CME-Overview] [CME-Rules].

<!-- ![Price–Time]({{ "/doc/lob-priority.png" | relative_url }}) -->

---

## Book Data Model (L2/L3)

- **Per-symbol shards** to scale cores independently.  
- **Price grid** (tick-mapped) with **per-level FIFO** (L3 orders) and **aggregates** (L2 size/count).  
- **Crossing** consumes best levels, emits fills, and rests residual quantity.  
This mirrors order-book structure discussed in exchange/ME literature: per-symbol books with buy/sell lists ordered by price, FIFO within level, and publishers emitting trades and book snapshots [Chronicle-ME] [CloudEx].

<!-- ![Book Model]({{ "/doc/lob-data-model.png" | relative_url }}) -->

---

## Core RTL Implementation (SystemVerilog)

**Pipeline (1–few cycles/stage):**  
1) Parse/normalize (SoupBinTCP → OUCH → internal command)  
2) Admission & risk checks (limits, duplicates, permissions)  
3) **Sequencer** (monotonic per-symbol order index)  
4) Route to symbol shard  
5) Match (price–time loop; partial fills; cancel/replace)  
6) Book update (L3 queues + L2 aggregates)  
7) Publish (ACK/NACK, exec reports, L2/L3 updates)

The **sequencer-before-matching** pattern is directly adopted from CloudEx to guarantee deterministic total order; the **publisher** produces trades and LOB snapshots as described there [CloudEx].

<!-- ![Core RTL]({{ "/doc/lob-core-rtl.png" | relative_url }}) -->

---

## Market Data Dissemination

The matching engine assigns release times and **publishes trades and periodic LOB snapshots** back to participants via the gateways (per-symbol subscriptions) [CloudEx].  
(For reference message formats, see ITCH specs used by several venues.) [ITCH] [ASX-ITCH]

<!-- ![Market Data]({{ "/doc/lob-marketdata.png" | relative_url }}) -->

---

## Verification Plan

- **Unit tests:** parser (SoupBinTCP/OUCH), risk, sequencer, matching edge cases (partials, cancels, replaces).  
- **Determinism:** permute arrival order → identical final book and event stream under the sequencer [CloudEx].  
- **Soak:** steady and bursty OUCH loads to validate backpressure and recovery (OUCH/SoupBinTCP behavior) [OUCH-5.0] [SoupBinTCP-4.0].

<!-- ![Verification]({{ "/doc/lob-verification.png" | relative_url }}) -->

---

## Citations (Named References)

- **[CloudEx]**: *CloudEx: Fast, Consistent, and Available Matching in the Cloud* (HotOS’21 / SIGOPS PDF) — gateways, sequencer, matching engine, market-data release.  
- **[OUCH-5.0]**: Nasdaq OUCH 5.0 — order entry spec (fixed-length messages).  
- **[SoupBinTCP-4.0]**: Nasdaq SoupBinTCP 4.0 — session/transport spec (sequenced delivery + recovery).  
- **[Chronicle-ME]**: Chronicle Matching Engine (technical report) — component hierarchy, per-symbol books, dissemination.  
- **[ITCH]**: Nasdaq TotalView-ITCH — market data (order-level depth, trades).  
- **[ASX-ITCH]**: ASX ITCH Message Specification — market data feed using SoupBinTCP framing.  
- **[CME-Overview]**: CME Matching Algorithm Overview — price–time baseline.  
- **[CME-Rules]**: CME/CBOT Rulebook excerpt — priority definitions and examples.

<!-- Link definitions -->
[CloudEx]: https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s06-ghalayini.pdf
[OUCH-5.0]: https://nasdaqtrader.com/content/technicalsupport/specifications/TradingProducts/Ouch5.0.pdf
[SoupBinTCP-4.0]: https://www.nasdaq.com/docs/SoupBinTCP%204.0.pdf
[Chronicle-ME]: https://chronicle.software/wp-content/uploads/2023/01/Chronicle-Matching-Engine.pdf
[ITCH]: https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHSpecification.pdf
[ASX-ITCH]: https://www.asxonline.com/content/dam/asxonline/public/documents/asx-trade-refresh-manuals/asx-trade-itch-message-specification.pdf
[CME-Overview]: https://www.cmegroup.com/education/matching-algorithm-overview.html
[CME-Rules]: https://www.cmegroup.com/rulebook/CBOT/I/5.pdf
