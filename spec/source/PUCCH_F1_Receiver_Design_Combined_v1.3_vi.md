# PUCCH Format 1 — Tài liệu thiết kế gNB Receiver (Tổng hợp)

**Phiên bản:** 1.4  
**Mục tiêu:** gNB L1 Receiver xử lý PUCCH Format 1  
**Tiêu chuẩn:** 3GPP TS 38.104 / 38.212 / 38.211 / 38.213, V15.10.0, Release 15  
**Hệ thống:** FR1, 100 MHz, 30 kHz SCS, Normal CP, 122.88 MHz implementation reference  

**Tài liệu này tổng hợp:**
- **PHẦN I — Algorithm Specification:** Xem file `PUCCH_F1_Algorithm_Spec_PhaseAccumulator_MLE_v1.4_vi.md` cho chi tiết thuật toán đầy đủ.
- **PHẦN II — Design Roadmap:** Lộ trình thiết kế từ specification đến RTL và verification.

> **Lưu ý v1.4:** Phần I (Algorithm) của file này tham chiếu sang Algorithm Spec v1.5 riêng. File này tập trung vào Design Roadmap với các bảng tra, interface, port definitions đã được đồng bộ và sửa format.

---

# MỤC LỤC

## PHẦN I — ALGORITHM SUMMARY (tham chiếu)
- Xem chi tiết tại `PUCCH_F1_Algorithm_Spec_PhaseAccumulator_MLE_v1.4_vi.md`

## PHẦN II — DESIGN ROADMAP
- [R1. Mục đích và phạm vi](#r1-mục-đích-và-phạm-vi)
- [R2. Phân loại tham số](#r2-phân-loại-tham-số-fixed--configurable--implementation-defined)
- [R3. Chuẩn tham chiếu](#r3-chuẩn-tham-chiếu-và-phạm-vi-từng-tài-liệu)
- [R4. Bước 1 — Freeze system timing](#r4-bước-1--freeze-system-timing)
- [R5. Bước 2 — Freeze receiver boundary](#r5-bước-2--freeze-receiver-boundary)
- [R6. Bước 3 — Configuration model](#r6-bước-3--thiết-kế-configuration-model)
- [R7. Bước 4 — Sequence engine](#r7-bước-4--thiết-kế-sequence-engine)
- [R8. Bước 5 — Resource extractor](#r8-bước-5--thiết-kế-resource-extractor)
- [R9. Bước 6 — OCC engine](#r9-bước-6--thiết-kế-occ-engine)
- [R10. Bước 7 — Derotation](#r10-bước-7--thiết-kế-derotation)
- [R11. Bước 8 — Coherent combining](#r11-bước-8--coherent-combining)
- [R12. Bước 9 — MLE detector](#r12-bước-9--mle-detector)
- [R13. Bước 10 — Numerical representation](#r13-bước-10--freeze-numerical-representation)
- [R14. Bước 11 — Pipeline](#r14-bước-11--freeze-pipeline)
- [R15. Bước 12 — Top-level interface](#r15-bước-12--top-level-interface)
- [R16. Bước 13 — Thứ tự viết RTL](#r16-bước-13--thứ-tự-viết-rtl)
- [R17. Verification plan](#r17-verification-plan)
- [R18. Error handling, reset, RTL scope](#r18-error-handling-reset-và-rtl-scope)
- [R19. Definition of Done](#r19-definition-of-done)
- [R20. Điều không được hard-code nhầm](#r20-các-điều-không-được-hard-code-nhầm)
- [R21. Minor consistency pass](#r21-minor-consistency-pass)

## PHẦN III — CROSS-REFERENCE & CÂU HỎI MỞ
- [CR. Cross-reference hai tài liệu](#cr-cross-reference-hai-tài-liệu)
- [Q. Câu hỏi mở cần verify](#q-câu-hỏi-mở-cần-verify-trước-khi-freeze)

---
---

# PHẦN II — DESIGN ROADMAP

## Lộ trình thiết kế gNB Receiver — PUCCH Format 1

**Tài liệu:** Design Roadmap / Implementation Plan
**Phiên bản:** 1.4

---

## R1. Mục đích và phạm vi

Tài liệu này quy định **thứ tự thiết kế** receiver từ specification đến RTL và verification. Tài liệu không khóa mọi tham số PUCCH Format 1 thành một giá trị duy nhất. Chỉ những đặc tính bản chất của Format 1 mới được cố định; các tham số có thể tạo ra nhánh xử lý theo 3GPP được giữ dưới dạng configuration.

Chuỗi thiết kế:

```
3GPP specification
      ↓
Physical/resource interpretation
      ↓
Receiver algorithm
      ↓
Configuration model
      ↓
Timing model
      ↓
Resource extraction
      ↓
Sequence/phase generation
      ↓
DM-RS / channel estimation interface
      ↓
Derotation
      ↓
OCC despreading
      ↓
Coherent accumulation
      ↓
MLE decision
      ↓
Fixed-point model
      ↓
Pipeline architecture
      ↓
RTL interface
      ↓
Verification
      ↓
Synthesis
```

---

## R2. Phân loại tham số: Fixed / Configurable / Implementation-defined

Đây là nguyên tắc quan trọng nhất của tài liệu.

### R2.1. Tham số cố định vì đang thiết kế Format 1

| **Tham số**             | **Giá trị**           | **Lý do**                       |
| ----------------------- | --------------------- | ------------------------------- |
| PUCCH format            | **Format 1**          | Phạm vi của thiết kế            |
| Số PRB của PUCCH        | **1 PRB**             | Format 1 sử dụng một PRB       |
| Số UCI bit tối đa       | **2 bit**             | Đặc tính của Format 1          |
| Kiểu modulation         | **BPSK / QPSK**       | 1 bit → BPSK, 2 bit → QPSK    |
| Sequence family         | **Low-PAPR sequence** | Theo TS 38.211                 |
| Block-wise spreading    | **Có**                | Đặc tính của Format 1          |
| OCC                     | **Có**                | Format 1 sử dụng `w_i(m)`     |
| Resource antenna port   | **p = 2000**          | Theo mapping của PUCCH Format 1|
| PRB width               | **12 subcarriers**    | Một PRB luôn có 12 subcarriers |

### R2.2. Tham số phải giữ configurable

| **Tham số**                      | **Trạng thái**                                  | **Ý nghĩa**                                                     |
| -------------------------------- | ----------------------------------------------- | ---------------------------------------------------------------- |
| `pucch_nsymb`                    | **Configurable: 4…14**                          | Format 1 cho phép nhiều độ dài truyền                            |
| `pucch_start_symbol`             | **Configurable: 0…10**                          | Vị trí bắt đầu PUCCH trong slot                                 |
| `intra_slot_hopping`             | **Configurable: 0/1**                           | Có thể bật/tắt theo higher-layer configuration                   |
| `starting_prb`                   | **Configurable: 0…272**                         | PRB của hop thứ nhất                                             |
| `second_hop_prb`                 | **Configurable: 0…272**                         | PRB của hop thứ hai                                              |
| `group_hopping_mode`             | **Configurable: 2'b00/01/10**                   | neither=00, enable=01, disable=10                                |
| `hopping_id` / `n_ID`            | **Configurable: 0…1023**                        | Identity dùng cho sequence/hopping                               |
| `initial_cyclic_shift` (`m_0`)   | **Configurable: 0…11**                          | Tham số PUCCH resource                                           |
| `time_domain_occ` (`i`)          | **Configurable: 0…6**                           | Chọn OCC trong bảng 6.3.2.4.1-2                                 |
| `uci_bit_count`                  | **Configurable: 1 hoặc 2**                      | Quyết định BPSK/QPSK                                            |

### R2.3. Tham số baseline để làm RTL revision đầu tiên

Các giá trị dưới đây **không phải yêu cầu của 3GPP**; chúng chỉ là một operating point giúp giảm độ phức tạp khi viết RTL đầu tiên:

| **Tham số**              | **Baseline revision A** | **Ghi chú**                                                           |
| ------------------------ | ----------------------- | --------------------------------------------------------------------- |
| `pucch_nsymb`            | 14                      | Dùng để freeze datapath đầu tiên; architecture vẫn phải nhận 4…14    |
| `intra_slot_hopping`     | 0                       | Tắt trong RTL revision A, nhưng port/config vẫn tồn tại              |
| `group_hopping_mode`     | `neither` (2'b00)       | Revision A; không loại bỏ logic/config interface cho hai mode còn lại |
| Rx antenna               | 1                       | Baseline compute engine                                               |
| UE contexts đồng thời    | 1                       | Baseline scheduler/data-path                                          |
| Input sample             | channel-compensated `Y_hat` | DM-RS/channel estimation nằm ngoài algorithm core                 |

**Kết luận:** `intra_slot_hopping` và `group_hopping_mode` **không được ghi vào mục "fixed because Format 1"**. Chúng là configuration của Format 1 và chỉ được chọn giá trị baseline cho revision RTL đầu tiên.

---

## R3. Chuẩn tham chiếu và phạm vi từng tài liệu

| **Tài liệu** | **Phiên bản** | **Vai trò**                                                        |
| ------------- | ------------- | ------------------------------------------------------------------ |
| TS 38.104     | V15.10.0      | Channel bandwidth, FR1, transmission bandwidth configuration       |
| TS 38.212     | V15.10.0      | UCI processing context / physical-layer bit procedures              |
| TS 38.211     | V15.10.0      | Sequence generation, PUCCH Format 1 modulation và mapping          |
| TS 38.213     | V15.10.0      | PUCCH resource/configuration và các tham số higher-layer liên quan |

Các clause trọng tâm của receiver:

- TS 38.211 §5.1: modulation mapping (BPSK §5.1.2, QPSK §5.1.3).
- TS 38.211 §5.2.1: Gold sequence.
- TS 38.211 §5.2.2: low-PAPR sequence (Table 5.2.2.2-2: φ_u(n) cho M_ZC=12).
- TS 38.211 §6.3.2.2: sequence và cyclic-shift hopping.
- TS 38.211 §6.3.2.4.1: PUCCH Format 1 sequence modulation (Table 6.3.2.4.1-1, 6.3.2.4.1-2).
- TS 38.211 §6.3.2.4.2: mapping lên physical resources.
- TS 38.211 §6.4.1.3.1: PUCCH Format 1 DM-RS (Table 6.4.1.3.1.1-1).
- TS 38.213 §9.2.1: PUCCH resource/configuration context.

---

## R4. Bước 1 — Freeze system timing

### R4.1. Numerology

Với `SCS = 30 kHz`:

$$
\Delta f=30\;\text{kHz}=2^1\times15\;\text{kHz}
$$

Chọn FFT reference:

$$
N_{\text{FFT}}=4096
$$

Reference sampling clock:

$$
F_s=N_{\text{FFT}}\cdot\Delta f=4096\times30\;\text{kHz}=122.88\;\text{MHz}
$$

### R4.2. Channel bandwidth

Thiết kế hệ thống là FR1, `100 MHz`, `30 kHz SCS`. Theo bảng transmission bandwidth configuration của TS 38.104, số PRB là:

$$
N_{\text{RB}}=273
$$

Do đó số active subcarriers theo transmission bandwidth configuration là:

$$
N_{\text{sc}}=273\times12=3276
$$

`4096` là FFT reference của hệ thống; không được nhầm `4096` với số active subcarriers của carrier.

### R4.3. Timing slot (pattern μ=1: 1 CP dài + 13 CP ngắn)

Với `μ = 1`, `N_FFT = 4096`, `Normal CP`, theo TS 38.211 §5.3.1:

- Số symbol trong một slot: `N_{symb}^{slot} = 14`.
- CP dài xuất hiện tại symbol `l = 0` (và `l = 14` nếu có symbol thứ 15, nhưng slot chỉ 0..13).
- Công thức CP (đơn vị `T_c`):
  - `N_{CP,l}^μ = 144·κ·2^(-μ) + 16·κ` cho `l = 0` hoặc `l = 7·2^μ`.
  - `N_{CP,l}^μ = 144·κ·2^(-μ)` cho các symbol còn lại.
- Với `κ = 64`, `μ = 1`: `144·64·2^(-1) = 4608`, `16·64 = 1024`.
- Đổi sang sample tại `Fs = 122.88 MHz` (1 sample = 1/122.88 μs):
  - CP ngắn = `4608 T_c` = **288 samples**.
  - CP dài = `(4608 + 1024) T_c` = **352 samples**.

Do đó timing model đúng cho μ=1/N\_FFT=4096:

| **Symbol** | **Vai trò** | **Useful** | **CP**  | **Tổng clock** |
| ---------- | ----------- | ---------- | ------- | -------------- |
| 0          | **CP dài**  | 4096       | **352** | **4448**       |
| 1…13       | CP ngắn     | 4096       | 288     | 4384           |

Kiểm tra:

$$
4448+13\times4384=4448+56992=61440
$$

Đúng bằng `Fs × T_slot = 122.88 MHz × 0.5 ms = 61440`.

**Lưu ý quan trọng:**

- Chỉ **1 CP dài** trong mỗi slot ở μ=1, tại symbol 0.
- Pattern `2 CP dài mỗi slot` (320/288) là đặc trưng của **μ=0** với `N_FFT=4096`, **không** áp dụng cho μ=1.
- Với `μ=1`, số CP dài là 1, không phải 2.
- Khi hệ thống có nhiều numerology, timing generator phải cấu hình được pattern CP theo μ.

Timing model trên là **implementation timing invariant**, không phải một field PUCCH.

### R4.4. Time-counter outputs

```systemverilog
output logic [9:0]  frame_idx;      // SFN, 0..1023
output logic [4:0]  slot_idx;       // slot trong frame, 0..19 (μ=1)
output logic [3:0]  symbol_idx;     // symbol trong slot, 0..13
output logic [12:0] clk_in_symbol;  // clock counter trong symbol, 0..4447/4383
output logic        slot_start;     // pulse đầu slot
output logic        symbol_start;   // pulse đầu symbol
output logic        symbol_end;     // pulse cuối symbol
```

Time counter chỉ tạo reference time. Nó không tự quyết định PUCCH có tồn tại hay không.

### R4.5. Mapping hardware counter ↔ 3GPP index

Để tránh nhầm lẫn giữa counter nội bộ và index 3GPP, khóa mapping như sau:

| **3GPP index**          | **Hardware counter**                     | **Range (μ=1)** | **Ghi chú**            |
| ----------------------- | ---------------------------------------- | --------------- | ---------------------- |
| `frame_idx` (SFN)       | `frame_idx[9:0]`                         | 0..1023         | SFN 10-bit             |
| `slot_idx` (trong frame)| `slot_idx[4:0]`                          | 0..19           | 20 slot/frame với μ=1  |
| `symbol_idx` (trong slot)| `symbol_idx[3:0]`                       | 0..13           | 14 symbol/slot         |
| `n_{s,f}^μ`             | `frame_idx × 20 + slot_idx`              | 0..20479        | Dùng trong `n_cs`      |
| `l`                     | `symbol_idx`                             | 0..13           | Dùng trong `n_cs`      |
| `N_{symb}^{slot}`       | hằng số = 14                             | —               | Với μ=1, Normal CP     |
| `m` (spreading)         | counter riêng, reset mỗi hop             | 0..N\_SF-1      | Không phải slot/symbol |
| `m'` (hop)              | `intra_slot_hopping ? hop_idx : 0`       | 0 hoặc 0,1      | —                      |

**Quan trọng:** `slot_idx` là slot trong **frame** (0..19 với μ=1), không phải slot trong subframe. `n_{s,f}^μ` trong công thức `n_cs` phải dùng giá trị `0..20479` (hoặc modulo theo frame nếu chu kỳ Gold sequence ngắn hơn).

---

## R5. Bước 2 — Freeze receiver boundary

Boundary của algorithm core:

```
FFT / resource grid
        ↓
PUCCH RE extraction
        ↓
DM-RS + channel estimation + equalization
        ↓
Y_hat(m,n)
        ↓
[Algorithm core của Algorithm Specification]
```

Input chính:

$$
\hat{Y}(m,n)=\hat{Y}_I(m,n)+j\hat{Y}_Q(m,n)
$$

Trong đó:

- `m`: chỉ số spreading symbol của Format 1, `m = 0…N_{SF}-1`.
- `n`: subcarrier index trong PRB, `n = 0…11`.
- `m'`: hop index, `m' = 0` (no hopping) hoặc `m' ∈ {0,1}` (hopping).

DM-RS không bị coi là UCI data. Resource extractor phải loại các RE được DM-RS sử dụng trước khi cấp `Y_hat` cho UCI detector.

---

## R6. Bước 3 — Thiết kế configuration model

Configuration phải phản ánh các nhánh thực sự của Format 1.

```systemverilog
input  logic        pucch_enable;          // 1-bit: bật PUCCH occasion
input  logic [3:0]  pucch_start_symbol;    // 4-bit: symbol bắt đầu, 0..10
input  logic [3:0]  pucch_nsymb;           // 4-bit: số symbol, 4..14
input  logic [8:0]  starting_prb;          // 9-bit: PRB index, 0..272
input  logic [8:0]  second_hop_prb;        // 9-bit: PRB index hop 2, 0..272
input  logic        intra_slot_hopping;    // 1-bit: 0=off, 1=on
input  logic [1:0]  group_hopping_mode;    // 2-bit: 00=neither,01=enable,10=disable
input  logic [9:0]  hopping_id;            // 10-bit: n_ID, 0..1023
input  logic [3:0]  initial_cyclic_shift;  // 4-bit: m_0, 0..11
input  logic [2:0]  time_domain_occ;       // 3-bit: OCC index i, 0..6
input  logic        uci_bit_count;         // 1-bit: 0=1bit(BPSK), 1=2bit(QPSK)
```

### R6.1. Nguyên tắc

`pucch_nsymb` không được hard-code thành 14 ở interface.

`intra_slot_hopping` không được coi là constant của Format 1.

`group_hopping_mode` không được coi là constant của Format 1.

Revision A có thể reset các field trên về một giá trị baseline nhưng controller phải đọc chúng từ configuration register.

### R6.2. Range và nguồn gốc của `time_domain_occ` (`i`)

- `i` là chỉ số OCC, chọn từ Table 6.3.2.4.1-2.
- Range: `0 ≤ i ≤ N_SF - 1`. Với `N_SF = 7` (baseline), `i ∈ {0,…,6}`.
- Nguồn gốc: higher-layer configuration (`PUCCH-Resource` → `timeDomainOCC` theo TS 38.213).
- Trong baseline RTL revision A, `i` là configurable register; mặc định `i = 0` cho test noise-free.

### R6.3. Interlace out-of-scope note

Thiết kế này **không nhắm** tới interlace mapping. Với FR1/30 kHz SCS PUCCH Format 1, `m_int = 0` là normative (xem Algorithm §7.0). Config field `interlace_related` chỉ giữ như port nếu system architecture mở rộng sang các kênh khác.

---

## R7. Bước 4 — Thiết kế sequence engine

Sequence engine gồm hai lớp:

```
3GPP sequence equations
        ↓
phase-domain realization
        ↓
phase accumulator
```

Các kết quả phải tạo ra:

```
c(n)                    // Gold sequence bits
f_gh                    // group hopping function (5-bit, 0..29)
f_ss                    // sequence shift (5-bit, 0..29)
u                       // sequence group (5-bit, 0..29)
v                       // sequence number (1-bit, 0 or 1)
n_cs                    // cyclic shift term (8-bit, 0..255)
alpha                   // cyclic shift angle (16-bit phase code)
phi_u[0:11]             // base sequence values (3-bit signed each)
theta_ref_code[0:11]    // reference phase codes (16-bit each)
```

### R7.1. Interface sequence engine

```systemverilog
// Inputs
input  logic [9:0]  frame_idx;
input  logic [4:0]  slot_idx;
input  logic [3:0]  symbol_idx;
input  logic [9:0]  hopping_id;        // n_ID
input  logic [1:0]  group_hopping_mode;
input  logic [3:0]  initial_cyclic_shift; // m_0
input  logic        intra_slot_hopping;
input  logic [3:0]  pucch_start_symbol; // l'

// Outputs
output logic [4:0]  seq_group_u;        // 0..29
output logic        seq_number_v;       // 0 or 1
output logic [15:0] alpha_phase_code;   // phase code of alpha
output logic [15:0] theta_ref_code;     // per-subcarrier, streamed n=0..11
output logic        theta_ref_valid;
```

Đối với one-PRB PUCCH:

$$
N_{\mathrm{sc}}^{\mathrm{RB}}=12
$$

Do đó TS 38.211 §5.2.2.2 sử dụng bảng `phi_u(n)` cho độ dài nhỏ hơn 36.

**Lưu ý:** phase base của length-12 phải dùng đúng:

$$
r_{u,v}(n)=e^{j\frac{\pi}{4}\phi_u(n)}
$$

không được thay bằng `π/2`. Xem Algorithm Spec §5 cho bảng `φ_u(n)` đầy đủ (Table 5.2.2.2-2).

---

## R8. Bước 5 — Thiết kế resource extractor

### R8.1. Interface

```systemverilog
// Inputs (from timing + config)
input  logic [9:0]  frame_idx;
input  logic [4:0]  slot_idx;
input  logic [3:0]  symbol_idx;
input  logic [3:0]  pucch_start_symbol; // 0..10
input  logic [3:0]  pucch_nsymb;        // 4..14
input  logic [8:0]  starting_prb;       // 0..272
input  logic [8:0]  second_hop_prb;     // 0..272
input  logic        intra_slot_hopping;

// Outputs (to algorithm core)
output logic signed [15:0] y_hat_i;     // Q1.15
output logic signed [15:0] y_hat_q;     // Q1.15
output logic [2:0]  sample_m;           // spreading symbol index, 0..6
output logic [3:0]  sample_n;           // subcarrier index, 0..11
output logic        sample_valid;       // 1 if data RE
output logic        sample_first;       // 1 if first RE of PUCCH occasion
output logic        sample_last;        // 1 if last RE of PUCCH occasion
output logic        hop_idx;            // 0 or 1
output logic [3:0]  symbol_in_slot;     // absolute symbol index in slot
```

### R8.2. Grid interface

```systemverilog
output logic        grid_rd_en;
output logic [14:0] grid_rd_addr;       // 15-bit: slot×14×273×12 worst case
input  logic [31:0] grid_rd_data;       // {I[15:0], Q[15:0]}
input  logic        grid_rd_valid;
```

Conceptual grid address nếu memory tổ chức theo:

```
slot → symbol → PRB → subcarrier
```

thì:

$$
\text{addr}=\left(\left(\text{symbol}\times273\right)+\text{PRB}\right)\times12+\text{sc}
$$

Đây chỉ là **reference address model**. Layout thực tế có thể đổi để phù hợp BRAM/SRAM.

### R8.3. DM-RS skip semantics và `sample_last`

Resource extractor phải phân biệt 3 loại RE:

| **Loại RE**       | **Đưa vào UCI detector?** | **Ghi chú**                 |
| ----------------- | ------------------------- | --------------------------- |
| UCI data RE       | Có                        | `sample_valid = 1`          |
| DM-RS RE          | Không                     | Bị skip, `sample_valid = 0` |
| Guard / empty RE  | Không                     | Bị skip                     |

**`sample_last` semantics:**

- `sample_last = 1` khi RE hiện tại là **RE cuối cùng của toàn bộ PUCCH occasion** (bao gồm cả hop, nếu có).
- Không dùng `sample_last` cho "cuối hop" hay "cuối symbol"; nếu cần, thêm cờ riêng `hop_last`, `symbol_last`.
- Trong baseline `N_symb=14`, no hopping: `sample_last = 1` tại RE thứ 84 (data RE cuối cùng).

**Lưu ý:** DM-RS pattern phải được đọc từ TS 38.211 Table 6.4.1.3.1.1-1 (xem Algorithm Spec §11.5). DM-RS symbols nằm ở vị trí chẵn (`l = 0, 2, 4, ...`), data ở vị trí lẻ (`l = 1, 3, 5, ...`).

---

## R9. Bước 6 — Thiết kế OCC engine

TS 38.211 quy định OCC `w_i(m)` theo bảng 6.3.2.4.1-2 (xem Algorithm Spec §11.2 cho bảng φ đầy đủ).

### R9.1. Interface

```systemverilog
// Inputs
input  logic [2:0]  occ_index_i;        // 0..6
input  logic [2:0]  spreading_m;        // 0..6
input  logic [2:0]  n_sf;               // N_SF value, 1..7

// Outputs
output logic [15:0] occ_phase_code;     // phase code of w_i*(m)
output logic        occ_valid;
```

OCC phải phụ thuộc vào:

```
pucch_nsymb
intra_slot_hopping
m'
i
m
```

Do đó không được cố định `N_SF = 7` ở kiến trúc tổng quát.

### R9.2. Bảng `N_SF` cho `N_symb = 4..14` (Table 6.3.2.4.1-1)

Bảng dưới đây là **normative** từ TS 38.211 V15.10.0:

| `N_symb` | **No hopping** (`m'=0`) | **Hopping** (`m'=0`) | **Hopping** (`m'=1`) |
| -------- | ----------------------- | -------------------- | -------------------- |
| 4        | 2                       | 1                    | 1                    |
| 5        | 2                       | 1                    | 1                    |
| 6        | 3                       | 1                    | 2                    |
| 7        | 3                       | 1                    | 2                    |
| 8        | 4                       | 2                    | 2                    |
| 9        | 4                       | 2                    | 2                    |
| 10       | 5                       | 2                    | 3                    |
| 11       | 5                       | 2                    | 3                    |
| 12       | 6                       | 3                    | 3                    |
| 13       | 6                       | 3                    | 3                    |
| 14       | 7                       | 3                    | 4                    |

OCC engine phải:

- Đọc `N_SF,m'` từ bảng trên (hoặc logic tương đương).
- Sinh `w_i(m)` theo DFT form (Algorithm §11.1):

$$
w_i(m)=e^{j\frac{2\pi}{N_{\mathrm{SF}}}\phi_i(m)}
$$

- Xuất `w_i^*(m)` dưới dạng **negated phase code**: `occ_phase_code = (65536 - occ_phase) & 0xFFFF`.

Receiver dùng:

$$
w_i^*(m)
$$

---

## R10. Bước 7 — Thiết kế derotation

Reference sequence:

$$
r_{u,v}^{(\alpha,\delta)}(n)=e^{j\theta_{\mathrm{ref}}(n)}
$$

Receiver phải nhân với liên hợp:

$$
\left(r_{u,v}^{(\alpha,\delta)}(n)\right)^*=e^{-j\theta_{\mathrm{ref}}(n)}
$$

Thay vì complex multiplier tổng quát, baseline dùng `CORDIC rotation`.

### R10.1. Interface CORDIC

```systemverilog
// Inputs
input  logic                in_valid;
input  logic signed [15:0]  in_i;           // Q1.15
input  logic signed [15:0]  in_q;           // Q1.15
input  logic        [15:0]  in_phase;       // = -theta_ref (unsigned modulo)
input  logic                in_last;

// Outputs
output logic                out_valid;
output logic signed [17:0]  out_i;          // 18-bit signed
output logic signed [17:0]  out_q;          // 18-bit signed
output logic                out_last;
```

Xem Algorithm §26.1 cho CORDIC gain và compensation policy.

---

## R11. Bước 8 — Coherent combining

Statistic của receiver được xây dựng từ data RE hợp lệ:

$$
Z=\sum_{m=0}^{N_{\mathrm{SF}}-1}\sum_{n=0}^{11}\hat{Y}(m,n)\,e^{-j\theta_{\mathrm{ref}}(m,n)}\,w_i^*(m)
$$

### R11.1. Interface accumulator

```systemverilog
// Inputs (from CORDIC + OCC)
input  logic signed [17:0]  sample_i;       // CORDIC output × OCC
input  logic signed [17:0]  sample_q;
input  logic                sample_valid;
input  logic                sample_last;

// Outputs
output logic signed [25:0]  z_i;            // 26-bit signed accumulator
output logic signed [25:0]  z_q;            // 26-bit signed accumulator
output logic                z_valid;        // pulse when Z is ready
output logic                overflow_flag;  // saturated during accumulation
```

Số lần cộng phụ thuộc vào `N_SF`, không phải luôn 84.

Ví dụ baseline 14-symbol/no-hopping:

$$
N_{\mathrm{SF}}=7, \quad N_{\mathrm{RE,data}}=7\times12=84
$$

Xem Algorithm §27.1 cho accumulator overflow policy.

---

## R12. Bước 9 — MLE detector

Sau coherent combining:

$$
Z=Ad+n
$$

với `d` là BPSK/QPSK symbol.

### R12.1. Interface MLE

```systemverilog
// Inputs
input  logic signed [25:0]  z_i;
input  logic signed [25:0]  z_q;
input  logic                z_valid;
input  logic                uci_bit_count;  // 0=BPSK, 1=QPSK

// Outputs
output logic [1:0]          uci_bits;       // b(0)=uci_bits[0], b(1)=uci_bits[1]
output logic                result_valid;
output logic signed [31:0]  metric_best;    // metric of winning candidate
output logic signed [31:0]  metric_second;  // metric of 2nd best candidate
```

### R12.2. Một bit (BPSK)

Theo Algorithm §22, với BPSK NR (điểm trên đường chéo):

$$
\mathcal{D}_1=\left\{\frac{1+j}{\sqrt{2}},\;-\frac{1+j}{\sqrt{2}}\right\}
$$

Decision tương đương: `b(0) = sign(−(Re{Z} + Im{Z}))`.

Hardware: `uci_bits[0] = Z_I[25] ^ Z_Q[25]` nếu dùng XOR, hoặc `(Z_I + Z_Q) < 0 → b(0)=1`.

### R12.3. Hai bit (QPSK)

Decision tương đương:
- `b(0) = Z_I[25]` (sign bit of Z_I → 1 if negative)
- `b(1) = Z_Q[25]` (sign bit of Z_Q → 1 if negative)

Xem Algorithm §23.1 cho bảng mapping đầy đủ.

---

## R13. Bước 10 — Freeze numerical representation

Các lựa chọn baseline:

| **Tín hiệu**        | **Width** | **Biểu diễn**                      |
| -------------------- | --------- | ----------------------------------- |
| `Y_hat_I/Q`          | 16 bit    | signed Q1.15                        |
| `theta_ref`          | 16 bit    | unsigned modulo phase (phase code)  |
| CORDIC input I/Q     | 16 bit    | signed Q1.15                        |
| CORDIC output I/Q    | 18 bit    | signed                              |
| OCC phase            | 16 bit    | unsigned modulo phase               |
| Accumulator I/Q      | 26 bit    | signed                              |
| MLE metric           | 32 bit    | signed                              |
| `alpha_phase_code`   | 16 bit    | unsigned modulo phase               |
| `base_phase_code`    | 16 bit    | unsigned modulo phase               |
| `cs_phase_code`      | 16 bit    | unsigned modulo phase               |
| `phi_u(n)`           | 3 bit     | signed ({−3,−1,1,3})               |
| `n_cs`               | 8 bit     | unsigned, 0..255                    |
| `m_0`                | 4 bit     | unsigned, 0..11                     |
| `u`                  | 5 bit     | unsigned, 0..29                     |
| `v`                  | 1 bit     | unsigned, 0 or 1                    |
| `n_ID`               | 10 bit    | unsigned, 0..1023                   |
| `N_SF`               | 3 bit     | unsigned, 1..7                      |

Phase mapping:

$$
2\pi\equiv 2^{16}
$$

nên:

$$
\theta_{\text{code}}=\left\lfloor\frac{\theta}{2\pi}2^{16}\right\rfloor\bmod 2^{16}
$$

Xem Algorithm §13.1 cho xử lý góc âm.

Các width này là **implementation choice**.

---

## R14. Bước 11 — Freeze pipeline

Pipeline đề xuất:

```
S0  PUCCH/resource identification
S1  Grid memory read
S2  Sequence parameter / alpha alignment
S3  Phase generation (base_phase + cs_phase → theta_ref)
S4  CORDIC derotation (16 iterations, pipelined or iterative)
S5  OCC despreading (phase multiply or sign/swap)
S6  Coherent accumulation
S7  MLE metric computation
S8  Decision / output
```

Mỗi stage phải ghi rõ:

```
data width and format
valid signal
last signal
latency (cycles)
reset behavior
overflow/saturation policy
```

Invariant quan trọng:

```
Y_hat(m,n)    — từ grid
phase_ref(n)  — từ sequence engine
OCC(m)        — từ OCC engine
```

phải luôn thuộc cùng một RE. Phase ref thay đổi per-subcarrier `n`, OCC thay đổi per-symbol `m`.

---

## R15. Bước 12 — Top-level interface

Top-level nên tách hai mặt:

```
CONTROL PLANE
    AXI4-Lite / register interface
    ↓
configuration / status / result

DATA PLANE
    Grid SRAM / BRAM interface
    ↓
PUCCH processing engine
```

Không dùng AXI4-Lite như đường truyền một RE một lần cho datapath tốc độ cao.

### R15.1. Top-level port list

```systemverilog
module pucch_f1_receiver_top (
    // Clock & reset
    input  logic        clk,
    input  logic        rst_n,

    // Configuration (from AXI4-Lite or register file)
    input  logic        cfg_pucch_enable,
    input  logic [3:0]  cfg_pucch_start_symbol,
    input  logic [3:0]  cfg_pucch_nsymb,
    input  logic [8:0]  cfg_starting_prb,
    input  logic [8:0]  cfg_second_hop_prb,
    input  logic        cfg_intra_slot_hopping,
    input  logic [1:0]  cfg_group_hopping_mode,
    input  logic [9:0]  cfg_hopping_id,
    input  logic [3:0]  cfg_initial_cyclic_shift,
    input  logic [2:0]  cfg_time_domain_occ,
    input  logic        cfg_uci_bit_count,

    // Timing
    input  logic [9:0]  frame_idx,
    input  logic [4:0]  slot_idx,
    input  logic [3:0]  symbol_idx,

    // Grid memory interface
    output logic        grid_rd_en,
    output logic [14:0] grid_rd_addr,
    input  logic [31:0] grid_rd_data,       // {I[15:0], Q[15:0]}
    input  logic        grid_rd_valid,

    // Status & result
    output logic        busy,
    output logic        result_valid,
    output logic [1:0]  uci_bits,           // b(0)=uci_bits[0]
    output logic signed [31:0] metric_best,
    output logic signed [31:0] metric_second,
    output logic        error
);
```

### R15.2. Semantics của `metric_second`

- `metric_second` là metric của candidate có giá trị lớn thứ hai trong comparator tree.
- Dùng cho:
  - Confidence metric (`metric_best − metric_second`).
  - Debug khi MLE quyết định sai.
- Không dùng cho soft-output (PUCCH F1 không có soft-output UCI).

### R15.3. `error` flag semantics

`error` được set khi:

- Accumulator saturation xảy ra.
- CORDIC saturation xảy ra.
- Config invalid (xem R18.1).
- `result_valid` chỉ set khi `error = 0`.

---

## R16. Bước 13 — Thứ tự viết RTL

```
1. time_counter
2. pucch_cfg_reg
3. gold_sequence_gen
4. sequence_parameter_engine (u, v, n_cs, alpha)
5. phase_generator (phi_u ROM + cyclic shift accumulator)
6. occ_generator (phi_i LUT + phase computation)
7. grid_extractor (address gen + DM-RS skip)
8. cordic_rotator
9. coherent_accumulator
10. mle_detector
11. controller (FSM + datapath coordination)
12. top integration
```

Không nên viết top-level trước rồi mới quyết định semantics của submodule.

---

## R17. Verification plan

### R17.1. Level 1 — golden model

Golden model phải tạo:

```
c(n)                    // Gold sequence
f_gh, f_ss, u, v        // Sequence parameters
n_cs, alpha              // Cyclic shift
phi_u(n)                 // Base sequence values
r_uv^(alpha,delta)(n)    // Reference sequence
w_i(m)                   // OCC values
z(m',m,n)                // Transmitted sequence
Z                        // Coherent statistic
MLE metrics              // All candidate metrics
final UCI                // Decision bits
```

### R17.2. Golden model environment

| **Hạng mục**          | **Lựa chọn baseline** | **Ghi chú**                                                   |
| --------------------- | --------------------- | ------------------------------------------------------------- |
| Ngôn ngữ              | Python 3.10+          | NumPy cho complex math                                        |
| Thư viện 3GPP         | Viết từ đầu cho PUCCH F1 | Có thể tham chiếu `srsRAN` / MATLAB 5G Toolbox để cross-check |
| Test vector            | Sinh nội bộ + cross-check | Không phụ thuộc vector ngoài                                  |
| Phase representation   | 16-bit phase code     | Khớp RTL                                                      |
| Complex representation | `complex128`          | Đủ chính xác để làm reference                                 |

**Yêu cầu tối thiểu:** Golden model phải chạy được toàn bộ chain từ UCI bits → `z` → `Y_hat` (kênh lý tưởng) → `Z` → UCI bits quyết định, và verify `UCI_in == UCI_out` cho noise-free.

### R17.3. Level 2 — exhaustive configuration corners

Ít nhất:

```
N_symb = 4…14
hop = disabled/enabled
group hopping = neither/enable/disable
1-bit / 2-bit
multiple m0 (0, 3, 6, 11)
multiple OCC i (0, 1, N_SF-1)
multiple hopping_id (0, 1, 30, 1023)
```

### R17.4. Level 3 — RTL numerical verification

So sánh tại từng checkpoint:

```
phase       (theta_ref_code vs golden)
CORDIC output (out_i, out_q vs golden)
OCC output  (after despreading vs golden)
accumulator (acc_i, acc_q vs golden)
Z           (z_i, z_q vs golden)
metric      (metric_best, metric_second vs golden)
decision    (uci_bits vs golden)
```

### R17.5. Level 4 — performance

```
SNR sweep (Eb/N0 = -5..20 dB)
AWGN channel
frequency-selective channel
residual channel-estimation error
phase error
amplitude scaling
```

---

## R18. Error handling, reset, và RTL scope

### R18.1. Error handling policy

Các trường hợp invalid config và behavior:

| **Trường hợp**                                      | **Hành vi**                           |
| --------------------------------------------------- | ------------------------------------- |
| `pucch_nsymb < 4` hoặc `> 14`                       | Set `error`, không chạy               |
| `pucch_nsymb = 15+`                                 | Set `error`, không chạy               |
| `initial_cyclic_shift > 11`                          | Set `error`                           |
| `time_domain_occ >= N_SF`                            | Set `error`                           |
| `group_hopping_mode == 2'b11`                        | Set `error` (reserved)                |
| Accumulator saturation                               | Set `error` sau khi có `result_valid` |
| `pucch_start_symbol + pucch_nsymb > 14` (tràn slot)  | Set `error` (xem R18.3)              |

`result_valid` chỉ được set khi `error = 0`. `uci_bits` không có ý nghĩa khi `error = 1`.

### R18.2. Reset và reconfiguration semantics

**Reset (`rst_n = 0`):**

- Xóa toàn bộ pipeline register.
- Reset phase accumulator về 0.
- Reset Gold sequence init state.
- Xóa config register.
- `busy = 0`, `result_valid = 0`, `error = 0`.

**Config thay đổi giữa các slot:**

- Nếu config thay đổi sau khi `busy` đã set: hardware hoàn thành PUCCH occasion hiện tại, sau đó reload config cho slot kế tiếp.
- Không cần reset accumulator giữa các slot, vì mỗi PUCCH occasion là độc lập.
- Phase accumulator phải reset đầu mỗi PUCCH occasion (initial value = `phase_code(α·δ) = 0` với baseline).

**`busy` timing:**

- `busy = 1` từ khi S0 bắt đầu đến khi S8 hoàn thành.
- `result_valid` pulse 1 chu kỳ khi `uci_bits` valid.

### R18.3. Inter-slot và cross-slot behavior

**Inter-slot frequency hopping (PUCCH F1 trải qua nhiều slot):**

- **Out of scope** cho baseline revision A.
- Lý do: baseline `N_symb = 14` vừa đủ một slot.
- Nếu cần hỗ trợ: resource extractor phải có logic wrap slot, và timing model phải hỗ trợ PUCCH occasion spanning multiple slots. Ghi rõ "out of scope" trong spec để tránh hiểu nhầm.

**Tràn slot (`pucch_start_symbol + pucch_nsymb > 14`):**

- **Out of scope** cho baseline revision A.
- Nếu xảy ra: set `error`.
- Ghi rõ trong error handling policy.

### R18.4. RTL revision A scope

| **Nhánh**                                                                     | **Rev A: test?** | **Rev A: port?** | **Ghi chú**                             |
| ----------------------------------------------------------------------------- | ---------------- | ---------------- | --------------------------------------- |
| `N_symb=14`, no hopping, `group_hopping=neither`, 1 hoặc 2 bit, `i=0`        | **Có** (chính)   | Có               | Baseline datapath                       |
| `N_symb = 4..13`, no hopping                                                  | Không            | Có               | Port config tồn tại, datapath chưa test |
| `intra_slot_hopping = 1`                                                      | Không            | Có               | Port config tồn tại, datapath chưa test |
| `group_hopping = enable/disable`                                              | Không            | Có               | Port config tồn tại, datapath chưa test |
| Inter-slot hopping                                                            | Không            | Không            | Out of scope                            |
| Tràn slot                                                                     | Không            | Không            | Out of scope (set error)                |

**Khuyến nghị:** Revision A nên implement đủ datapath cho tất cả các nhánh (chỉ cần thêm mux), nhưng verification chỉ cover baseline. Revision B mở rộng verification.

---

## R19. Definition of Done

Thiết kế được coi là hoàn thành khi:

1. Mọi tham số 3GPP liên quan Format 1 đã được phân loại `fixed/configurable`.
2. Tất cả equation trong algorithm spec khớp Release 15.10.0.
3. Phase-accumulator model khớp sample-by-sample với direct complex sequence generation.
4. Grid extraction xác định đúng mọi RE.
5. OCC despreading đúng theo từng `N_SF`.
6. CORDIC có latency và scaling được freeze.
7. `Z` khớp golden model trong sai số fixed-point đã định nghĩa.
8. MLE chọn đúng candidate trong mọi testbench chuẩn.
9. RTL port contract không còn tín hiệu có semantics mơ hồ.
10. Verification bao phủ các nhánh configurable chính.

---

## R20. Các điều không được hard-code nhầm

```
KHÔNG hard-code vì Format 1:
    intra_slot_hopping = 0
    group_hopping = neither
    N_symb = 14
    starting_symbol = constant
    m0 = constant
    OCC i = constant
    hopping_id = constant

CÓ THỂ fixed bởi baseline implementation:
    Rx antenna = 1
    UE context = 1
    Q1.15 input
    16-bit phase
    16-iteration CORDIC
    26-bit accumulator
    m_int = 0 (cho PUCCH F1 Rel-15)
    delta = 0 (cho PUCCH F1 Rel-15)
```

Đây là ranh giới giữa **physical-channel specification** và **hardware revision**.

---

## R21. Minor consistency pass

Các thuật ngữ đã được chuẩn hóa giữa 2 tài liệu:

| **Thuật ngữ**                    | **Định nghĩa thống nhất**                         |
| -------------------------------- | -------------------------------------------------- |
| `phase code`                     | 16-bit unsigned modulo phase (không phải Q-format)  |
| `spreading symbol index m`       | 0..N\_SF-1, không phải slot/symbol                 |
| `hop index m'`                   | 0 hoặc 0,1 (intra-slot hopping)                    |
| `symbol index l`                 | 0..N_symb-1 trong PUCCH transmission               |
| `slot index n_{s,f}^μ`           | 0..19 trong frame (μ=1)                            |
| `N_SF`                           | Số data spreading symbol, phụ thuộc `N_symb` và hopping |
| `N_{symb}^{slot}`                | 14 (hằng số với μ=1)                               |
| `N_{RE,data}`                    | Tổng data RE = `N_SF × 12`                         |
| `Z`                              | Coherent statistic sau accumulator                  |
| `metric_best`, `metric_second`   | MLE metric của candidate tốt nhất/nhì              |

---
---

# PHẦN III — CROSS-REFERENCE & CÂU HỎI MỞ

## CR. Cross-reference hai tài liệu

| **Algorithm Spec** | **Design Roadmap** | **Nội dung**                   |
| ------------------ | ------------------ | ------------------------------ |
| §3                 | §R12               | Modulation / MLE               |
| §5 (Table φ_u)     | §R7                | Sequence generation / φ table  |
| §6 (hopping)       | §R7                | Group/sequence hopping         |
| §9–§11 (OCC)       | §R9                | OCC / spreading                |
| §10 (N_SF table)   | §R9.2              | N_SF lookup table              |
| §11.5 (DM-RS)      | §R8.3              | DM-RS pattern                  |
| §12–§17            | §R10               | Phase accumulator / derotation |
| §18–§20            | §R11               | Coherent combining             |
| §21–§24            | §R12               | MLE                            |
| §25–§27            | §R13               | Fixed-point contract           |
| §28–§32            | §R17, §R18         | Debug / verification           |

---

## Q. Câu hỏi mở cần verify trước khi freeze

Các điểm sau cần xác nhận với spec gốc hoặc cross-check:

1. **Gold sequence chu kỳ:** `n_cs` generator init mỗi radio frame — cần xác nhận Gold sequence cho `f_gh`/`v` cũng init mỗi radio frame (đã confirm từ §6.3.2.2.1: "initialized at the beginning of each radio frame").
2. **`v` cho PUCCH F1:** Mode `neither` → `v=0`. Mode `enable`/`disable` → `v = c(8·n_{s,f}^μ + n_hop)`, có thể = 0 hoặc 1.
3. **CORDIC gain compensation:** Cần chọn lựa chọn (a)/(b)/(c) trong Algorithm §26.1 cho RTL revision A. Baseline recommend (c).
4. **`Y_hat` amplitude convention:** Cần khóa giữa equalizer và algorithm core.
5. **RTL revision A scope:** Cần xác nhận implementation đủ datapath cho tất cả nhánh hay chỉ baseline.

---

## Tổng kết thay đổi v1.3 → v1.4

### Design Combined v1.4

- **Sửa tất cả bảng**: Header columns không còn bị dồn vào cột đầu.
- **Đồng bộ N_SF table**: Sửa theo TS 38.211 Table 6.3.2.4.1-1 chính thức (v1.3 bị sai nhiều dòng).
- **Thêm port definitions**: Tất cả interface có SystemVerilog port list với bit width rõ ràng.
- **Thêm R6 config port list**: Đầy đủ bit width cho mỗi config register.
- **Thêm R7.1**: Interface sequence engine.
- **Thêm R8.1**: Interface resource extractor với bit width.
- **Thêm R9.1**: Interface OCC engine.
- **Thêm R10.1**: Interface CORDIC.
- **Thêm R11.1**: Interface accumulator.
- **Thêm R12.1**: Interface MLE detector.
- **Sửa R13**: Thêm đầy đủ bit width cho tất cả tín hiệu trung gian (phi_u, n_cs, m_0, u, v, n_ID, N_SF).
- **Sửa R15.1**: Top-level port list đầy đủ SystemVerilog.
- **Sửa R16**: Thêm `gold_sequence_gen` vào thứ tự RTL.
- **Loại bỏ câu hỏi mở đã giải quyết**: DM-RS pattern, N_SF table, OCC table đã có đầy đủ trong Algorithm Spec v1.5.
- **Sửa công thức LaTeX** cho chuẩn format MathJax.
- **PHẦN I** giờ tham chiếu sang Algorithm Spec file riêng, tránh duplicate.

### Algorithm Specification v1.5 (file riêng)

- Xem changelog trong file `PUCCH_F1_Algorithm_Spec_PhaseAccumulator_MLE_v1.4_vi.md`.

---
