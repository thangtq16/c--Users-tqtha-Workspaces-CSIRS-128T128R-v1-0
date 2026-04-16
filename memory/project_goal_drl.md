---
name: Project Ultimate Goal — DRL-based MU-MIMO Scheduler
description: Mục tiêu đích cuối của project: xây môi trường mô phỏng NR Cell chuẩn Rel-19 128T128R làm gym cho DRL agent thay thế scheduler truyền thống
type: project
originSessionId: 45e131e7-bcbe-4b79-8f02-c91c8dbc1be7
---
## Mục tiêu cuối cùng

Xây dựng một **môi trường mô phỏng MATLAB NR Cell chuẩn 3GPP Release 19, 128T128R** đủ chính xác để:
1. Làm "gym environment" cho DRL agent học policy scheduling MU-MIMO downlink
2. So sánh DRL scheduler vs. conventional MU-MIMO scheduler (baseline từ v3_0)

**Why:** Đây là bài toán nghiên cứu — không phải chỉ so sánh codebook. Codebook (v1/v2) là nền tảng để hiểu sâu PHY layer; system-level (v3/v4) là môi trường thực sự để train DRL.

## Kiến trúc cuối (v4_0_SOCKET)

```
Python DRL Agent  ←── TCP socket :6666 ──→  MATLAB NR Cell (v4_0)
  học scheduling policy                        wirelessNetworkSimulator
  PPO/SAC/...                                  nrDRLScheduler (classdef)
       ↑                                              ↑
  action: [NumLayers × NumRBGs]           obs: TTI_START JSON / TTI_DONE
  reward: PF throughput                   eTypeII-r19, subband CQI
```

## Yêu cầu "chuẩn Rel-19" của môi trường

- **Codebook**: eTypeII-r19 (TS 38.214 §5.2.2.2.5a), L=2 beams, Panel [1×16×4]
- **PMI/CQI**: Subband mode — `gNB.PMICQIMode = "Subband"` BẮT BUỘC trước `connectUE`
- **Antenna**: 128T128R (4V×16H×2pol), UE 4Rx (cho rank > 1)
- **Channel**: CDL-D, DS=100ns, Doppler=5Hz

## Observation Space (8 features, gửi qua TTI_START)

| # | Key JSON | Mô tả | Trạng thái |
|---|---|---|---|
| 1 | `avg_tp` | EMA throughput thực (`ActualTputEMA`, từ `MAC.ReceivedBytes`) | ✅ VERIFIED |
| 2 | `ue_rank` | RI từng UE | ✅ VERIFIED |
| 3 | `occupied_rbgs` | RBG occupied (=0 khi RVSequence=0, no retransmit) | ✅ VERIFIED |
| 4 | `buf` | Buffer DL (bytes) | ✅ VERIFIED |
| 5 | `curr_mcs` | MCS sau OLLA offset | ✅ VERIFIED |
| 6 | `ue_i1` | eTypeII beam-group index i1[3 elements: i11,i12,i1_lc] | ✅ VERIFIED |
| 7 | `sub_cqi` | Subband CQI vector [numRBGs] per UE (resample 6→12 RBGs) | ✅ VERIFIED (flat vì CDL-D high SNR, tune TxPower để có variation) |
| 8 | `max_cross_corr` | Max-column kappa κ [MaxUEs×MaxUEs×numRBGs] | ✅ VERIFIED |

**RNTI convention**: `eligible_ues` gửi **0-based** (MATLAB RNTI−1). Python arrays indexed 0..MaxUEs−1.
**avg_tp**: gửi `obj.ActualTputEMA` (EMA), KHÔNG phải `inst_tp_all` (instantaneous).
**sub_cqi variation**: TxPower=40dBm vẫn flat CQI=15 với CDL-D. Cần ~20dBm hoặc CDL-C/B để có variation.

**Công thức max_cross_corr**: κ(W_i, W_j) = max over columns of |W_i^H W_j| / (‖w_i‖ ‖w_j‖), tính per-RBG. Khác hoàn toàn với `SemiOrthogonalityFactor` check của MATLAB (dùng flattened dot product).

## Lý do dùng FTP Model 3 thay vì FullBuffer

FullBuffer → buffer luôn ≠ 0 → feature `buf` (obs #4) không có biến động → DRL không học được priority. FTP3 (6 Mbps, 1500B pkts) → buffer có lúc trống → reward differentiates scheduling decisions.

## Cẩn thận khi sửa code

- `gNB.PMICQIMode` phải set TRƯỚC `connectUE` (nrNodeValidation đọc khi connect)
- `SchedulerStrategy=1` (numeric) gây crash — dùng string `'ProportionalFair'` / `'RoundRobin'` / `'BestCQI'` trong `configureScheduler`
- OLLA không reset (State 3): MCSOffset tích lũy toàn simulation — intentional design
- `addpath(wirelessnetwork + 5g, '-begin')` bắt buộc để enable eTypeII-r19 + 128T
- `csirsConfig` là array → dùng `csirsConfig(1).CSIRSPeriod` (không phải `csirsConfig.CSIRSPeriod`)
- `avg_tp` trong TTI_START phải gửi `obj.ActualTputEMA`, KHÔNG phải `inst_tp_all`
- `eligible_ues` phải gửi `eligibleUEs - 1` (0-based) để khớp với Python array indexing

## Testing Protocol (đã hoàn thành)

| Level | Mô tả | Kết quả |
|---|---|---|
| 0 | v3_0 baseline run (5 frames) | ✅ PASS — i1=[4,2,1431] confirmed eTypeII-r19 |
| 1 | Socket handshake + TTI protocol | ✅ PASS — 21 TTI_START/DONE, STOP received |
| 2 | Obs value inspection (JSON dump) | ✅ PASS — tất cả 8 features có giá trị hợp lệ |
| 3 | Random action smoke test | ✅ PASS — cell_tput>0, avg_tp EMA tích lũy đúng |

## Baseline so sánh DRL

`CSIRS_128T128R_v4_0_BASELINE.m` — cùng env, dùng `ProportionalFair` scheduler (50 frames).
Đổi scheduler: thay `schedulerType = 'RoundRobin'` hoặc `'BestCQI'` ở đầu file.

**How to apply:** Khi đề xuất fix/thay đổi, ưu tiên đúng semantic PHY (PMI/CQI spec) trước rồi mới nghĩ đến socket protocol. Mọi thay đổi observation space phải đồng bộ cả MATLAB (TTI_START payload) và Python adapter.
