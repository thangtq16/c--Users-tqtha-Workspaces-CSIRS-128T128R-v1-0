---
name: CSIRS 128T128R Project Overview
description: 5G Advanced PHY simulation project - CSI-RS and CSI Feedback from Rel-15/16 to Rel-19 for 128T128R massive MIMO
type: project
---

Main MATLAB file: `CSIRS_128T128R_v1_0.m` in the project root.
Author: ThangTQ23 - VSI, Date: 2026-03

**System Configuration:**
- Carrier: 52 PRBs, SCS 30 kHz (10 MHz), n78 (3.5 GHz)
- Antenna: 128T (gNB: 4V×16H×2pol) x 4R (UE: 1V×2H×2pol)
- CSI-RS: 4 NZP-CSI-RS Resources × 32 ports (Row 18, CDM8 = FD-CDM2 × TD-CDM4)
- Layout: 2-slot (Slot 0: R0 l0=2, R1 l0=8; Slot 1: R2 l0=2, R3 l0=8)
- Channel: CDL-B (changed from CDL-C for rank-4 test), DS=100ns, Doppler=5Hz, SNR=20dB
- Custom library paths: `5g/` and `wirelessnetwork/`

**4 Approaches (selectedApproach = 'ALL'):**
- A: Rel-15/16 Per-32-Port Type I Single Panel (TS 38.214 S5.2.2.2.1), struct-based reportConfig, PanelDimensions=[8,2]
- B: Full 128-port SVD upper bound (ideal reference), uses svd(H_wb,'econ')
- C: Rel-19 Type I Single-Panel Mode A (TS 38.214 S5.2.2.2.1a), CodebookType='typeI-SinglePanel-r19', CodebookMode=1, PanelDimensions=[1,16,4]
- D: Rel-19 Type I Single-Panel Mode B (TS 38.214 S5.2.2.2.1a), same config as C but CodebookMode=2

**Pipeline (12 sections):**
1. System Config → 2. CSI-RS Resource Config → 3. Signal Gen & Grid Map →
4. Grid Visualization → 5. CDL Channel Model → 6. TX through Channel + AWGN →
7. Timing Sync + OFDM Demod → 8. Channel Estimation (per resource) →
9. CE Error Analysis (NMSE) → 10. CSI Feedback (PMI/RI/CQI for A/B/C/D) →
11. Beamforming Pattern Visualization → 12. Summary Table

**Key internal functions used:**
- `nr5g.internal.nrRISelect`, `nr5g.internal.nrCQISelect` (for Approach A)
- `nr5g.internal.nrPMIReport` (for Approach C & D)
- `nrChannelEstimate` with CDMLengths=[2,4] (Row 18 CDM8)

**Why:** Goal is to evaluate gain of Rel-19 Refined Type I codebook over Rel-15/16 for massive MIMO 128T128R. Known limitation: ModeA TypeIRel19 gives RI=1 with CDL-C; CDL-B was introduced to test higher rank.

**Current terminal output state (as of 2026-03-30): user is satisfied.**
Key output fixes applied in this session:
- Approach A: added capacity column (SVD per 32-port sub-block, select max → R2 = 18.18 bits/s/Hz)
- Approach B: replaced single `Singular values` line with per-SV table (SV#, Singular val, SNR dB, Cap bits/s/Hz)
- Approach C/D: changed `bpcu` → `bits/s/Hz`, added header "RI selection (capacity per rank candidate):"
- Comparison table: removed duplicate SVD block, added Approach A row, all 4 approaches now comparable
- Summary block: all approaches now show RI / Cap / CQI in consistent format

**How to apply:** When suggesting changes, keep the 12-section pipeline structure. Use `nr5g.internal.*` namespace for internal PMI/RI/CQI functions. Approach B (SVD) is always the upper bound reference for capacity comparison. Do NOT revert terminal output formatting — user has approved current state.
