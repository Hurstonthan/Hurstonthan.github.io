---
layout: page
title: UVM Verification Learning
permalink: /uvm_verification.html
---
## UVM Verification — **MAC_rx**

**Environment** (industry‑style separation):
- `xgmii_if` (driver modport) → drives `xgmii_rxd/rxc` (control chars `/S/`, `/T/`, IDLE).  
- `mac_rx_out_if` (monitor modport) → samples `MAC_payload_rcv`, `MAC_valid`, `CRC_flush`, `frame_ok`, `bytes_rcv_len`.  
- `xgmii_sequencer`, `xgmii_driver`, `mac_rx_monitor`, `mac_rx_scoreboard`, `xgmii_agent`, `mac_rx_env`.  
- Scoreboard expects headers & CRC to match configured values (mirrors DUT defaults):  
  `DST=FFFF_FFCC_BBAA`, `SRC=AACC_BBFF_FFFF`, `ETH=0x0800`.

**How to run (Questa example)**
```bash
cd sim
make questa UVM_HOME=$QUESTA_HOME/uvm-1.2

# For picking a specific test:
# vsim -c top +UVM_TESTNAME=mac_rx_smoke_test -do "run -all; quit -f"
```

### Test Matrix & Results

| ID | UVM Class | Scenario | SOF Lane | Headers | CRC | Expected | Observed | Status |
|----|-----------|----------|---------:|---------|-----|----------|----------|--------|
| T1 | `good_frame_seq` (in `mac_rx_smoke_test`) | Good frame (baseline) | 0 | Match | Good | `frame_ok=1`, payload equals sent | As expected | ✅ PASS |
| T2 | `sof_lane4_seq` (in `mac_rx_alignment_and_error_test`) | Misaligned SOF | 4 | Match | Good | `frame_ok=1`, payload equals sent | As expected | ✅ PASS |
| T3 | `crc_error_seq` (in `mac_rx_alignment_and_error_test`) | Bad FCS | 0 | Match | **Bad** | `frame_ok=0` (reject) | As expected | ✅ PASS |
| T4 | `bad_dst_seq` (in `mac_rx_smoke_test`) | Wrong destination MAC | 0 | **Bad DST** | Good | `frame_ok=0` (reject) | As expected | ✅ PASS |

**Notes**
- Driver places `/S/` at `sof_lane` (0 or 4) and streams data; `/T/` is inserted after FCS.  
- Monitor trims last beat using `bytes_rcv_len` when `CRC_flush` asserts; then emits frame with `frame_ok`.  
- Scoreboard pairs *expected* (from Driver AP) vs *observed* (from Monitor AP) in order.

### Logs / Seeds
- Build invokes `-sv_seed random` (tool‑controlled).  
- UVM verbosity: `UVM_MEDIUM` by default (override with `+UVM_VERBOSITY=UVM_HIGH`).  
- Enable config DB tracing with `+UVM_CONFIG_DB_TRACE` if debugging VI bindings.

### Coverage (functional checklist)
- [x] Good frame, SOF lane0
- [x] Good frame, SOF lane4 alignment
- [x] CRC error drop
- [x] Bad destination drop
- [ ] VLAN‑tagged frames
- [ ] Early `/T/` (runt) / code violation
- [ ] Jumbo frames
- [ ] Inter‑frame gap stress
- [ ] Randomized payload lengths and seeds (constrained)

### Next Steps
- Add sequences for VLAN (single & stacked), early `/T/`, jumbo, and randomized payload sizes.  
- Add covergroups for SOF lane distribution, header match/mismatch bins, CRC pass/fail, and IFG spacing.  
- Gate a continuous regression on random seeds and wider payload distributions.

---

## Implementation and Testing
- UVM TB (Questa/VCS/Xcelium/Riviera) with industry‑style class separation and two tests:  
  `mac_rx_smoke_test`, `mac_rx_alignment_and_error_test`.
- DUT + TB wiring in `tb/top.sv`; virtual interfaces set via UVM config DB.

## Results
- All four MAC_rx scenarios above **PASS** under the scoreboard rules.  
- Waveforms recommended for debug (e.g., `frame_ok`, `CRC_flush`, `MAC_valid`, `bytes_rcv_len`).

<!--
- **MAC**
  ![MAC Result]({{ "/doc/fpga/MAC_result.png" | relative_url }})
  <p align="center"><em>Figure M.2 — MAC datapath and CRC frame verification.</em></p>
-->

---

## Source

Code on GitHub → <https://github.com/Hurstonthan/UVM_Learning/tree/main>