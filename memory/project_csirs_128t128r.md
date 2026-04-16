---
name: CSIRS 128T128R Project Overview
description: 5G Advanced PHY simulation project — từ CSI-RS link-level (v1/v2) đến NR Cell MU-MIMO system-level (v3), 128T128R massive MIMO, Rel-15/16 → Rel-19 eTypeII
type: project
originSessionId: 45e131e7-bcbe-4b79-8f02-c91c8dbc1be7
---
## Tổng quan

Project đánh giá và so sánh hiệu năng các codebook thế hệ khác nhau (Rel-15/16 Type I → Rel-19 eTypeII refined) trên hệ thống 128T128R massive MIMO downlink.
**Author:** ThangTQ23 — VSI, 2026-03/04

---

## Cấu trúc file chính

| File | Mô tả |
|---|---|
| `CSIRS_128T128R_v1_0.m` | Link-level: 4 approaches (A/B/C/D), single UE, 1 channel realization |
| `CSIRS_128T128R_v2_0.m` | Link-level: SNR sweep + Monte Carlo, approaches B/C/D/E (added eTypeII) |
| `CSIRS_128T128R_v3_0_MUMIMO.m` | System-level: NR Cell MU-MIMO, 16 UEs, event-driven simulator |
| `CSIRS_32T32R_SOCKET.m` | Skeleton/socket file cho 32T32R variant (reference only, incomplete) |
| `CSIRS_128T128R_v4_0_SOCKET.m` | **[ACTIVE]** 128T128R + eTypeII-r19 + subband + nrDRLScheduler + TCP socket |
| `CSIRS_128T128R_v4_0_BASELINE.m` | **[ACTIVE]** Identical env as v4_0_SOCKET nhưng dùng ProportionalFair (baseline so sánh DRL) |
| `nrDRLScheduler.m` | classdef extends nrScheduler — DRL scheduler với socket protocol |
| `test_dummy_server.py` | Python test server (Level 1–3): NOOP/random actions, obs_dump, validation |

**Module functions (v2):** `csirs_feedback.m`, `csirs_computeMetrics.m`, `csirs_buildGrid.m`, `csirs_channelEstimate.m`, `csirs_channelSetup.m`, `csirs_txRx.m`, `csirs_printSweepTable.m`

**Helper functions (v3):** `hNRCreateCDLChannels.m`, `hNRCustomChannelModel.m`, `hArrayGeometry.m`, `helperNRSchedulingLogger.m`, `helperNRPhyLogger.m`, `helperNRMetricsVisualizer.m`

---

## Cấu hình hệ thống

| Tham số | v1/v2 (Link-level) | v3 (System-level) |
|---|---|---|
| Carrier | 52 PRBs, SCS 30kHz, n78 3.5GHz | 24 PRBs, SCS 30kHz, n79 4.9GHz |
| gNB antenna | 128T (4V×16H×2pol) | 128T/128R |
| UE antenna | 4R (1V×2H×2pol) | 16 UEs × 4T/4R |
| CSI-RS | 4 NZP × 32-port (Row 18, CDM8) | CSI-RS (eTypeII-r19 feedback) |
| Channel | CDL-B/C/D, DS=100ns, Doppler=5Hz | CDL-D, DS=100ns, Doppler=5Hz |
| Duplex | N/A | TDD |

---

## 5 Approaches (v2: B/C/D/E; v1: A/B/C/D)

| | Approach | Mô tả | Spec |
|---|---|---|---|
| **A** | Rel-15/16 Type I Single Panel | Per-32-port, PanelDimensions=[8,2] | TS 38.214 §5.2.2.2.1 |
| **B** | SVD Upper Bound | Full 128-port, lý thuyết — chuẩn tham chiếu | — |
| **C** | Rel-19 Type I Mode A | CodebookMode=1, PanelDimensions=[1,16,4] | TS 38.214 §5.2.2.2.1a |
| **D** | Rel-19 Type I Mode B | CodebookMode=2, same config C | TS 38.214 §5.2.2.2.1a |
| **E** | Rel-19 Refined eTypeII 128-port | L=2 beams, PC=1, Panel=[1,16,4] | TS 38.214 §5.2.2.2.5a |

**Known limitation:** CDL-C cho RI=1 với ModeA TypeI Rel-19; CDL-B/D dùng cho test rank cao hơn.

---

## PMI/CQI Mode (v2 và v3)

- `pmiMode = 'Wideband' | 'Subband'`
- `cqiMode = 'Wideband' | 'Subband'`
- Subband size: 4 RBs (NRB∈[24,72], TS 38.214 Table 5.2.1.4-2)
- Approach D PMI luôn wideband (Mode B beam search constraint), nhưng CQI theo `cqiMode`

---

## v3 MU-MIMO System-level (CSIRS_128T128R_v3_0_MUMIMO.m)

**Architecture:** Event-driven `wirelessNetworkSimulator` (MATLAB Wireless Network Toolbox)

**Scheduler config:**
- `MUMIMOConfigDL`: MaxNumUsersPaired=4, MaxNumLayers=8, SemiOrthogonalityFactor=0.7
- DOF constraint: L=2 beams × 2pol = 4D subspace → K×r ≤ 4
- OLLA: InitialOffset=0, StepUp=0.27, StepDown=0.03 → TargetBLER≈10%
- CSI source: CSI-RS (eTypeII-r19), SRSPeriodicity=20ms, CSIReportPeriodicity=10 slots

**UE deployment:** 16 UEs, r∈[50,500]m, az∈[-60°,+60°], phân bố đồng đều trên diện tích (sqrt-trick)

**Output sections:**
- §8: Simulation Conditions Summary (in đẹp terminal)
- §9: Run simulation (50 frames)
- §10: KPI Summary (`displayPerformanceIndicators`)
- §11: Save traces → `simulationLogs.mat`
- §11b: Per-UE Grant/MCS Summary table
- §11c: Per-frame BLER & MCS trend (OLLA Convergence Monitor plot)
- §12: MU-MIMO Pairing Analysis — histogram UEs per RB

**OLLA behavior (State 3 — no reset):** MCSOffset tích lũy qua toàn bộ simulation qua HARQ ACK/NACK. BLER hội tụ về ~10% sau nhiều frames.

---

## Lịch sử phát triển (commit timeline)

| Giai đoạn | Commits | Nội dung |
|---|---|---|
| v1 baseline | `34a5248`→`30a9d8a` | 4 approaches A/B/C/D, output formatting |
| v2 Phase 1 | `78def83`→`0c97018` | Subband processing Phase 1 & 2 |
| v2 Phase 2 | `7873fca`→`3c89140` | Wideband/Subband CQI+PMI, Approach E eTypeII |
| v3 Wideband | `91ef93f`→`8662314` | NR Cell Simulation, wideband, `wirelessNetworkSimulator` |
| v3 Subband | `047d0fb`→`4959ec7` | Subband PMI trong system-level, PR #1 merged |
| v3 Fixes | `5ec44f7`→`a863e68` | OLLA no-reset, CDL-D, rename/clean project |
| v4 DRL Env | (current, untagged) | v4_0_SOCKET + nrDRLScheduler, Level 1–3 test passed, baseline created |

---

**How to apply:** Project có 2 lớp simulation riêng biệt — link-level (v1/v2, `CSIRS_128T128R_v*.m`) và system-level (v3, event-driven). Khi đề xuất thay đổi cần phân biệt rõ lớp nào. Approach E (eTypeII-r19) là codebook tiên tiến nhất, Approach B là upper bound tham chiếu. OLLA không reset (State 3) là thiết kế có chủ đích.
