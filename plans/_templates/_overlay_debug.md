<!-- OVERLAY: DEBUG. Ap len _PLAN_CORE.md. KHONG sua file nay. -->

# Overlay: `DEBUG`

## Khi nào dùng

Có **hiện tượng sai cụ thể**: sim fail, assertion fail, lint violation, CDC violation, synthesis
error, STA violation, waveform bất thường, regression fail.

Dấu hiệu: "đang fail…", "sai ở cycle…", "vì sao ra X", "lỗi này là gì".

**Thứ tự bắt buộc: Observed failure → Reproduction → Evidence → Root cause → Fix → Regression.**
Chưa reproduce được thì **chưa được đề xuất fix**.

## §1 Metadata — giá trị mặc định

| Field | Value |
|---|---|
| Task type | `DEBUG` |
| Flow phase(s) | *(kèm stage bị lùi: 05/06 RTL, 09 Testbench nếu phải sửa)* |
| Skills / Agents | **KHÔNG CÓ SKILL** — bộ `rtl-design-flow-skills` không chạy tool nên không chẩn đoán được lỗi runtime. Chẩn đoán + fix hoàn toàn **MANUAL/USER-OWNED**; agent chỉ hỗ trợ đọc code/log tôi dán vào, không tự chạy sim/lint/synth và không tự suy ra kết quả. Nếu fix cần sửa RTL, bước sửa dùng `rtl-implementer` (stage 06) sau khi root cause đã rõ. |
| Failing item | `tb_<name>.sv` / `TC-00x` / lint rule / timing path |
| Failure class | `RTL_BUG` / `TC_BUG` / `DOC_AMBIGUITY` / `TOOL` / `ENV` *(agent tự phân loại dựa trên evidence tôi cung cấp, không có skill chẩn đoán riêng)* |
| Related artifacts | `issue/<ip>/issue_NNN_<tc>.md`, `issue/<ip>/debug_NNN_<tc>.md`, `run_summary.md` |

## Đổi tên trong core

- §2 `Objective & Requirements` → **`Objective & Failure Report`**
- §5 `Implementation Plan` → **`Fix Plan`**

## Chèn thêm

### §2.9 Observed failure

Trích **nguyên văn** log / message lỗi (không kể lại bằng lời), kèm đường dẫn file log.

```text
<dán log thật vào đây>
```

### §2.10 Reproduction

| Field | Value |
|---|---|
| Command | `make <tc>` trong `tb/<ip>/` |
| Seed / param | |
| Tool + version | |
| **Reproduce được?** | **CÓ** / chưa — *(chưa reproduce thì không được đề xuất fix)* |
| Deterministic? | |

### §2.11 Expected vs Actual

| Signal / Check | Time / Cycle | Expected | Actual | Nguồn |
|---|---|---|---|---|
| | | | | waveform / log line |

Cơ sở của "expected": `ddoc/<m>_req.md` / `doc/<m>.md` / §2.1. Nếu spec **mơ hồ** chứ không phải RTL
sai → đây là `DOC_AMBIGUITY`: cần tôi quyết hành vi đúng **trước** khi fix.

### §4.7 Signal trace path

```text
<tb check> ← <top signal> ← <module>.<signal> ← <module>.<signal> ← <nguồn>
```

Mỗi mắt phải có evidence ở §3, không suy diễn trắng.

### §4.8 Root cause

Nêu **chính xác** file / signal / dòng và cơ chế sai. Mẫu:

> `rtl/pulse_counter.v` tăng `count` khi `pulse_valid` cao, nhưng không gate bằng `enable`.
> Evidence: `rtl/pulse_counter.v:42`, signal `enable`, TC-004 tại cycle 137 (`sim/tc004/sim.log`).

Phải khớp `issue/<ip>/debug_NNN_*.md` nếu tôi đã tự chẩn đoán và ghi lại trước đó; khác thì giải
thích vì sao (không có skill chẩn đoán tự động trong bộ này).

### §4.9 Candidates ruled out

| Giả thuyết | Vì sao loại | Bằng chứng |
|---|---|---|

### §4.10 Blast radius

| Nơi khác | Cùng root cause? | Bằng chứng | Xử lý trong plan này? |
|---|---|---|---|
| | | | có / không — lý do |

### §6.8 Fix validation

| Bước | Điều kiện | Status |
|---|---|---|
| (a) Reproduce fail **trước** khi sửa | log tái hiện đúng §2.9 | NOT_RUN |
| (b) Đúng TC đó PASS sau fix | `[FINISH] PASS` | NOT_RUN |
| (c) Regression toàn bộ `tc_list.md` | tất cả PASS, có `run_summary.md` | NOT_RUN |
| (d) TC chặn tái phát | TC mới cover đúng root cause, hoặc `N/A — lý do` | NOT_RUN |

## Acceptance bắt buộc thêm

- [ ] Lỗi §2.9 đã reproduce được **trước** khi fix (có log)
- [ ] Root cause §4.8 có bằng chứng `file:line` + cycle, không phải suy đoán
- [ ] Fix sửa **gốc**, không che triệu chứng
- [ ] TC đang fail giờ PASS; regression full PASS (có `run_summary.md`)
- [ ] Có TC chặn tái phát, hoặc `N/A — lý do`
- [ ] `issue/<ip>/issue_NNN_*.md` đã cập nhật kết quả

## FE Flow Impact mặc định

| Stage | Required? | Reason |
|---|---:|---|
| RTL | MAYBE | YES nếu `RTL_BUG`; NO nếu `TC_BUG` |
| Testbench | MAYBE | YES nếu `TC_BUG` hoặc cần TC chặn tái phát |
| Simulation (TC fail) | YES | chứng minh đã fix — **MANUAL**, tôi tự chạy và dán log |
| Simulation (regression) | YES | chứng minh không phá cái khác — **MANUAL** |
| Lint | YES | nếu có sửa RTL — **MANUAL** |
| CDC | MAYBE | YES nếu triệu chứng liên quan clock domain — **MANUAL**, không có tool CDC |
| Synthesis / Timing | MAYBE | YES nếu lỗi là timing / synth, hoặc fix đổi logic depth — **MANUAL** |
| Documentation | MAYBE | YES nếu hành vi đúng khác doc hiện tại |

## Cấm

- Không sửa RTL khi failure class là `TC_BUG`, và ngược lại.
- Không refactor kèm, không "sửa luôn cho đẹp".
- Không đổi expected value trong testcase để cho pass.
- Agent không tự chạy tool (sim/lint/synth) và không tự bịa/suy ra kết quả tool — bộ skill này
  không có bước chạy tool, mọi evidence trong §2.9/§4.7/§4.8 phải do tôi cung cấp.
