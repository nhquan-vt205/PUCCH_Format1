# Session: Phân tích Algorithm Spec PUCCH Format 1 v1.4

- Date / Time: 2025-01-02 21:15
- Status: IN_PROGRESS
- Project / IP: PUCCH_Format1
- Module(s): Toàn bộ receiver chain (sequence gen, CORDIC, accumulator, MLE)
- Stage / Skill: 01-spec | ad-hoc (phân tích spec để hiểu requirement)
- Related Plan: N/A (chỉ đọc và phân tích)
- Plan Status: N/A
- Git: main / f33d93f

> Heading giữ tiếng Anh, nội dung tiếng Việt. Task này là đọc hiểu spec, không thực hiện implementation.

## User Request

"bắt đầu phân tích spec"

Context: User vừa mở file `spec/source/PUCCH_F1_Algorithm_Spec_PhaseAccumulator_MLE_v1.4_vi.md` trong IDE. Spec này là tài liệu thuật toán chi tiết cho gNB L1 Receiver xử lý PUCCH Format 1.

## Plan (read-only analysis)

1. Đọc toàn bộ spec (32 sections, ~1789 dòng)
2. Tóm tắt các phần chính:
   - Algorithm boundary và objective
   - 3GPP normative model vs hardware realization
   - Sequence generation (base sequence, cyclic shift, OCC)
   - Phase accumulator approach
   - CORDIC derotation
   - Coherent combining và MLE
3. Xác định các contract quan trọng (phase representation, CORDIC, accumulator, MLE)
4. Liệt kê các invariant và verification requirement
5. Highlight các điểm dễ nhầm lẫn hoặc critical

## TODO List

- [x] Đọc §0-2: Algorithm boundary, objective, 3GPP vs hardware
- [ ] Đọc §3-11: Modulation, sequence generation, OCC
- [ ] Đọc §12-17: Phase representation, accumulator, tương đương toán học
- [ ] Đọc §18-23: Received signal model, statistic Z, MLE
- [ ] Đọc §24-28: Hardware realization, CORDIC, accumulator contract
- [ ] Đọc §29-32: Pseudocode, verification, invariants, baseline results
- [ ] Tổng hợp phân tích

## Stage & Skill Used

- Skill: MANUAL (ad-hoc spec review)
- Why: Đây là bước đầu tiên để hiểu requirement trước khi bắt đầu bất kỳ design/implementation nào. Spec này là normative reference cho toàn bộ receiver chain.

## What Was Done

### Đọc §0-2: Phạm vi và phân biệt 3GPP vs Hardware

**§0. Algorithm boundary:**
- Input: $Y_{hat}$ (equalized samples sau channel estimation)
- Output: UCI decision (1-2 bits)
- Không bao gồm: channel estimation, equalization, DM-RS processing

**§1. Mục tiêu:**
- Khôi phục symbol $d(0)$ từ PUCCH Format 1 resource elements
- Thiết kế dùng **phase-domain reference generation** và **phase accumulator**
- Spec trình bày **3GPP transmitter model trước**, sau đó biến đổi sang phase accumulator

**Luồng tổng quát:**
```
3GPP UE transmitter model
    → sequence r_uv^(alpha,delta)(n)
    → PUCCH F1 modulation
    → block-wise spreading w_i(m)
    → z(...)
    → received Y_hat
    → conjugate reference + conjugate OCC
    → coherent statistic Z
    → MLE
    → UCI bits
```

**§2. Phân biệt 3GPP vs Hardware:**

**3GPP-defined (normative):**
- Max 2 bits: 1 bit → BPSK, 2 bits → QPSK
- Symbol d(0) × low-PAPR sequence
- Block-wise spread với w_i(m)
- Sequence group u, sequence number v theo §6.3.2.2.1
- Cyclic shift alpha theo §6.3.2.2.2
- OCC w_i(m) từ Table 6.3.2.4.1-2
- DM-RS RE phải bị loại ra

**Hardware-defined (implementation):**
- I/Q: signed Q1.15 (16-bit)
- Phase: 16-bit modulo 2^16
- CORDIC: rotation mode, 16 iterations
- CORDIC output: 18-bit I/Q
- Accumulator: 26-bit I/Q
- Rx antenna: 1

**Observation quan trọng:**
- Spec này rất rõ ràng về việc **không thay đổi toán học 3GPP**, chỉ thay đổi cách biểu diễn số
- Phase accumulator là **optimization**, không phải thay đổi algorithm
- Verification phải chứng minh tương đương: `Z_direct = Z_phase_accumulator` (trong sai số lượng tử)

## Design Decisions

Chưa có (đây là session read-only).

## Interface / Register Map Changes

Chưa có (đây là session read-only).

## Traceability

Chưa áp dụng (chưa có REQ formal từ spec này).

## Files Touched

- `spec/source/PUCCH_F1_Algorithm_Spec_PhaseAccumulator_MLE_v1.4_vi.md` — đọc (1789 dòng)

## Tool Results (MANUAL)

| Tool | Status | Report / Log | Ghi chú |
|------|--------|-------------|---------|
| Lint | NOT_RUN | N/A | Chưa có RTL |
| Sim | NOT_RUN | N/A | Chưa có RTL/TB |
| Synth | NOT_RUN | N/A | Chưa có RTL |

## Open Issues / Debug

Chưa có.

## FE Flow Impact

| Stage | Impact | Lý do |
|-------|--------|-------|
| RTL | MAYBE | Spec này định nghĩa algorithm — sẽ dẫn tới RTL implementation sau |
| TB | MAYBE | Cần testbench verify tương đương 3GPP ↔ phase accumulator |
| Sim/Lint/Synth/Timing | MAYBE | Tùy implementation |
| CDC | NO | Single clock domain (receiver chain) |
| Doc | YES | Spec này chính là doc normative |

## Gaps / Known Limitations

**Đã đọc (§0-2):**
- Algorithm boundary rõ ràng: bắt đầu từ Y_hat, kết thúc ở UCI bits
- Hardware parameter baseline: 16-bit phase, 18-bit CORDIC out, 26-bit accumulator

**Chưa đọc (§3-32):**
- Chi tiết modulation BPSK/QPSK (§3)
- Low-PAPR sequence generation: base sequence φ_u(n), cyclic shift alpha, sequence group u/v (§4-8)
- OCC w_i(m) và DM-RS pattern (§9-11)
- Phase representation và accumulator (§12-16)
- Received signal model và statistic Z (§18-19)
- MLE cho 1-bit và 2-bit (§22-23)
- CORDIC contract, accumulator contract (§26-27)
- Debug signals, pseudocode, verification (§28-32)

## Key Decisions

**Từ §0-2:**
1. **Phase accumulator là core optimization** — spec rất nhấn mạnh việc trình bày 3GPP model trước, rồi mới biến đổi sang phase accumulator.
2. **Verification strategy:** phải chứng minh `Z_direct = Z_phase_accumulator` trong sai số lượng tử đã định nghĩa.
3. **Hardware baseline đã khóa:** 16-bit phase, CORDIC 16 iterations, 26-bit accumulator.

## Blockers / Notes for Next Agent

**Cần đọc tiếp:**
- §3-11: Sequence generation (base, cyclic shift, OCC) — đây là core của transmitter model 3GPP
- §12-17: Phase representation và accumulator — đây là core của hardware optimization
- §18-23: Receiver processing và MLE — đây là core của detection algorithm
- §24-28: Hardware contract (CORDIC, accumulator) — đây là contract cho RTL implementation
- §29-32: Pseudocode, verification, invariants — đây là checklist cho verification

**Quan sát ban đầu:**
- Spec này rất chi tiết (32 sections, 1789 dòng), viết theo style "normative + implementation note"
- Có nhiều bảng tra cứu từ 3GPP (Table 5.2.2.2-2 cho φ_u(n), Table 6.3.2.4.1-1 cho N_SF, Table 6.3.2.4.1-2 cho OCC)
- Có nhiều công thức LaTeX — cần verify rendering trong Obsidian
- Spec nhấn mạnh **bit ordering convention** (§3.3) và **BPSK không nằm trên trục thực** (§3.1) — đây là nguồn lỗi phổ biến

**Câu hỏi cần clarify với user (sau khi đọc hết):**
- Mục tiêu cuối cùng: chỉ phân tích spec, hay cần tạo requirement formal, hay cần bắt đầu design?
- Có golden model Python nào đã có sẵn không? (Spec nhắc tới "golden model" nhiều lần)
- Tool chain: Verilator? iverilog? Vivado? (để biết flow nào cần plan)

## Read These First Next Time

- `spec/source/PUCCH_F1_Algorithm_Spec_PhaseAccumulator_MLE_v1.4_vi.md` — **đọc tiếp từ §3**
- `spec/source/PUCCH_F1_Receiver_Design_Combined_v1.3_vi.md` — có thể là design doc bổ sung
- `PUCCH_F1_Receiver_Design_Roadmap_v1.3_vi.md` — roadmap tổng quan

---

## Progress: §0-2 done, §3-32 in progress
