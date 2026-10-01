# PUCCH Format 1 — Algorithm Specification

**Phiên bản:** 1.5  
**Mục tiêu:** gNB L1 Receiver xử lý PUCCH Format 1  
**Tiêu chuẩn:** 3GPP TS 38.104 / 38.212 / 38.211 / 38.213, V15.10.0, Release 15  
**Hệ thống:** FR1, 100 MHz, 30 kHz SCS, Normal CP, 122.88 MHz implementation reference

---

## 0. Phạm vi thuật toán

**Algorithm boundary:** bắt đầu từ $Y_{hat}$ sau channel estimation/equalization và kết thúc ở UCI decision  

> **Quy ước hiển thị công thức:** Các phương trình được viết bằng LaTeX display-math `$$...$$` để Obsidian MathJax hiển thị đúng chỉ số dưới/trên, phân số và số mũ.

---

## 1. Mục tiêu thuật toán

Mục tiêu của receiver là khôi phục symbol $d(0)$ mang UCI của PUCCH Format 1 từ các resource elements thuộc PUCCH occasion.

Thiết kế phần cứng dùng **phase-domain reference generation** và **phase accumulator**, nhưng algorithm spec trước hết phải trình bày đúng **chuỗi phát theo 3GPP**. Sau khi khóa mô hình 3GPP, tài liệu mới biến từng bước thành một cách tính tương đương bằng cộng dồn pha.

Luồng tổng quát:

```text
3GPP UE transmitter model
        ↓
sequence r_uv^(alpha,delta)(n)
        ↓
PUCCH Format 1 sequence modulation
        ↓
block-wise spreading w_i(m)
        ↓
z(m' Nsc N_SF + m Nsc + n)
        ↓
received / equalized Y_hat
        ↓
conjugate reference + conjugate OCC
        ↓
coherent statistic Z
        ↓
MLE
        ↓
UCI bits
````


---

## 2. Phân biệt chuẩn 3GPP và hardware realization

### 2.1. 3GPP-defined

Các quan hệ dưới đây phải được xem là normative model của algorithm:

- PUCCH Format 1 dùng tối đa hai bit.
- Một bit → BPSK; hai bit → QPSK.
- Symbol `d(0)` được nhân với low-PAPR sequence.
- Sequence sau đó được block-wise spread bằng `w_i(m)`.
- Sequence group `u` và sequence number `v` được xác định bởi §6.3.2.2.1.
- Cyclic shift `alpha` được xác định bởi §6.3.2.2.2.
- `w_i(m)` lấy từ Table 6.3.2.4.1-2.
- Mapping lên resource elements loại các RE dành cho DM-RS.

### 2.2. Hardware-defined

Baseline hardware:

| **Tham số**    | **Giá trị**                  |
| -------------- | ---------------------------- |
| I/Q            | signed Q1.15, 16 bit         |
| Phase          | 16 bit modulo `2^{16}`       |
| CORDIC         | rotation mode, 16 iterations |
| CORDIC output  | 18-bit I/Q                   |
| Accumulator    | 26-bit I/Q                   |
| Rx antenna     | 1                            |

Các giá trị này không thay đổi ý nghĩa toán học của sequence 3GPP; chúng chỉ là cách biểu diễn số trong RTL.

---

## 3. PUCCH Format 1: modulation của bit UCI

### 3.0. Luồng modulation tổng thể phía UE

Theo TS 38.211 §6.3.2.4.1, UE thực hiện chuỗi xử lý sau cho PUCCH Format 1:

```text
UCI bits b(0), [b(1)]
        ↓
Modulation mapper (§5.1)
  M_bit=1 → BPSK
  M_bit=2 → QPSK
        ↓
d(0)  (1 complex symbol)
        ↓
Nhân với low-PAPR sequence r_{u,v}^{(α,δ)}(n)
  y(n) = d(0) · r_{u,v}^{(α,δ)}(n),  n = 0..11
        ↓
Block-wise spreading với OCC w_i(m)
  z(m'·N_sc·N_SF + m·N_sc + n) = w_i(m) · y(n)
        ↓
Mapping lên physical resource elements (§6.3.2.4.2)
  - Bỏ qua các RE dành cho DM-RS
  - Thứ tự: tăng dần k (subcarrier), rồi l (symbol)
  - Antenna port p = 2000
        ↓
Nhân amplitude scaling β_{PUCCH,1}
        ↓
Transmit
```

### 3.1. Modulation mapper — BPSK (TS 38.211 §5.1.2)

Công thức normative cho BPSK:

$$
d(i)=\frac{1}{\sqrt{2}}\left(1-2b(i)\right)+j\,\frac{1}{\sqrt{2}}\left(1-2b(i)\right)
$$

Với PUCCH Format 1 chỉ có $d(0)$:

$$
d(0)=\frac{1}{\sqrt{2}}\left(1-2b(0)\right)+j\,\frac{1}{\sqrt{2}}\left(1-2b(0)\right)
$$

Bảng constellation mapping BPSK:

| $b(0)$ | $d(0)$ (I + jQ)  | Phase  | Diễn giải                  |
| ------ | ----------------- | ------ | -------------------------- |
| 0      | $+1/√2 + j·1/√2$ | +45°   | điểm trên đường chéo +45°  |
| 1      | $−1/√2 − j·1/√2$ | −135°  | điểm trên đường chéo −135° |

**Lưu ý quan trọng:** BPSK trong NR PUCCH **không nằm trên trục thực**. Hai điểm nằm trên đường chéo $I = Q$. Điều này ảnh hưởng trực tiếp tới MLE sign-decision ở §22.

### 3.2. Modulation mapper — QPSK (TS 38.211 §5.1.3)

Công thức normative cho QPSK:

$$
d(i)=\frac{1}{\sqrt{2}}\left(1-2b(2i)\right)+j\,\frac{1}{\sqrt{2}}\left(1-2b(2i+1)\right)
$$

Với PUCCH Format 1, $i = 0$ nên dùng $b(0)$ và $b(1)$:

$$
d(0)=\frac{1}{\sqrt{2}}\left(1-2b(0)\right)+j\,\frac{1}{\sqrt{2}}\left(1-2b(1)\right)
$$

Bảng constellation mapping QPSK:

| $b(0)$ | $b(1)$ | $d(0)$ (I + jQ)  | Phase  |
| ------ | ------ | ----------------- | ------ |
| 0      | 0      | $+1/√2 + j·1/√2$ | +45°   |
| 0      | 1      | $+1/√2 − j·1/√2$ | −45°   |
| 1      | 0      | $−1/√2 + j·1/√2$ | +135°  |
| 1      | 1      | $−1/√2 − j·1/√2$ | −135°  |

**Normalization:** $1/√2$ là normalization để $|d(0)|² = 1$.

### 3.3. Bit ordering convention

- $b(0)$ là **bit đầu tiên** của UCI block (theo TS 38.212 §6.3.x).
- Trong hardware, `uci_bits[1:0]` cần được định nghĩa rõ:
  - `uci_bits[0]` = $b(0)$ (bit đầu tiên, cũng là bit LSB của UCI vector).
  - `uci_bits[1]` = $b(1)$ (chỉ valid khi $uci_{bit}_{count} = 2$).

**Quan trọng:** Bit ordering giữa golden model (Python) và RTL phải thống nhất. Đây là nguồn lỗi phổ biến khi MLE sign-decision trông "đúng" nhưng bit cuối bị flip.

### 3.4. Phase-domain representation cho hardware

Cho RTL, $d(0)$ có thể biểu diễn bằng phase code 16-bit:

| $b(0)$ | $b(1)$ | Phase (radian) | Phase code 16-bit |
| ------ | ------ | -------------- | ------------------ |
| 0      | —      | $π/4$          | `0x2000`           |
| 1      | —      | $−3π/4$        | `0xA000`           |
| 0      | 0      | $π/4$          | `0x2000`           |
| 0      | 1      | $−π/4$         | `0xE000`           |
| 1      | 0      | $3π/4$         | `0x6000`           |
| 1      | 1      | $−3π/4$        | `0xA000`           |

---

## 4. Sinh low-PAPR sequence theo 3GPP

TS 38.211 §6.3.2.4.1 quy định:

$$
y(n)=d(0)\,r_{u,v}^{(\alpha,\delta)}(n)
$$

với:

$$
n=0,1,\ldots,N_{\mathrm{sc}}^{\mathrm{RB}}-1
$$

Đối với một PRB:

$$
N_{\mathrm{sc}}^{\mathrm{RB}}=12
$$

Do đó có đúng 12 $y(n)$ cho mỗi block.

---

## 5. Định nghĩa low-PAPR sequence gốc

TS 38.211 §5.2.2 định nghĩa low-PAPR sequence thông qua cyclic shift của base sequence.

Với độ dài:

$$
M_{\mathrm{ZC}}=12
$$

ta thuộc trường hợp $M_{\mathrm{ZC}}$ < 36, vì vậy base sequence được tạo bởi bảng $\phi_{u}(n)$ trong Table 5.2.2.2-2.

Công thức chuẩn:

$$
r_{u,v}(n)=e^{j\frac{\pi}{4}\phi_u(n)}
$$

với:

$$
n=0,1,\ldots,11
$$

và $\phi_{u}(n)$ là giá trị nguyên thuộc bảng 3GPP.

### Quan trọng

Trong thiết kế hardware, **không được thay hệ số $π/4$ thành $π/2$**. Đây là phase mapping của bảng $\phi_{u}(n)$ cho sequence length 12.

### 5.1. Table 5.2.2.2-2: Định nghĩa $φ_{u}(n)$ cho $M_{ZC} = 12$

Bảng dưới đây là **normative** từ TS 38.211 V15.10.0, Table 5.2.2.2-2.

| $u$ | $φ(0)$ | $φ(1)$ | $φ(2)$ | $φ(3)$ | $φ(4)$ | $φ(5)$ | $φ(6)$ | $φ(7)$ | $φ(8)$ | $φ(9)$ | $φ(10)$ | $φ(11)$ |
| --- | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------- | ------- |
| 0   | −3     | 1      | −3     | −3     | −3     | 3      | −3     | −1     | 1      | 1      | 1       | −3      |
| 1   | −3     | 3      | 1      | −3     | 1      | 3      | −1     | −1     | 1      | 3      | 3       | 3       |
| 2   | −3     | 3      | 3      | 1      | −3     | 3      | −1     | 1      | 3      | −3     | 3       | −3      |
| 3   | −3     | −3     | −1     | 3      | 3      | 3      | −3     | 3      | −3     | 1      | −1      | −3      |
| 4   | −3     | −1     | −1     | 1      | 3      | 1      | 1      | −1     | 1      | −1     | −3      | 1       |
| 5   | −3     | −3     | 3      | 1      | −3     | −3     | −3     | −1     | 3      | −1     | 1       | 3       |
| 6   | 1      | −1     | 3      | −1     | −1     | −1     | −3     | −1     | 1      | 1      | 1       | −3      |
| 7   | −1     | −3     | 3      | −1     | −3     | −3     | −3     | −1     | 1      | −1     | 1       | −3      |
| 8   | −3     | −1     | 3      | 1      | −3     | −1     | −3     | 3      | 1      | 3      | 3       | 1       |
| 9   | −3     | −1     | −1     | −3     | −3     | −1     | −3     | 3      | 1      | 3      | −1      | −3      |
| 10  | −3     | 3      | −3     | 3      | 3      | −3     | −1     | −1     | 3      | 3      | 1       | −3      |
| 11  | −3     | −1     | −3     | −1     | −1     | −3     | 3      | 3      | −1     | −1     | 1       | −3      |
| 12  | −3     | −1     | 3      | −3     | −3     | −1     | −3     | 1      | −1     | −3     | 3       | 3       |
| 13  | −3     | 1      | −1     | −1     | 3      | 3      | −3     | −1     | −1     | −3     | −1      | −3      |
| 14  | 1      | 3      | −3     | 1      | 3      | 3      | 3      | 1      | −1     | 1      | −1      | 3       |
| 15  | −3     | 1      | 3      | −1     | −1     | −3     | −3     | −1     | −1     | 3      | 1       | −3      |
| 16  | −1     | −1     | −1     | −1     | 1      | −3     | −1     | 3      | 3      | −1     | −3      | 1       |
| 17  | −1     | 1      | 1      | −1     | 1      | 3      | 3      | −1     | −1     | −3     | 1       | −3      |
| 18  | −3     | 1      | 3      | 3      | −1     | −1     | −3     | 3      | 3      | −3     | 3       | −3      |
| 19  | −3     | −3     | 3      | −3     | −1     | 3      | 3      | 3      | −1     | −3     | 1       | −3      |
| 20  | 3      | 1      | 3      | 1      | 3      | −3     | −1     | 1      | 3      | 1      | −1      | −3      |
| 21  | −3     | 3      | 1      | 3      | −3     | 1      | 1      | 1      | 1      | 3      | −3      | 3       |
| 22  | −3     | 3      | 3      | 3      | −1     | −3     | −3     | −1     | −3     | 1      | 3       | −3      |
| 23  | 3      | −1     | −3     | 3      | −3     | −1     | 3      | 3      | 3      | −3     | −1      | −3      |
| 24  | −3     | −1     | 1      | −3     | 1      | 3      | 3      | 3      | −1     | −3     | 3       | 3       |
| 25  | −3     | 3      | 1      | −1     | 3      | 3      | −3     | 1      | −1     | 1      | −1      | 1       |
| 26  | −1     | 1      | 3      | −3     | 1      | −1     | 1      | −1     | −1     | −3     | 1       | −1      |
| 27  | −3     | −3     | 3      | 3      | 3      | −3     | −1     | 1      | −3     | 3      | 1       | −3      |
| 28  | 1      | −1     | 3      | 1      | 1      | −1     | −1     | −1     | 1      | 3      | −3      | 1       |
| 29  | −3     | 3      | −3     | 3      | −3     | −3     | 3      | −1     | −1     | 1      | 3       | −3      |

### 5.2. Phase code 16-bit tương ứng cho $φ_{u}(n)$

Vì $φ$ nhận giá trị trong tập ${−3, −1, 1, 3}$, bảng chuyển đổi sang phase code 16-bit:

$$
\text{base\_phase\_code} = \text{phase\_code}\left(\frac{\pi}{4}\phi\right)
$$

| $φ$ | $(π/4)·φ$ (radian) | Phase code 16-bit |
| --- | ------------------- | ----------------- |
| −3  | $−3π/4$             | `0xA000`          |
| −1  | $−π/4$              | `0xE000`          |
| 1   | $+π/4$              | `0x2000`          |
| 3   | $+3π/4$             | `0x6000`          |

Bảng này cho phép RTL thực hiện lookup từ $φ$ (2 bit sign + 1 bit value) sang phase code mà không cần multiplier.

---

## 6. Sequence group $u$ và sequence number $v$

TS 38.211 §6.3.2.2.1 xác định:

$$
u=\left(f_{\mathrm{gh}}+f_{\mathrm{ss}}\right)\bmod 30
$$

và $v$ phụ thuộc `pucch-GroupHopping`.

Ba mode phải được hỗ trợ ở configuration layer:

```
neither
enable
disable
```

Trong đó $n_{\text{ID}}$ được lấy từ higher-layer parameter `hoppingId` nếu configured, nếu không thì $n_{\text{ID}} = N_{\text{ID}}^{\text{cell}}$.

### 6.1. Mode `neither`

Theo TS 38.211 §6.3.2.2.1:

$$
f_{\mathrm{gh}}=0
$$

$$
f_{\mathrm{ss}}=n_{\mathrm{ID}}\bmod 30
$$

$$
v=0
$$

suy ra:

$$
u=n_{\mathrm{ID}}\bmod 30
$$

Đây chỉ là một mode. Nó **không phải tính chất cố định của PUCCH Format 1**.

### 6.2. Mode `enable`

Theo TS 38.211 §6.3.2.2.1:

$$
f_{\mathrm{gh}}=\left(\sum_{m=0}^{7}2^{m}\,c\!\left(8n_{s,f}^{\mu}+m\right)\right)\bmod 30
$$

$$
f_{\mathrm{ss}}=n_{\mathrm{ID}}\bmod 30
$$

$$
v=c\!\left(8n_{s,f}^{\mu}+n_{\mathrm{hop}}\right)
$$

trong đó:
- Pseudo-random sequence $c(i)$ được sinh theo §5.2.1 (xem §6.5).
- Generator được **khởi tạo đầu mỗi radio frame** với:

$$
c_{\mathrm{init}}=\left\lfloor\frac{n_{\mathrm{ID}}}{30}\right\rfloor
$$

- $n_{hop} = 0$ nếu không có intra-slot frequency hopping, $n_{hop} = 0$ cho hop đầu, $n_{hop} = 1$ cho hop thứ hai.

### 6.3. Mode `disable`

Theo TS 38.211 §6.3.2.2.1:

$$
f_{\mathrm{gh}}=0
$$

$$
f_{\mathrm{ss}}=n_{\mathrm{ID}}\bmod 30
$$

$$
v=c\!\left(8n_{s,f}^{\mu}+n_{\mathrm{hop}}\right)
$$

trong đó pseudo-random sequence generator được khởi tạo đầu mỗi radio frame với:

$$
c_{\mathrm{init}}=2^{5}\left\lfloor\frac{n_{\mathrm{ID}}}{30}\right\rfloor+\left(n_{\mathrm{ID}}\bmod 30\right)
$$

### 6.4. Design rule

Sequence generator không được assume:

```
u = n_ID mod 30
v = 0
```

trong mọi trường hợp. Hai biểu thức này chỉ đúng cho mode `neither`.

### 6.5. Gold sequence initialization

Gold sequence $c(i)$ theo TS 38.211 §5.2.1:

$$
c(n)=\left(x_1(n+N_C)+x_2(n+N_C)\right)\bmod 2
$$

$$
x_1(n+31)=\left(x_1(n+3)+x_1(n)\right)\bmod 2
$$

$$
x_2(n+31)=\left(x_2(n+3)+x_2(n+2)+x_2(n+1)+x_2(n)\right)\bmod 2
$$

với:

$$
N_C=1600
$$

Initialization:

$$
\begin{aligned}x_1(0)&=1,\\x_1(n)&=0,\quad n=1,\ldots,30\end{aligned}
$$

$$
c_{\mathrm{init}}=\sum_{i=0}^{30}x_2(i)\,2^i
$$

### 6.6. Bảng tổng hợp `c_init` cho PUCCH Format 1

| **Đại lượng**               | **`c_init`**                                                          | **Chu kỳ cập nhật** | **Nguồn**              |
| --------------------------- | --------------------------------------------------------------------- | ------------------- | ---------------------- |
| `f_gh` (mode `enable`)      | $floor(n_{ID} / 30)$                                                   | mỗi radio frame     | TS 38.211 §6.3.2.2.1  |
| $v$ (mode `enable`)         | $floor(n_{ID} / 30)$ (cùng sequence với `f_gh`)                        | mỗi slot            | TS 38.211 §6.3.2.2.1  |
| $v$ (mode `disable`)        | $2^5 · floor(n_{ID}/30) + (n_{ID} mod 30)$                               | mỗi radio frame     | TS 38.211 §6.3.2.2.1  |
| $n_{cs}$                      | `n_ID`                                                                | mỗi radio frame     | TS 38.211 §6.3.2.2.2  |

**Lưu ý:** Giá trị `c_init` trên đây là theo Rel-15. Khi mở rộng sang các release sau, cần verify lại với spec tương ứng.

### 6.7. Bảng tổng hợp $u$, $v$ theo mode

| **Mode**    | **`f_gh`**          | **`f_ss`**          | **$v$**                                   |
| ----------- | ------------------- | ------------------- | ----------------------------------------- |
| `neither`   | 0                   | `n_ID mod 30`       | 0                                         |
| `enable`    | Gold-seq mod 30     | `n_ID mod 30`       | $c(8·n_{s,f}^μ + n_{hop})$                  |
| `disable`   | 0                   | `n_ID mod 30`       | $c(8·n_{s,f}^μ + n_{hop})$                  |

---

## 7. Cyclic shift $alpha$ theo 3GPP

TS 38.211 §6.3.2.2.2 định nghĩa:

$$
\alpha_l=\frac{2\pi}{N_{\mathrm{sc}}^{\mathrm{RB}}}\left(\left(m_0+m_{\mathrm{cs}}+n_{\mathrm{cs}}(n_{s,f}^{\mu},l+l')\right)\bmod N_{\mathrm{sc}}^{\mathrm{RB}}\right)
$$

trong đó:

- $n_{s,f}^μ$ là slot number trong radio frame.
- $l$ là OFDM symbol number trong PUCCH transmission, $l$ = 0 tương ứng symbol đầu tiên của PUCCH.
- $l'$ là index OFDM symbol trong slot tương ứng symbol đầu tiên PUCCH (= `pucch_start_symbol`).
- $m_{0}$ lấy từ TS 38.213 (higher-layer configuration cho PUCCH format 0 và 1).
- $m_{cs}$ = 0 cho PUCCH Format 1 (chỉ format 0 mới có $m_{cs}$ ≠ 0).

Với PUCCH Format 1 baseline:

$$
m_{\mathrm{cs}}=0
$$

### 7.0. Định nghĩa `m_int` cho PUCCH Format 1

Trong Rel-15, với **PUCCH Format 1** ở FR1:

$$
m_{\mathrm{int}}=0
$$

Đây là normative cho PUCCH F1. Trường `m_int` chỉ có ý nghĩa với các kênh hỗ trợ interlaced mapping (không áp dụng cho PUCCH FR1 Rel-15).

**Design rule:** Receiver baseline có thể hard-code $m_{int} = 0$ cho PUCCH F1. Port config chỉ cần giữ nếu system architecture mở rộng sang các kênh khác (ví dụ interlace-based PUCCH cho các cấu hình đặc biệt).

### 7.1. Gold-derived cyclic-shift term

TS 38.211 §6.3.2.2.2 định nghĩa:

$$
n_{\mathrm{cs}}(n_{s,f}^{\mu},l)=\sum_{m=0}^{7}2^m\,c\left(8N_{\mathrm{symb}}^{\mathrm{slot}}n_{s,f}^{\mu}+8l+m\right)
$$

Pseudo-random sequence $c(i)$ được sinh theo §5.2.1 với:

$$
c_{\mathrm{init}}=n_{\mathrm{ID}}
$$

Generator được khởi tạo đầu mỗi radio frame.

**Ý nghĩa:**
- $n_{cs}$ là 8-bit (range 0..255) nhưng chỉ giá trị `mod 12` mới dùng trong $\alpha$.
- $c(i)$ cần được sinh cho đủ số bit: $8 × N_{symb}^{slot} × n_{s,f}^μ + 8l + 7$. Với μ=1, slot 19, symbol 13: cần $c(i)$ tới $i = 8 × 14 × 19 + 8 × 13 + 7 = 2235$.
- Tổng cộng cần sinh Gold sequence tối thiểu $N_{C} + 2236 = 3836$ bit.

### 7.2. Ý nghĩa

$m_{0}$ là configuration/index.

$n_{cs}$ là thành phần pseudo-random thay đổi theo slot/symbol.

$\alpha_{l}$ mới là phase rotation thực sự dùng khi tạo sequence.

Không được coi:

$$
m_0=\alpha
$$

vì hai đại lượng có đơn vị và ý nghĩa khác nhau.

### 7.3. Định nghĩa index

Để tránh nhầm lẫn giữa hardware counter và 3GPP index, bảng dưới đây khóa định nghĩa:

| **Ký hiệu**            | **Định nghĩa**                            | **Range (μ=1)** | **Ghi chú**                          |
| ---------------------- | ----------------------------------------- | --------------- | ------------------------------------ |
| $l$                    | symbol index **trong PUCCH transmission** | 0..$N_{symb}$−1 | Dùng trong $n_{cs}$                  |
| $l'$                   | start symbol offset trong slot            | 0..13           | = `pucch_start_symbol`               |
| $n_{s,f}^μ$            | slot index **trong frame**                | 0..19           | $frame_{idx} × 20 + slot_{idx}_{in}_{frame}$ |
| $N_{symb}^{slot}$      | số symbol trong một slot                  | 14              | Với Normal CP, μ=1                   |
| $n_{s,f}^μ$ (hardware) | = $frame_{idx}[9:0] × 20 + slot_{idx}[4:0]$   | 0..20479        | Phải modulo theo frame nếu cần       |
| $l$ (hardware)         | = `symbol_idx[3:0]`                       | 0..13           |                                      |

**Quan trọng:** $N_{symb}^{slot}$ trong công thức $n_{cs}$ là **14** (không phải 12), vì đây là số symbol của slot theo numerology. Số symbol data thực tế (sau khi loại DM-RS) là $N_{SF}$.

---

## 8. Low-PAPR sequence sau cyclic shift

Kết hợp base sequence với cyclic shift:

$$
r_{u,v}^{(\alpha,\delta)}(n)=r_{u,v}(n)e^{j\alpha(n+\delta)}
$$

với $delta$ theo định nghĩa sequence generation của TS 38.211.

### 8.1. Định nghĩa $δ$ cho PUCCH Format 1

Theo TS 38.211 §6.3.2.2, PUCCH formats 0, 1, 3, 4 dùng sequences $r_{u,v}^{(α,δ)}(n)$ với $δ = 0$:

- Với PUCCH Format 1 (baseline FR1, Rel-15): **$δ = 0$**.

**Design rule:** Với baseline PUCCH F1, hard-code $δ = 0$. Phase accumulator khởi tạo từ `0` (§14), không cần thêm offset.

Đối với receiver Format 1 baseline, sequence này phải được đánh giá đúng theo index $n$ của 12 subcarriers.

Thay base sequence:

$$
r_{u,v}(n)=e^{j\frac{\pi}{4}\phi_u(n)}
$$

ta có:

$$
r_{u,v}^{(\alpha,\delta)}(n)=e^{j\frac{\pi}{4}\phi_u(n)}e^{j\alpha(n+\delta)}
$$

và do đó:

$$
r_{u,v}^{(\alpha,\delta)}(n)=\exp\left(j\left[\frac{\pi}{4}\phi_u(n)+\alpha(n+\delta)\right]\right)
$$

Đây là bước quan trọng để chuyển sang phase-domain implementation.

**Lưu ý:** Công thức trên là **base sequence đã cyclic shift**, chưa nhân OCC. OCC $w_{i}(m)$ được áp riêng ở §9.

---

## 9. PUCCH Format 1 block-wise spreading — công thức chuẩn 3GPP

Đây là công thức cần giữ **giống notation 3GPP** trong algorithm reference model:

$$
z\left(m' N_{\mathrm{sc}}^{\mathrm{RB}}N_{\mathrm{SF},m'}^{\mathrm{PUCCH},1}+mN_{\mathrm{sc}}^{\mathrm{RB}}+n\right)=w_i(m)y(n)
$$

với:

$$
n=0,1,\ldots,N_{\mathrm{sc}}^{\mathrm{RB}}-1
$$

$$
m=0,1,\ldots,N_{\mathrm{SF},m'}^{\mathrm{PUCCH},1}-1
$$

và:

$$
m'=\begin{cases}0,&\text{không có intra-slot frequency hopping},\\0,1,&\text{có intra-slot frequency hopping},\end{cases}
$$

Đây là công thức chuẩn cần dùng làm **reference equation** trước khi viết bất kỳ optimized hardware equation nào.

---

## 10. $N_{SF}$ và nhánh intra-slot hopping

### 10.1. Table 6.3.2.4.1-1: Số data symbols $N_{SF}$ cho PUCCH Format 1

Bảng dưới đây là **normative** từ TS 38.211 V15.10.0, Table 6.3.2.4.1-1.

| $N_{symb}^{PUCCH,1}$ | No intra-slot hopping ($m'=0$) | Intra-slot hopping ($m'=0$) | Intra-slot hopping ($m'=1$) |
| ------------------- | ------------------------------ | --------------------------- | --------------------------- |
| 4                   | 2                              | 1                           | 1                           |
| 5                   | 2                              | 1                           | 1                           |
| 6                   | 3                              | 1                           | 2                           |
| 7                   | 3                              | 1                           | 2                           |
| 8                   | 4                              | 2                           | 2                           |
| 9                   | 4                              | 2                           | 2                           |
| 10                  | 5                              | 2                           | 3                           |
| 11                  | 5                              | 2                           | 3                           |
| 12                  | 6                              | 3                           | 3                           |
| 13                  | 6                              | 3                           | 3                           |
| 14                  | 7                              | 3                           | 4                           |

### 10.2. Ví dụ với $N_{symb} = 14$

| **Trạng thái** | `m'` | `N_SF,m'` |
| --------------- | ---- | --------- |
| Không hopping   | 0    | 7         |
| Có hopping      | 0    | 3         |
| Có hopping      | 1    | 4         |

Do đó khi $N_{symb}=14$:

- no hopping → tổng 84 data RE;
- hopping → hop 0 có 36 data RE và hop 1 có 48 data RE.

Đây là lý do `intra_slot_hopping` phải configurable trong architecture.

### 10.3. Bảng tổng hợp số data RE

| $N_{symb}$ | No hopping (data RE) | Hopping hop0 (data RE) | Hopping hop1 (data RE) | Hopping tổng |
| -------- | -------------------- | ---------------------- | ---------------------- | ------------ |
| 4        | 24                   | 12                     | 12                     | 24           |
| 5        | 24                   | 12                     | 12                     | 24           |
| 6        | 36                   | 12                     | 24                     | 36           |
| 7        | 36                   | 12                     | 24                     | 36           |
| 8        | 48                   | 24                     | 24                     | 48           |
| 9        | 48                   | 24                     | 24                     | 48           |
| 10       | 60                   | 24                     | 36                     | 60           |
| 11       | 60                   | 24                     | 36                     | 60           |
| 12       | 72                   | 36                     | 36                     | 72           |
| 13       | 72                   | 36                     | 36                     | 72           |
| 14       | 84                   | 36                     | 48                     | 84           |

---

## 11. OCC chuẩn 3GPP

### 11.1. Công thức OCC

TS 38.211 Table 6.3.2.4.1-2 định nghĩa orthogonal sequence:

$$
w_i(m)=e^{j\frac{2\pi}{N_{\mathrm{SF},m'}^{\mathrm{PUCCH},1}}\phi_i(m)}
$$

trong đó $φ_{i}(m)$ được cho bởi bảng dưới đây.

Receiver phải dùng:

$$
w_i^*(m)
$$

để despread.

### 11.2. Table 6.3.2.4.1-2: Giá trị $φ_{i}(m)$ cho OCC

Bảng dưới đây là **normative** từ TS 38.211 V15.10.0, Table 6.3.2.4.1-2.

**$N_{SF} = 1$:**

| $i$ | $φ_{i}(m)$ |
| --- | -------- |
| 0   | [0]      |

**$N_{SF} = 2$:**

| $i$ | $φ_{i}(0)$ | $φ_{i}(1)$ |
| --- | -------- | -------- |
| 0   | 0        | 0        |
| 1   | 0        | 1        |

**$N_{SF} = 3$:**

| $i$ | $φ_{i}(0)$ | $φ_{i}(1)$ | $φ_{i}(2)$ |
| --- | -------- | -------- | -------- |
| 0   | 0        | 0        | 0        |
| 1   | 0        | 1        | 2        |
| 2   | 0        | 2        | 1        |

**$N_{SF} = 4$:**

| $i$ | $φ_{i}(0)$ | $φ_{i}(1)$ | $φ_{i}(2)$ | $φ_{i}(3)$ |
| --- | -------- | -------- | -------- | -------- |
| 0   | 0        | 0        | 0        | 0        |
| 1   | 0        | 2        | 0        | 2        |
| 2   | 0        | 0        | 2        | 2        |
| 3   | 0        | 2        | 2        | 0        |

**$N_{SF} = 5$:**

| $i$ | $φ_{i}(0)$ | $φ_{i}(1)$ | $φ_{i}(2)$ | $φ_{i}(3)$ | $φ_{i}(4)$ |
| --- | -------- | -------- | -------- | -------- | -------- |
| 0   | 0        | 0        | 0        | 0        | 0        |
| 1   | 0        | 1        | 2        | 3        | 4        |
| 2   | 0        | 2        | 4        | 1        | 3        |
| 3   | 0        | 3        | 1        | 4        | 2        |
| 4   | 0        | 4        | 3        | 2        | 1        |

**$N_{SF} = 6$:**

| $i$ | $φ_{i}(0)$ | $φ_{i}(1)$ | $φ_{i}(2)$ | $φ_{i}(3)$ | $φ_{i}(4)$ | $φ_{i}(5)$ |
| --- | -------- | -------- | -------- | -------- | -------- | -------- |
| 0   | 0        | 0        | 0        | 0        | 0        | 0        |
| 1   | 0        | 1        | 2        | 3        | 4        | 5        |
| 2   | 0        | 2        | 4        | 0        | 2        | 4        |
| 3   | 0        | 3        | 0        | 3        | 0        | 3        |
| 4   | 0        | 4        | 2        | 0        | 4        | 2        |
| 5   | 0        | 5        | 4        | 3        | 2        | 1        |

**$N_{SF} = 7$:**

| $i$ | $φ_{i}(0)$ | $φ_{i}(1)$ | $φ_{i}(2)$ | $φ_{i}(3)$ | $φ_{i}(4)$ | $φ_{i}(5)$ | $φ_{i}(6)$ |
| --- | -------- | -------- | -------- | -------- | -------- | -------- | -------- |
| 0   | 0        | 0        | 0        | 0        | 0        | 0        | 0        |
| 1   | 0        | 1        | 2        | 3        | 4        | 5        | 6        |
| 2   | 0        | 2        | 4        | 6        | 1        | 3        | 5        |
| 3   | 0        | 3        | 6        | 2        | 5        | 1        | 4        |
| 4   | 0        | 4        | 1        | 5        | 2        | 6        | 3        |
| 5   | 0        | 5        | 3        | 1        | 6        | 4        | 2        |
| 6   | 0        | 6        | 5        | 4        | 3        | 2        | 1        |

### 11.3. OCC phase code 16-bit cho RTL

Từ $φ_{i}(m)$, OCC phase code 16-bit được tính:

$$
\text{occ\_phase\_code} = \text{phase\_code}\left(\frac{2\pi}{N_{\mathrm{SF}}}\phi_i(m)\right) = \left\lfloor\frac{\phi_i(m)}{N_{\mathrm{SF}}} \cdot 65536\right\rfloor \bmod 65536
$$

Bảng phase code cho các $N_{SF}$ thường dùng:

| $N_{SF}$ | Phase quantum $2π/N_{SF}$ | Phase code quantum |
| ------ | ----------------------- | ------------------ |
| 1      | $2π$                    | 0 (= 65536)       |
| 2      | $π$                     | 32768 (`0x8000`)   |
| 3      | $2π/3$                  | 21845 (`0x5555`)   |
| 4      | $π/2$                   | 16384 (`0x4000`)   |
| 5      | $2π/5$                  | 13107 (`0x3333`)   |
| 6      | $π/3$                   | 10923 (`0x2AAB`)   |
| 7      | $2π/7$                  | 9362  (`0x2492`)   |

**Lưu ý:** Với $N_{SF} = 4$, OCC chỉ cần sign/swap logic (0°, 90°, 180°, 270°) — không cần CORDIC hay multiplier. Với $N_{SF} = 2$, chỉ cần negate.

### 11.4. Giá trị OCC dạng complex cho baseline verification

#### N\_SF = 7 (baseline no-hopping, N\_symb=14)

Đặt $ω = exp(j·2π/7)$:

| $i$ | $w_{i}(0)$ | $w_{i}(1)$ | $w_{i}(2)$ | $w_{i}(3)$ | $w_{i}(4)$ | $w_{i}(5)$ | $w_{i}(6)$ |
| --- | --------- | --------- | --------- | --------- | --------- | --------- | --------- |
| 0   | 1         | 1         | 1         | 1         | 1         | 1         | 1         |
| 1   | 1         | ω         | ω²        | ω³        | ω⁴        | ω⁵        | ω⁶        |
| 2   | 1         | ω²        | ω⁴        | ω⁶        | ω         | ω³        | ω⁵        |
| 3   | 1         | ω³        | ω⁶        | ω²        | ω⁵        | ω         | ω⁴        |
| 4   | 1         | ω⁴        | ω         | ω⁵        | ω²        | ω⁶        | ω³        |
| 5   | 1         | ω⁵        | ω³        | ω         | ω⁶        | ω⁴        | ω²        |
| 6   | 1         | ω⁶        | ω⁵        | ω⁴        | ω³        | ω²        | ω         |

#### N\_SF = 4 (hopping, hop 1 với N\_symb=14)

| $i$ | $w_{i}(0)$ | $w_{i}(1)$ | $w_{i}(2)$ | $w_{i}(3)$ |
| --- | --------- | --------- | --------- | --------- |
| 0   | 1         | 1         | 1         | 1         |
| 1   | 1         | +j        | −1        | −j        |
| 2   | 1         | −1        | 1         | −1        |
| 3   | 1         | −j        | −1        | +j        |

#### N\_SF = 3 (hopping, hop 0 với N\_symb=14)

Đặt $γ = exp(j·2π/3)$:

| $i$ | $w_{i}(0)$ | $w_{i}(1)$ | $w_{i}(2)$ |
| --- | --------- | --------- | --------- |
| 0   | 1         | 1         | 1         |
| 1   | 1         | γ         | γ²        |
| 2   | 1         | γ²        | γ         |

#### N\_SF = 2

| $i$ | $w_{i}(0)$ | $w_{i}(1)$ |
| --- | --------- | --------- |
| 0   | 1         | 1         |
| 1   | 1         | −1        |

**Quan trọng:** Giá trị OCC trên đây phải được verify với TS 38.211 V15.10.0 Table 6.3.2.4.1-2. Trong RTL, $w_{i}(m)$ được implement bằng phase code 16-bit và có thể dùng sign/swap logic thay vì complex multiplier tổng quát.

### 11.5. DM-RS pattern cho PUCCH Format 1

DM-RS của PUCCH Format 1 được định nghĩa trong TS 38.211 §6.4.1.3.1.

Các DM-RS symbol **không được đưa vào accumulation của UCI data**. Resource extractor phải loại chúng trước khi cấp $Y_{hat}$ cho algorithm core.

#### Table 6.4.1.3.1.1-1: Số DM-RS symbols $N_{SF}$ cho DM-RS PUCCH Format 1

Bảng dưới đây là **normative** từ TS 38.211 V15.10.0, Table 6.4.1.3.1.1-1.

| $N_{symb}^{PUCCH,1}$ | No intra-slot hopping ($m'=0$) | Intra-slot hopping ($m'=0$) | Intra-slot hopping ($m'=1$) |
| ------------------- | ------------------------------ | --------------------------- | --------------------------- |
| 4                   | 2                              | 1                           | 1                           |
| 5                   | 3                              | 1                           | 2                           |
| 6                   | 3                              | 2                           | 1                           |
| 7                   | 4                              | 2                           | 2                           |
| 8                   | 4                              | 2                           | 2                           |
| 9                   | 5                              | 2                           | 3                           |
| 10                  | 5                              | 3                           | 2                           |
| 11                  | 6                              | 3                           | 3                           |
| 12                  | 6                              | 3                           | 3                           |
| 13                  | 7                              | 3                           | 4                           |
| 14                  | 7                              | 4                           | 3                           |

**Quan trọng:** DM-RS N_SF (**Table 6.4.1.3.1.1-1**) **khác** data N_SF (**Table 6.3.2.4.1-1**). Tổng hai bảng phải bằng $N_{symb}$:

$$
N_{\mathrm{SF,data}} + N_{\mathrm{SF,DMRS}} = N_{\mathrm{symb}}^{\mathrm{PUCCH,1}}
$$

Ví dụ kiểm tra cho $N_{symb} = 14$, no hopping:
- Data: $N_{SF} = 7$, DM-RS: $N_{SF} = 7$ → Tổng = 14. ✓

#### Bảng tổng hợp: phân bố data/DMRS

| $N_{symb}$ | No hop: Data | No hop: DMRS | Hop m'=0: Data | Hop m'=0: DMRS | Hop m'=1: Data | Hop m'=1: DMRS |
| -------- | ------------ | ------------ | -------------- | -------------- | -------------- | -------------- |
| 4        | 2            | 2            | 1              | 1              | 1              | 1              |
| 5        | 2            | 3            | 1              | 1              | 1              | 2              |
| 6        | 3            | 3            | 1              | 2              | 2              | 1              |
| 7        | 3            | 4            | 1              | 2              | 2              | 2              |
| 8        | 4            | 4            | 2              | 2              | 2              | 2              |
| 9        | 4            | 5            | 2              | 2              | 2              | 3              |
| 10       | 5            | 5            | 2              | 3              | 3              | 2              |
| 11       | 5            | 6            | 2              | 3              | 3              | 3              |
| 12       | 6            | 6            | 3              | 3              | 3              | 3              |
| 13       | 6            | 7            | 3              | 3              | 3              | 4              |
| 14       | 7            | 7            | 3              | 4              | 4              | 3              |

### 11.6. DM-RS và Data symbol mapping trong slot

Với PUCCH Format 1, DM-RS và data symbol xen kẽ nhau. OFDM symbol trong PUCCH được đánh số $l = 0, 1, ..., N_{symb}−1$. Theo TS 38.211 §6.4.1.3.1.2, DM-RS được map vào các OFDM symbol chẵn ($l = 0, 2, 4, ...$):

$$
l_{\mathrm{DMRS}} = 0, 2, 4, \ldots
$$

và data symbol vào các OFDM symbol lẻ:

$$
l_{\mathrm{data}} = 1, 3, 5, \ldots
$$

Ví dụ baseline $N_{symb} = 14$, no hopping:

| $l$ (relative) | 0    | 1    | 2    | 3    | 4    | 5    | 6    | 7    | 8    | 9    | 10   | 11   | 12   | 13   |
| --------------- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| Loại            | DMRS | Data | DMRS | Data | DMRS | Data | DMRS | Data | DMRS | Data | DMRS | Data | DMRS | Data |
| $m$ (data idx)  | —    | 0    | —    | 1    | —    | 2    | —    | 3    | —    | 4    | —    | 5    | —    | 6    |

---

## 12. Tách phase của sequence

Từ:

$$
r_{u,v}^{(\alpha,\delta)}(n)=\exp\left(j\left[\frac{\pi}{4}\phi_u(n)+\alpha(n+\delta)\right]\right)
$$

đặt:

$$
\theta_{\mathrm{base}}(n)=\frac{\pi}{4}\phi_u(n)
$$

và:

$$
\theta_{\mathrm{cs}}(n)=\alpha(n+\delta)
$$

suy ra:

$$
\theta_{\mathrm{ref}}(n)=\theta_{\mathrm{base}}(n)+\theta_{\mathrm{cs}}(n)\pmod{2\pi}
$$

và:

$$
r_{u,v}^{(\alpha,\delta)}(n)=e^{j\theta_{\mathrm{ref}}(n)}
$$

Đây là **chuyển đổi toán học**, chưa phải optimization.

---

## 13. Biến phase thực sang phase code 16 bit

Chọn phase representation:

$$
2\pi\equiv 2^{16}=65536
$$

Do đó:

$$
\mathrm{phase\_code}(\theta)=\left\lfloor\frac{\theta}{2\pi}2^{16}\right\rfloor\bmod 2^{16}
$$

Một số giá trị:

| **Phase** | **Code**   |
| --------- | ---------- |
| `0`       | `0x0000`   |
| $π/4$     | `0x2000`   |
| $π/2$     | `0x4000`   |
| $3π/4$    | `0x6000`   |
| $π$       | `0x8000`   |
| $−3π/4$   | `0xA000`   |
| $−π/2$    | `0xC000`   |
| $−π/4$    | `0xE000`   |
| $2π$      | `0x0000`   |

Đây là representation của phase, không phải signed Q-format của I/Q.

### 13.1. Xử lý phase\_code cho góc âm

`floor()` trên số âm khác `mod` trên số dương. Để tránh ambiguity giữa golden model (Python) và RTL:

**Định nghĩa thống nhất:**

$$
\mathrm{phase\_code}(\theta)=\left(\left\lfloor\frac{\theta}{2\pi}2^{16}\right\rfloor\right)\mathbin{\&}\,\mathrm{0xFFFF}
$$

Trong đó `& 0xFFFF` là bitwise AND với mask 16 bit, tương đương `mod 2^16` cho cả số âm (two's complement).

**Ví dụ:**

- $θ = -π/2$ → $floor(-0.25 × 65536) = floor(-16384) = -16384$ → $-16384 & 0xFFFF = 0xC000$ (tức 3π/2). ✓
- $θ = -α$ với $α > 0$ → phase code = $(65536 - phase_{code}(α)) & 0xFFFF$.

**RTL contract:** Phase accumulator dùng wraparound tự nhiên của 16-bit unsigned. Cộng/trừ phase code thực hiện modulo 2^16 bằng bit-width của thanh ghi.

---

## 14. Chuyển phần cyclic shift thành phase accumulator

Ta có:

$$
\theta_{\mathrm{cs}}(n)=\alpha(n+\delta)
$$

nên:

$$
\theta_{\mathrm{cs}}(n+1)-\theta_{\mathrm{cs}}(n)=\alpha
$$

Do đó trong phase code:

$$
\Delta\theta_{\mathrm{cs,code}}=\mathrm{phase\_code}(\alpha)
$$

và phase accumulator có thể thực hiện:

$$
\theta_{\mathrm{cs,code}}[n+1]=\mathrm{wrap}\left(\theta_{\mathrm{cs,code}}[n]+\Delta\theta_{\mathrm{cs,code}}\right)
$$

Initial value:

$$
\theta_{\mathrm{cs,code}}[0]=\mathrm{phase\_code}(\alpha\delta)
$$

nếu giữ trực tiếp biểu thức $alpha(n+delta)$.

Trường hợp $δ=0$ (baseline PUCCH F1), accumulator bắt đầu từ 0.

### 14.1. Tính $Δθ_{cs,code}$ từ $alpha$

Từ §7:

$$
\alpha_l = \frac{2\pi}{12}\left((m_0 + n_{\mathrm{cs}}) \bmod 12\right)
$$

Đặt $k = (m_{0} + n_{\mathrm{cs}}) \bmod 12$, thì:

$$
\Delta\theta_{\mathrm{cs,code}} = \mathrm{phase\_code}\!\left(\frac{2\pi k}{12}\right) = \left\lfloor\frac{k}{12} \cdot 65536\right\rfloor
$$

Bảng tra $Δθ_{cs,code}$ theo $k$:

| $k$ | $Δθ_{cs,code}$ (decimal) | $Δθ_{cs,code}$ (hex) |
| --- | ---------------------- | ------------------- |
| 0   | 0                      | `0x0000`            |
| 1   | 5461                   | `0x1555`            |
| 2   | 10922                  | `0x2AAA`            |
| 3   | 16384                  | `0x4000`            |
| 4   | 21845                  | `0x5555`            |
| 5   | 27306                  | `0x6AAA`            |
| 6   | 32768                  | `0x8000`            |
| 7   | 38229                  | `0x9555`            |
| 8   | 43690                  | `0xAAAA`            |
| 9   | 49152                  | `0xC000`            |
| 10  | 54613                  | `0xD555`            |
| 11  | 60074                  | `0xEAAA`            |

---

## 15. Tách base phase và cyclic-shift phase

Đối với mỗi $n$:

### Bước 1 — đọc bảng 3GPP

$$
\phi_u(n)
$$

### Bước 2 — chuyển thành phase

$$
\theta_{\mathrm{base}}(n)=\frac{\pi}{4}\phi_u(n)
$$

### Bước 3 — chuyển thành phase code

$$
\mathrm{base\_phase\_code}(n)=\mathrm{phase\_code}\left(\frac{\pi}{4}\phi_u(n)\right)
$$

### Bước 4 — lấy phase accumulator

$$
\mathrm{cs\_phase\_code}(n)=\theta_{\mathrm{cs,code}}[n]
$$

### Bước 5 — cộng phase

$$
\theta_{\mathrm{ref,code}}(n)=\mathrm{wrap}\left(\mathrm{base\_phase\_code}(n)+\mathrm{cs\_phase\_code}(n)\right)
$$

### Bước 6 — cập nhật accumulator

$$
\mathrm{cs\_phase\_code}[n+1]=\mathrm{wrap}\left(\mathrm{cs\_phase\_code}[n]+\Delta\theta_{\mathrm{cs,code}}\right)
$$

Kết quả chính xác về mặt biểu diễn pha là:

$$
\theta_{\mathrm{ref,code}}(n)\leftrightarrow r_{u,v}^{(\alpha,\delta)}(n)
$$

trong sai số lượng tử phase.

---

## 16. Tại sao không cần sinh $exp(jθ)$ trong sequence generator

Cách trực tiếp:

$$
r(n)=e^{j\theta(n)}
$$

sẽ yêu cầu tạo sin/cos hoặc ROM cho từng phase.

Thiết kế mới chỉ xuất:

$$
\theta_{\mathrm{ref,code}}(n)
$$

Sau đó CORDIC nhận:

$$
-\theta_{\mathrm{ref}}
$$

và quay trực tiếp $Y_{hat}$.

Như vậy complex reference sequence không nhất thiết phải tồn tại dưới dạng bộ nhớ $I+jQ$.

---

## 17. Tương đương toán học giữa direct sequence và phase implementation

Direct reference:

$$
r(n)=e^{j\theta(n)}
$$

Receiver direct multiplication:

$$
Y(n)r^*(n)
$$

với:

$$
r^*(n)=e^{-j\theta(n)}
$$

Phase-domain CORDIC:

$$
Y'(n)=\mathrm{CORDIC}\left(Y(n),-\theta(n)\right)
$$

và lý tưởng:

$$
Y'(n)=Y(n)e^{-j\theta(n)}
$$

Vì vậy CORDIC là realization phần cứng của phép derotation chứ không thay đổi algorithm.

---

## 18. Mô hình received signal

Sau channel estimation/equalization:

$$
\hat{Y}(m,n)
$$

Nếu bỏ qua residual channel error và noise:

$$
\hat{Y}(m,n)\approx d(0)w_i(m)r_{u,v}^{(\alpha,\delta)}(n)
$$

**Lưu ý:** $m$ ở đây là **spreading symbol index** (0..N\_SF-1), không phải slot index hay hop index `m'`.

Receiver áp dụng conjugate reference:

$$
Y'(m,n)=\hat{Y}(m,n)\left[r_{u,v}^{(\alpha,\delta)}(n)\right]^*
$$

suy ra lý tưởng:

$$
Y'(m,n)\approx d(0)w_i(m)
$$

Tiếp theo despread:

$$
Y''(m,n)=Y'(m,n)w_i^*(m)
$$

nên:

$$
Y''(m,n)\approx d(0)
$$

Tất cả data RE sau đó có cùng symbol direction và có thể cộng coherently.

---

## 19. Statistic $Z$

Receiver statistic:

$$
Z=\sum_{m=0}^{N_{\mathrm{SF}}-1}\sum_{n=0}^{11}\hat{Y}(m,n)\left[r_{u,v}^{(\alpha,\delta)}(n)\right]^*w_i^*(m)
$$

Trong phase-domain:

$$
Z=\sum_m\sum_n\hat{Y}(m,n)e^{-j\theta_{\mathrm{ref}}(m,n)}w_i^*(m)
$$

Đây là matched-filter / coherent-combining statistic.

---

## 20. Trường hợp baseline $N_{symb}=14$, không hopping

Theo Table 6.3.2.4.1-1:

$$
N_{\mathrm{SF}}=7
$$

nên:

$$
N_{\mathrm{RE,data}}=7\times12=84
$$

84 RE này là input cho UCI detector. Các RE được DM-RS sử dụng không được đưa vào accumulation của UCI data.

---

## 21. Mô hình thống kê

Sau coherent combining:

$$
Z=Ad+n
$$

trong đó:

- $d$: candidate BPSK/QPSK symbol.
- $A$: effective non-negative amplitude sau equalization/combining.
- $n$: circular complex Gaussian noise.

Với các candidate có cùng năng lượng, maximum-likelihood decision có thể đưa về correlation metric:

$$
M(d)=\Re\left\{d^*Z\right\}
$$

và:

$$
\hat{d}=\arg\max_{d\in\mathcal{D}}M(d)
$$

---

## 22. MLE cho một bit

Candidate set BPSK theo §3.1:

$$
\mathcal{D}_1=\left\{\frac{1+j}{\sqrt{2}},-\frac{1+j}{\sqrt{2}}\right\}
$$

Metric:

$$
M(d_0)=\Re\left\{d_0^*Z\right\}
$$

Với $d_{0} = (1+j)/√2$:

$$
M(+)=\Re\left\{\frac{1-j}{\sqrt{2}}Z\right\}=\frac{1}{\sqrt{2}}\left(\Re\{Z\}+\Im\{Z\}\right)
$$

Với $d_{0} = -(1+j)/√2$:

$$
M(-)=-\frac{1}{\sqrt{2}}\left(\Re\{Z\}+\Im\{Z\}\right)
$$

Do đó:

$$
\Re\{Z\}+\Im\{Z\}\ge0\;\Rightarrow\;\hat{d}=\frac{1+j}{\sqrt{2}}\;\Rightarrow\;b(0)=0
$$

$$
\Re\{Z\}+\Im\{Z\}<0\;\Rightarrow\;\hat{d}=-\frac{1+j}{\sqrt{2}}\;\Rightarrow\;b(0)=1
$$

**Quan trọng:** Vì BPSK NR nằm trên đường chéo, decision **không phải** $Re(Z) ≥ 0$ mà là $Re(Z) + Im(Z) ≥ 0$. Đây là khác biệt so với BPSK cổ điển trên trục thực.

### 22.1. Lưu ý về biểu diễn BPSK

Nếu hardware chọn triển khai BPSK như BPSK cổ điển (trên trục thực) và dùng $d = ±1$, thì phải có bước xoay $-45°$ ($e^{-j\pi/4}$) trước khi so sánh. Đây là lựa chọn implementation, nhưng phải document rõ để tránh nhầm giữa golden model và RTL.

---

## 23. MLE cho hai bit

Candidate set QPSK theo TS 38.211 §5.1.3:

$$
\mathcal{D}_2=\{d_0,d_1,d_2,d_3\}
$$

với:

$$
d_0=\frac{1+j}{\sqrt{2}},\quad d_1=\frac{1-j}{\sqrt{2}},\quad d_2=\frac{-1+j}{\sqrt{2}},\quad d_3=\frac{-1-j}{\sqrt{2}}
$$

tương ứng $b(0)b(1) = 00, 01, 10, 11$.

Tính:

$$
M_k=\Re\left\{d_k^*Z\right\}
$$

sau đó:

$$
\hat{d}=d_{\arg\max_k M_k}
$$

### 23.1. QPSK bit mapping table

| $b(0)$ | $b(1)$ | `d_k`       | `M_k`                | Decision region        |
| ------ | ------ | ----------- | -------------------- | ---------------------- |
| 0      | 0      | $(1+j)/√2$  | $(Re{Z}+Im{Z})/√2$  | $Re{Z}+Im{Z}$ max     |
| 0      | 1      | $(1−j)/√2$  | $(Re{Z}−Im{Z})/√2$  | $Re{Z}−Im{Z}$ max     |
| 1      | 0      | $(−1+j)/√2$ | $(−Re{Z}+Im{Z})/√2$ | $−Re{Z}+Im{Z}$ max    |
| 1      | 1      | $(−1−j)/√2$ | $(−Re{Z}−Im{Z})/√2$ | $−Re{Z}−Im{Z}$ max    |

Equivalent sign-decision form:

- $Re{Z} ≥ 0$ và $Im{Z} ≥ 0$ → $b(0)b(1) = 00$
- $Re{Z} ≥ 0$ và $Im{Z} < 0$ → $b(0)b(1) = 01$
- $Re{Z} < 0$ và $Im{Z} ≥ 0$ → $b(0)b(1) = 10$
- $Re{Z} < 0$ và $Im{Z} < 0$ → $b(0)b(1) = 11$

Tóm gọn:

$$
b(0) = \begin{cases}0 & \text{if } \Re\{Z\} \ge 0\\1 & \text{if } \Re\{Z\} < 0\end{cases}
$$

$$
b(1) = \begin{cases}0 & \text{if } \Im\{Z\} \ge 0\\1 & \text{if } \Im\{Z\} < 0\end{cases}
$$

**Normalization:** Hệ số $1/√2$ ảnh hưởng tới giá trị tuyệt đối của `metric_best`. Nếu muốn `metric_best` phản ánh $|Z|$ trực tiếp, có thể bỏ normalization trong metric và chỉ áp dụng khi cần báo cáo. Cần document rõ convention.

---

## 24. Hardware realization của MLE

### Generic implementation

```
Z
 ↓
4 candidate metrics
 ↓
max comparator tree
 ↓
winning candidate
 ↓
bit mapping
```

### Baseline optimized implementation

```
Z_I --------------------→ sign / threshold
Z_Q --------------------→ sign / threshold
            ↓
        bit mapping
```

Không được tối ưu trước khi golden model xác nhận bit mapping.

---

## 25. Phase representation

Chọn:

$$
\mathrm{PHASE\_FULL}=65536
$$

và:

$$
\mathrm{phase\_code}(\theta)=\left\lfloor\frac{\theta}{2\pi}65536\right\rfloor\bmod 65536
$$

Phase accumulator thực hiện modulo bằng wraparound của 16 bit (xem §13.1 cho góc âm).

---

## 26. CORDIC contract

Input:

```
in_valid
in_i[15:0]
in_q[15:0]
in_phase[15:0]
in_last
```

Trong đó:

$$
\mathrm{input\_phase}=-\theta_{\mathrm{ref}}
$$

Output:

```
out_valid
out_i[17:0]
out_q[17:0]
out_last
```

CORDIC baseline:

- rotation mode;
- 16 iterations;
- 16-bit phase;
- 16-bit signed Q1.15 input;
- 18-bit signed output;
- fixed latency;
- documented gain compensation;
- documented rounding/saturation.

### 26.1. CORDIC gain và compensation policy

CORDIC rotation mode với `N` iterations có gain:

$$
A_N=\prod_{i=0}^{N-1}\sqrt{1+2^{-2i}}
$$

Với $N = 16$:

$$
A_{16}\approx1.6467602581
$$

(tương đương ~4.33 dB).

**Ba lựa chọn compensation:**

| **Lựa chọn**             | **Mô tả**                                      | **Ưu điểm**                | **Nhược điểm**                |
| ------------------------ | ----------------------------------------------- | -------------------------- | ----------------------------- |
| (a) Scale trong CORDIC   | Nhân output với $1/A$ ngay trong CORDIC         | $Z$ giữ scale khớp $Y_{hat}$ | Thêm multiplier               |
| (b) Scale sau accumulator | Nhân $Z$ với $1/A$ trước MLE                   | Multiplier chỉ 1 lần       | Cần biết $A$ chính xác        |
| (c) Không scale          | Giữ nguyên $A$, MLE tự scale threshold          | Tiết kiệm hardware         | `metric_best` bị lệch $A$ lần |

**Baseline khuyến nghị:** (c) — không scale, vì MLE là so sánh tương đối giữa các candidate. `metric_best` được document là "có scale $A$".

**RTL contract:**

- $A_{16} = 1.6467602581$ (documented constant).
- Nếu chọn (a) hoặc (b): hệ số scale là $1/A_{16} ≈ 0.607252935$.
- Width sau scale: cần verify 18-bit đủ headroom cho $1/A × max_{input}$.

**Saturation:** output CORDIC phải saturate về $±2^17−1$ nếu vượt range, không wrap-around.

---

## 27. Accumulator contract

Input:

```
sample_i[17:0]
sample_q[17:0]
sample_valid
sample_last
occ_phase[15:0]
```

Accumulator thực hiện:

$$
Z_I+jZ_Q=\sum x_{\mathrm{occ}}(m,n)
$$

trong đó `x_occ` là sample sau CORDIC và OCC conjugation.

Baseline width:

```
26-bit signed I
26-bit signed Q
```

### 27.1. Accumulator overflow policy

**Worst-case analysis:**

- Số RE tối đa: `N_RE,data` ≤ `N_symb × 12` (với $N_{symb} = 14$ → 168; nhưng do DM-RS chiếm một nửa, baseline = 84).
- CORDIC output: 18-bit signed ($±2^17$).
- Sau CORDIC gain (nếu không bù): $1.6468 × 2^17 ≈ 2^17.72$.
- Tổng worst-case: $84 × 2^17.72 ≈ 2^24.1$.

Với 26-bit signed accumulator ($±2^25$), còn headroom ~$2^0.9$ ≈ 1.87×. **Đủ cho baseline**, nhưng cần lưu ý:

| **Trường hợp**                | **Số RE**  | **Worst-case magnitude** | **26-bit đủ?** |
| ----------------------------- | ---------- | ------------------------ | -------------- |
| Baseline (N\_symb=14, no hop) | 84         | ~2^24.1                  | Có             |
| N\_symb=14, hop               | 36 hoặc 48 | ~2^23.4                  | Có             |
| N\_symb nhỏ hơn               | ≤ 84       | ≤ 2^24.1                 | Có             |

**Policy:** Accumulator **saturate** khi overflow (không wrap-around). Flag `error` được set nếu saturation xảy ra.

**Ghi chú:** Nếu system architecture hỗ trợ multi-antenna combining hoặc multiple UE contexts, cần review lại width.

---

## 28. Tín hiệu trung gian bắt buộc phải debug được

```
n_ID
f_gh
f_ss
u
v
n_cs
m0
alpha
phi_u(n)
base_phase_code
cs_phase_code
theta_ref_code
w_i(m)
Y_hat_I
Y_hat_Q
CORDIC_out_I
CORDIC_out_Q
OCC_out_I
OCC_out_Q
acc_I
acc_Q
Z_I
Z_Q
MLE_metric[k]
best_candidate
uci_bits
```

Mục tiêu là khi `final_bits` sai, có thể xác định lỗi thuộc sequence generation, extraction, phase, CORDIC, OCC, accumulation hay MLE.

---

## 29. Pseudocode chuẩn tham chiếu

```
INPUT:
    PUCCH configuration
    frame_idx
    slot_idx
    Y_hat(m,n)

STEP 1:
    Determine pucch_nsymb
    Determine hopping state
    Determine m'
    Determine N_SF,m'

STEP 2:
    Derive n_ID
    Derive f_gh, f_ss, u, v

STEP 3:
    For each PUCCH data symbol:
        Calculate n_cs
        Calculate alpha

STEP 4:
    For n = 0..11:
        Read phi_u(n)
        Calculate base phase
        Calculate r_uv^(alpha,delta)(n)

STEP 5:
    Construct 3GPP z using:
        y(n) = d(0) r_uv^(alpha,delta)(n)
        z(index) = w_i(m) y(n)

STEP 6:
    Receiver:
        take equalized Y_hat
        generate theta_ref using phase accumulator
        CORDIC rotate by -theta_ref
        multiply by w_i*(m) using OCC sign/swap logic
        accumulate coherently into Z

STEP 7:
    MLE:
        if 1 bit:
            compare BPSK candidates
        if 2 bits:
            compare QPSK candidates

OUTPUT:
    UCI bits
    best metric
    optional second-best metric
    result_valid
```

---

## 30. Chuỗi kiểm chứng tương đương 3GPP ↔ Phase Accumulator

Verification phải chứng minh:

$$
Z_{\mathrm{direct}}=Z_{\mathrm{phase\_accumulator}}
$$

trong sai số lượng tử đã định nghĩa.

Cụ thể:

```
3GPP phi_u(n)
      ↓
Direct complex exp(j*theta)
      ↓
Reference r(n)
      ↓
matched filter
      ↓
Z_direct
```

phải khớp với:

```
3GPP phi_u(n)
      ↓
base phase code
      +
cyclic-shift phase accumulator
      ↓
theta_ref_code
      ↓
CORDIC derotation
      ↓
coherent accumulation
      ↓
Z_phase_accumulator
```

Sai khác được phân loại:

1. sai khác do phase quantization;
2. sai khác do CORDIC approximation;
3. sai khác do fixed-point accumulation;
4. sai khác do rounding/saturation.

Không được chấp nhận sai khác do sequence equation.

---

## 31. Algorithm invariants

### Invariant 1 — 3GPP reference sequence

$$
\theta_{\mathrm{ref}}(n)\equiv\angle r_{u,v}^{(\alpha,\delta)}(n)\pmod{2\pi}
$$

### Invariant 2 — conjugate

Receiver luôn dùng:

$$
\left[r_{u,v}^{(\alpha,\delta)}(n)\right]^*
$$

và:

$$
w_i^*(m)
$$

### Invariant 3 — sample alignment

```
Y_hat(m,n)
phase(m,n)
OCC(m)
```

phải cùng xác định một RE.

### Invariant 4 — coherent sum

Sau compensation lý tưởng:

$$
Y''(m,n)\approx d(0)
$$

### Invariant 5 — MLE

Candidate thắng phải là candidate có metric lớn nhất.

---

## 32. Kết quả baseline cần đạt

Với $N_{symb}=14$, no hopping, one PRB, one Rx:

$$
N_{\mathrm{SF}}=7
$$

và:

$$
N_{\mathrm{RE,data}}=84
$$

### 32.1. Amplitude normalization convention

**Input contract:** $Y_{hat}$ được cung cấp bởi equalizer với convention:

- Mỗi data RE có amplitude kỳ vọng $|Y_{hat}| ≈ 1$ sau equalization.
- Channel estimation đã compensate fading (ZF/MMSE).
- Residual error và noise được mô hình hóa riêng.

**Baseline sanity check (noise-free, ideal equalization):**

Với BPSK ($d = ±(1+j)/√2$):

$$
Z\approx84d=\pm84\cdot\frac{1+j}{\sqrt{2}}
$$

Nên:

- $Re{Z} + Im{Z} ≈ ±84·√2$ (tùy dấu $d$).
- $|Z| ≈ 84$.

Với QPSK ($d = d_{k}$):

$$
Z\approx84d_k
$$

Nên $|Z| ≈ 84$.

**Ghi chú về normalization:** Nếu muốn $Z ≈ 84·d_{unit}$ với `d_unit` có $|d_{unit}| = 1$, thì convention $Y_{hat}$ amplitude = 1 là đúng. Nếu $Y_{hat}$ amplitude = $1/√2$ (tức đã normalize theo $d(0)$ có magnitude $1/√2$), thì $Z ≈ 84/√2 ·$`d_unit_normalized`. Cần khóa convention giữa golden model và RTL.

**Quan trọng:** Cần phân biệt:

- $d(0)$ theo TS 38.211 có magnitude `1` (do normalization $1/√2$ trên cả I và Q, tổng magnitude = 1).
- $Y_{hat}$ amplitude convention do equalizer quyết định.

Đây là sanity check đầu tiên trước khi kiểm tra AWGN và channel.

---

---
