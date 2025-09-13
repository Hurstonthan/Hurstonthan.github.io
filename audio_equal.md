---
layout: page
title: A Basic Audio Equalizer
permalink: /audio-equal.html
---

## Summary

- **Goal** — Build a 3-band audio equalizer using op-amp filters (HPF, BPF, LPF), adjustable inverting gains, summing, and a power stage that can drive a speaker. Target cutoffs at ~320 Hz (LPF) and 3.2 kHz (HPF) at −3 dB; BPF spans 320 Hz–3.2 kHz. Minimum output power > 0.4 W.  
- **ICs** — LM324 (multi-stage filtering, buffers, adjustable gains, summing) and LM386 (power amplifier).  
- **Key Design Choices** — Voltage buffers to isolate cascaded filters; inverting stages for easy gain tuning (potentiometers); non-inverting LM386 output to maintain phase into the speaker.  
- **Reported Results** — Verified −3 dB points near spec (e.g., LPF ~316 Hz; HPF ~3.02 kHz), V_amp ≈ 15 mVrms (min) and ~100 mVrms (max), output power ≈ 1.125 W into 8 Ω; observed noise with heavy-bass material.  

<!-- ![System Overview Placeholder]({{ "/doc/audio_eq/audio_eq_overview.png" | relative_url }})
<p align="center"><em>Figure S.1 — Overall equalizer signal flow (placeholder).</em></p> -->

---

## Architecture / Signal Flow

**Five conceptual stages:** input → filtering (HPF, BPF, LPF) → per-band adjustable gain → recombination via summing amplifier → power amplifier to speaker. Voltage buffers (LM324) decouple filter sections; the LM386 provides final drive.

![Architecture Placeholder]({{ "/doc/audio_eq/audio_pic1.png" | relative_url }})
<p align="center"><em>Figure 1 — Block diagram of stages (placeholder).</em></p>

---

## Filters (HPF, LPF, BPF)

- **Design target:** −3 dB at 320 Hz (LPF) and 3.2 kHz (HPF) using RC sections; BPF formed by cascading HPF + LPF with a buffer between to avoid loading. Example values: C=0.01 µF, R=4.9 kΩ for HPF; larger C for LPF to reach 320 Hz.  
- **Theory:** f_c=1/(2πRC) for 1st-order sections; BPF passband ≈ intersection of HPF/LPF corners.  
- **Measured examples:** HPF ~3.02 kHz at ~−3 dB; LPF ~316 Hz; BPF ~316 Hz and 3.162 kHz at ~−3.2/−3.27 dB.  

![Filters Placeholder]({{ "/doc/audio_eq/audio_pic2.png" | relative_url }})
<p align="center"><em>Figure 2 — HPF/LPF/BPF schematics & target responses (placeholder).</em></p>

---

## Adjustable Gain (Per-Band)

Each band uses an inverting LM324 stage with a 10 kΩ potentiometer in the feedback path for easy gain tuning:  
A_v = -Rf/Rin. Positive input at ground, negative input takes the filter output. Rails at ±5 V.

![Gain Stage Placeholder]({{ "/doc/audio_eq/audio_pic3.png" | relative_url }})
<p align="center"><em>Figure 3 — Adjustable inverting gain stage (placeholder).</em></p>

---

## Summing Amplifier (Recombination & Master Volume)

A summing inverting amplifier (LM324) combines band outputs. Example: Rin=33 kΩ per band and Rf≈68 kΩ (or a pot) to target gain ≈ 2 overall (tunable). This stage doubles as master volume.

<!-- ![Summing Placeholder]({{ "/doc/audio_eq/summing.png" | relative_url }})
<p align="center"><em>Figure 4 — Summing amplifier with adjustable Rf (placeholder).</em></p> -->

---

## Power Amplifier (Speaker Drive)

The LM386 is used as a non-inverting power stage. Output is AC-coupled (e.g., 200 µF) to the speaker (~8 Ω). Target >0.4 W; measured ~1.125 W based on Vrms and P=V^2/R.

<!-- ![Power Stage Placeholder]({{ "/doc/audio_eq/power_stage.png" | relative_url }})
<p align="center"><em>Figure 5 — LM386 output stage and AC coupling (placeholder).</em></p> -->

---

## Materials (Core Parts)

Representative parts: LM324, LM386, 0.01 µF / 0.1 µF / 200 µF capacitors, 4.9 kΩ / 10 kΩ / 33 kΩ / 68 kΩ resistors, 10 kΩ potentiometers, 8 Ω speaker.

<!-- ![BOM Placeholder]({{ "/doc/audio_eq/bom_table.png" | relative_url }})
<p align="center"><em>Figure 6 — BOM snapshot (placeholder).</em></p> -->

---

## Procedure

1) Build filters (HPF, LPF, BPF with buffer).  
2) Add adjustable per-band gains (LM324 inverting stages with 10 kΩ pots).  
3) Combine via summing (LM324 inverting summer; master volume in Rf).  
4) Drive with LM386 to the 8 Ω speaker through AC coupling.  
5) Test with tones (200 Hz, 2 kHz, 10 kHz) and program material; verify −3 dB points, band behavior, Vrms levels, ripple.

<!-- ![Procedure Placeholder]({{ "/doc/audio_eq/procedure.png" | relative_url }})
<p align="center"><em>Figure 7 — High-level wiring & test flow (placeholder).</em></p> -->

---

## Results

- **Filter Verification** — HPF ~3.02 kHz, LPF ~316 Hz, BPF cutoffs near 316 Hz and 3.162 kHz at ~−3 dB.  
- **Level Targets** — V_amp ≈ 15 mVrms (min) and ~100 mVrms (max) across 200 Hz, 2 kHz, 10 kHz; ripple example ≈ 10.9 mV.  
- **Power** — With ~8 Ω speaker, Vrms implies ~1.125 W output (meets >0.4 W requirement).  
- **Observation** — Audible noise/distortion with strong-bass videos; otherwise acceptable.  

[▶︎ Video Demo of Equalizer Output]({{ "https://youtu.be/FEWgP-OD_og" | relative_url }})

---

## Discussion & Interpretation

- **What worked** — Targets for −3 dB frequencies and output power met; adjustable gains and summing behaved as expected.  
- **Issues** — Clipping/noise in some settings; possible causes: LM324 headroom and LM386 non-inverting configuration plus layout/grounding.  
- **Next Steps** — Consider inverting power stage to reduce distortion; increase pot values for finer control; increase output coupling capacitance to reduce low-frequency ripple/noise.

![Discussion Placeholder]({{ "/doc/audio_eq/discussion.png" | relative_url }})
<p align="center"><em>Figure 8 — Example scope captures for discussion (placeholder).</em></p>

---

## References

- Final project report and figures informing specs, values, and results.

![References Placeholder]({{ "/doc/audio_eq/references.png" | relative_url }})
<p align="center"><em>Figure 9 — References (placeholder).</em></p>
