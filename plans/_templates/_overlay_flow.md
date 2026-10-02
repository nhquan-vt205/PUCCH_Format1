<!-- OVERLAY: FLOW. Ap len _PLAN_CORE.md. KHONG sua file nay. -->

# Overlay: `FLOW`

## Khi nào dùng

Chạy / dựng / sửa **tool flow** chứ không sửa chức năng: lint, CDC, synthesis, STA, constraints,
QoR, chạy full front-end flow, dựng script regression. Toàn bộ nằm **ngoài** phạm vi 9 stage của
`rtl-design-flow-skills` (bộ skill dừng ở testbench) — không có orchestrator tool-execution nào
tương đương với skill set cũ (`vflow`).

Dấu hiệu: "chạy synth cho module này", "chạy full FE flow", "thêm constraint", "Fmax bao nhiêu",
"dựng lại filelist", "chạy lại toàn bộ flow xem còn phase nào dở".

## Bao nhiêu plan là đủ — 3 mức

| Tình huống | Cần gì |
|---|---|
| **Chỉ chạy tool để xem số**, output nằm trong `synth/` `syn/` `sim/` `prj/`, không sửa constraint / RTL / script ngoài các thư mục đó | **Không cần plan.** Chạy, đọc report, báo số kèm path. Hook không chặn các thư mục này. |
| Chạy có mục tiêu (đối chiếu baseline, trước/sau một thay đổi), muốn giữ lại làm record, nhưng **không** đổi constraint / RTL / script | **`FLOW` + `Plan weight: LITE`** — xem danh sách mục bắt buộc dưới |
| **Signoff**, hoặc đổi constraint / `.xdc` ngoài `synth/` / Makefile / flow config / waiver | **`FLOW` + `Plan weight: FULL`** |

Không rõ mình đang ở mức nào → mặc định mức thấp hơn và nói rõ; tôi nâng lên nếu muốn.

### `Plan weight: LITE` — mục bắt buộc

Chỉ cần: §1 Metadata · §2.1 User Request · §2.3 Expected Outcome · §2.4 Scope · §2.9 Stages &
signoff intent · §3.1 Flow status · §3.4 Current verification status *(làm baseline)* · §4.7 Tool
configuration · §6.3 FE Flow Impact · §6.4 Tool runs · §6.8 Reports & QoR · §7.4 Tool availability ·
§8.3 Execution Log · §8.5 Final Result.

Các mục còn lại ghi **`N/A — FLOW-LITE (thăm dò, không đổi thiết kế)`** — vẫn **không xoá heading**.
`§6.9 Signoff criteria` → `N/A — lần chạy thăm dò`.

Evidence rule **không** được nới ở LITE: số nào cũng phải kèm path report đã đọc thật.

## §1 Metadata — giá trị mặc định

| Field | Value |
|---|---|
| Task type | `FLOW` |
| Plan weight | `LITE` (thăm dò) / `FULL` (signoff, hoặc đổi constraint/script) |
| Flow phase(s) | sau Stage 09 (Testbench) — ngoài phạm vi 9 stage của `rtl-design-flow-skills` |
| Skills / Agents | **KHÔNG CÓ SKILL** — bộ `rtl-design-flow-skills` dừng ở testbench; không skill nào gọi tool bên ngoài (lint/sim/CDC/synth/STA/power). Toàn bộ FLOW là **MANUAL/USER-OWNED**: tôi tự chạy tool, agent chỉ đọc report tôi dán vào và không được bịa số. |
| Related artifacts | `synth/<m>/{run_synth.tcl,utilization.rpt,timing_summary.rpt,power.rpt,synth_report.md}`, `flow_status.md`, `tb/<ip>/script/` |

## Đổi tên trong core

- §5 `Implementation Plan` → **`Flow Execution Plan`**

## Chèn thêm

### §2.9 Stages to run & signoff intent

| Stage | Chạy? | Tool | Mục tiêu của lần chạy này |
|---|---:|---|---|
| Lint | | | |
| Simulation / regression | | | |
| Synthesis | | | |
| Timing (STA) | | | |
| Area | | | |
| Power | | | |

Đây là lần chạy **thăm dò** (xem số ra sao) hay **signoff** (phải đạt target §2.8)? Nói rõ — nó
quyết định plan này FAIL hay chỉ báo số.

### §4.7 Tool configuration

| Tool | Version | Script / entry | Ghi chú |
|---|---|---|---|
| Vivado | | `synth/<m>/run_synth.tcl` | part / device |
| xvlog / xelab / xsim | | `tb/<ip>/script/library.mk` | |
| verilator / iverilog | | | |

Thiếu tool → `NOT_RUN` + stage đó `✗ BLOCKED (tool unavailable: <tool>)`. Không bịa số.

### §4.8 Constraints

| Constraint | File | Giá trị | New / Changed |
|---|---|---|---|
| clock period / Fmax target | | | |
| input delay / output delay | | | |
| false path / multicycle | | | |

Đổi constraint là **đổi giả định thiết kế** — phải ghi ở §7.2 và tôi duyệt, không tự nới lỏng
constraint để cho timing pass.

### §6.8 Reports & QoR

So với baseline (lần chạy trước, nếu có). Điền **sau** khi chạy; lúc lập plan để `NOT_RUN`.

| Metric | Baseline | Sau lần chạy này | Delta | Target §2.8 | Verdict |
|---|---|---|---|---|---|
| LUT / FF / BRAM / DSP | | | | | NOT_RUN |
| Fmax / WNS | | | | | NOT_RUN |
| Power | | | | | NOT_RUN |

Artifact: `utilization.rpt` · `timing_summary.rpt` · `power.rpt` · `synth_report.md` — kèm path thật.

### §6.9 Signoff criteria

Chỉ dùng khi §2.9 nói đây là signoff; ngược lại `N/A — lần chạy thăm dò`.

- [ ] Synthesis hoàn tất, không unresolved reference, không structure sinh ngoài ý (latch, memory)
- [ ] Setup / hold clean, không unconstrained path
- [ ] Fmax ≥ target §2.8
- [ ] Area / Power trong target
- [ ] Lint sạch hoặc waiver có ghi chép
- [ ] `flow_status.md` phản ánh đúng trạng thái sau lần chạy

## Acceptance bắt buộc thêm

- [ ] Mỗi stage đã chạy đều có **đường dẫn report thật** và đã đọc report đó
- [ ] Stage không chạy được vì thiếu tool → ghi `✗ BLOCKED (tool unavailable: <tool>)`, không ghi PASS
- [ ] Không sửa `rtl/` / `lib/` trong plan này (nếu flow lộ ra lỗi → mở plan `DEBUG`)
- [ ] Constraint thay đổi đã được tôi duyệt ở §7.2 *(LITE: không được đổi constraint — đổi thì phải nâng lên FULL)*
- [ ] `flow_status.md` được cập nhật (thủ công, dựa trên report đã đọc)

## FE Flow Impact mặc định

| Stage | Required? | Reason |
|---|---:|---|
| RTL | NO | FLOW không đổi chức năng RTL |
| Testbench | MAYBE | YES nếu phải sửa `script/`, `rtl.f`, Makefile |
| Simulation | MAYBE | theo §2.9 |
| Lint / Synthesis / Timing / Area / Power | theo §2.9 | đây là trọng tâm của plan |
| CDC | MAYBE | YES nếu có tool CDC; hiện tại thường `NO — tool unavailable` |
| Documentation | MAYBE | YES nếu `synth_report.md` / doc QoR cần cập nhật |

## Cấm

- Không nới constraint / bỏ check để báo "PASS".
- Không sửa RTL trong plan FLOW — phát hiện lỗi thì mở plan `DEBUG` và ghi ở §8.5 `Follow-up`.
- Không đoán số area/timing khi Vivado không chạy được.
