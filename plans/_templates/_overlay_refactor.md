<!-- OVERLAY: REFACTOR. Ap len _PLAN_CORE.md. KHONG sua file nay. -->

# Overlay: `REFACTOR`

## Khi nào dùng

Code đổi nhưng **behavior phải giữ nguyên**: cleanup RTL, tách module, tách `always` block lớn,
restructure FSM, parameterize, đổi cấu trúc vì area/timing nhưng giữ chức năng.

Dấu hiệu: "refactor…", "tách…", "gọn lại…", "tham số hoá…", "đổi cấu trúc nhưng giữ chức năng".
Nếu có thêm chức năng mới → đó là `IMPLEMENT` (hoặc tách thành 2 plan).

**Acceptance chính không phải "compile được" mà là `behavior mới == behavior cũ`.**

## §1 Metadata — giá trị mặc định

| Field | Value |
|---|---|
| Task type | `REFACTOR` |
| Flow phase(s) | Stage 06 RTL *(dùng `change-impact-analyzer` trước để khoanh vùng; regression/QoR là MANUAL)* |
| Skills / Agents | `change-impact-analyzer` (khoanh vùng ảnh hưởng) · `rtl-implementer` (áp thay đổi) · regression + QoR comparison: **không có skill, MANUAL** |
| Related artifacts | `rtl/<m>.v`, microarchitecture proposal, `tb/<ip>/tc_list.md`, `synth/<m>/` |

## Đổi tên trong core

- §2 `Objective & Requirements` → **`Objective & Behavior Contract`**
- §2.5 `Requirements` → giữ, nhưng requirement ở đây là **ràng buộc bảo toàn**, không phải chức năng mới.

## Chèn thêm

### §2.9 Behavior Invariants

Tick nghĩa là **cam kết không đổi**; mục nào buộc phải đổi thì bỏ tick và giải thích ngay dưới.

- [ ] Functional behavior unchanged
- [ ] Interface unchanged (port list, width, direction, protocol)
- [ ] Reset behavior unchanged (polarity, đồng bộ, giá trị sau reset)
- [ ] Timing intent unchanged (số pipeline stage, latency, throughput)
- [ ] CDC behavior unchanged (không thêm/bớt crossing, không đổi synchronizer)
- [ ] Synthesis intent unchanged (không sinh thêm memory/DSP/latch ngoài ý)
- [ ] Reg-map unchanged

**Ngoại lệ được phép (nếu có):** ... *(mỗi ngoại lệ phải có một dòng lý do + tôi duyệt)*

### §4.7 Before / After Architecture

| Khía cạnh | Before | After | Vì sao đổi |
|---|---|---|---|
| Phân rã block | | | |
| Số always block / process | | | |
| FSM state / encoding | | | |
| Tham số | | | |

### §6.8 Equivalence Strategy

Chứng minh **tương đương** bằng cách nào — nêu rõ giới hạn của cách đó:

| Phương pháp | Cách làm | Chứng minh được gì | Không chứng minh được gì | Status |
|---|---|---|---|---|
| Regression y nguyên tập TC cũ | chạy toàn bộ `tc_list.md` **không sửa testcase** | hành vi trên các case đã cover | case chưa cover | NOT_RUN |
| So sánh waveform tín hiệu chính | dump cùng stimulus, diff signal list | tương đương cycle-by-cycle ở các tín hiệu đó | tín hiệu nội bộ khác | NOT_RUN |
| So sánh QoR | `utilization.rpt` + `timing_summary.rpt` before/after | không xấu đi về area/timing | tính đúng đắn chức năng | NOT_RUN |

**Baseline phải chụp TRƯỚC khi refactor** (log sim + report synth của bản cũ), nếu không thì không
có gì để so sánh:

| Baseline artifact | Đường dẫn | Đã chụp? |
|---|---|---|
| sim log bản cũ | | [ ] |
| `utilization.rpt` bản cũ | | [ ] |
| `timing_summary.rpt` bản cũ | | [ ] |

## Acceptance bắt buộc thêm

- [ ] Baseline đã chụp trước khi sửa (có path)
- [ ] Toàn bộ TC cũ PASS **mà không sửa testcase nào** (có `run_summary.md`)
- [ ] Mọi invariant §2.9 còn tick, hoặc ngoại lệ đã được tôi duyệt
- [ ] Interface diff = rỗng (hoặc đã khai ở §2.6 và tôi duyệt)
- [ ] Area / timing không xấu đi so với baseline, hoặc đã giải thích
- [ ] Không phát sinh lint warning mới

## FE Flow Impact mặc định

| Stage | Required? | Reason |
|---|---:|---|
| RTL | YES | chính là task này |
| Testbench | NO | **không sửa TC** — sửa TC làm mất giá trị của regression |
| Simulation | YES | bằng chứng behavior không đổi — **MANUAL**, tôi tự chạy regression |
| Lint | YES | RTL thay đổi — **MANUAL** |
| CDC | MAYBE | YES nếu chạm tới synchronizer / clock domain — **MANUAL**, không có tool CDC |
| Synthesis | YES | phải chứng minh synthesis intent không đổi — **MANUAL** |
| Timing / Area | YES | so sánh với baseline — **MANUAL** |
| Power | MAYBE | YES nếu §2.8 có target |
| Documentation | MAYBE | YES nếu cấu trúc trong `doc/<m>.md` mô tả bị lệch |

## Cẩn trọng

- **Sửa testcase để cho pass là vi phạm** mục đích của plan này. TC fail sau refactor = refactor sai
  (hoặc TC vốn đã sai — phải chứng minh bằng plan `DEBUG` riêng — xem lưu ý "không có skill" ở
  `_overlay_debug.md` — không sửa lặng lẽ).
- Không nhân cơ hội thêm chức năng. Thấy bug trong lúc refactor → ghi ở §8.5 `Follow-up`, mở plan
  `DEBUG`, đừng fix chung.
