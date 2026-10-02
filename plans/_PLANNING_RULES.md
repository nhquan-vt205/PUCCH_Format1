<!-- Planning rules dung chung cho moi plan trong thu muc nay. KHONG sua file nay trong project;
     ban goc o ~/.claude/plan-gate/templates/_planning_rules.md -->

# Planning Rules

PLAN_POLICY: rtl-design-flow

File này khai báo **policy** của project (Layer 2 — khác với **scope**, do `roots.txt` quyết định
ở Layer 1). Dòng `PLAN_POLICY:` ở trên là nguồn duy nhất Plan Gate dùng để biết áp dụng workflow
nào; đừng xoá hay đổi key `PLAN_POLICY`, chỉ đổi giá trị nếu project này thực ra theo một policy
khác (hiện chỉ `rtl-design-flow` có workflow đã implement).

Toàn bộ nội dung RTL chi tiết dưới đây (mục 1, 8, …) là nội dung của riêng policy
`rtl-design-flow` — áp dụng vì dòng `PLAN_POLICY` ở trên khai báo đúng policy đó, không phải vì
project nằm trong `roots.txt`.

Plan trong `plans/` là **implementation contract** giữa tôi và agent, không phải TODO list.
Agent phải chứng minh được: **hiểu yêu cầu → hiểu hiện trạng → biết sẽ đổi gì → biết verify thế
nào → tôi duyệt → mới được làm.**

---

## 1. Chọn plan type

Plan được ghép từ **core dùng chung** + **overlay theo type**:

```text
_templates/_PLAN_CORE.md   (8 mục, dùng chung)
        +
_templates/_overlay_<type>.md   (đổi tên mục / chèn mục chuyên biệt / acceptance riêng)
        ↓
plans/YYYY-MM-DD-HHmm-<short-title>.md   (một file TỰ CHỨA)
```

| Type | Dùng khi | Trọng tâm | Stage · Skill |
|---|---|---|---|
| `DESIGN` | quyết định thiết kế: architecture, phân rã, interface, FSM, pipeline | spec → requirement → architecture → interface → microarchitecture → logic diagram | 01 · `spec-analyzer`, 02 · `rtl-architect`, 03 · `microarchitecture-designer`, 04 · `logic-diagram-designer` |
| `IMPLEMENT` | thiết kế đã rõ, viết code: thêm module/feature/port/register, tích hợp `lib/` | requirement → RTL → unit sim | 05 · `implementation-planner`, 06 · `rtl-implementer` |
| `REFACTOR` | đổi cấu trúc code nhưng **giữ behavior** | behavior equivalence → impact → regression | `change-impact-analyzer` để khoanh vùng; 05 · `implementation-planner`, 06 · `rtl-implementer` để sửa; 07 · `rtl-reviewer` để review lại; QoR/regression là **MANUAL**, không skill |
| `DEBUG` | có hiện tượng sai cụ thể: sim/lint/synth/timing/CDC fail | reproduce → root cause → fix → regression | **không có skill** — bộ skill này không chạy tool nên không chẩn đoán lỗi tool được; hoàn toàn MANUAL |
| `VERIFY` | xây/sửa verification: bench, testcase, scoreboard, coverage | test strategy → coverage → self-check | 08 · `vplan-designer`, 09 · `tb-generator` (chạy test vẫn là MANUAL) |
| `FLOW` | chạy/dựng tool flow: lint, CDC, synth, STA, constraints, QoR | tool flow → constraints → reports → signoff | **không có skill** — flow này dừng ở testbench, không skill nào gọi tool bên ngoài; hoàn toàn MANUAL/USER-OWNED |
| `REVIEW` | đánh giá, **không sửa source** | findings → severity → recommendation | 07 · `rtl-reviewer`, cross-cutting `design-reviewer` |
| `DOC` | viết/cập nhật/tổng hợp tài liệu | doc contract → đối chiếu doc ↔ RTL | `rtl-documenter` (chiều ngược — dùng khi repo đã có RTL nhưng thiếu doc) |

### Ai chọn type

Agent **tự chọn** và phải nói rõ ở **dòng đầu câu trả lời** + ở `> Plan type:` đầu file plan:
`type + một câu lý do`. Tôi bác thì đổi lại, không tranh luận dài.

Mapping nhanh từ cách tôi nói → type:

| Tôi nói kiểu | Type |
|---|---|
| "thiết kế…", "chia module thế nào", "chọn kiến trúc nào" | `DESIGN` |
| "implement…", "thêm enable/reset vào…", "viết module theo `_req.md`" | `IMPLEMENT` |
| "refactor…", "tách…", "gọn lại…", "tham số hoá…" | `REFACTOR` |
| "đang fail…", "sai ở cycle…", "vì sao ra X" | `DEBUG` |
| "viết testbench…", "thêm testcase…", "cover trường hợp…" | `VERIFY` |
| "chạy synth/full flow…", "Fmax bao nhiêu", "thêm constraint" | `FLOW` |
| "review…", "kiểm tra xem có đúng…", "soát lại…" | `REVIEW` |
| "viết doc…", "doc đang lệch RTL", "tổng hợp tài liệu IP" | `DOC` |

Ranh giới mờ (ví dụ refactor kèm thêm feature, hay debug mà phải đổi kiến trúc) → **hỏi tôi**, hoặc
đề xuất **tách thành 2 plan** và nói rõ thứ tự.

### Cách ghép core + overlay

1. Copy toàn bộ `_templates/_PLAN_CORE.md` thành file plan mới.
2. Mở `_templates/_overlay_<type>.md` và áp **đúng** những gì nó nói:
   - điền `§1 Metadata` theo bảng giá trị mặc định của overlay;
   - **đổi tên** các mục mà overlay yêu cầu (giữ nguyên số);
   - **chèn** các mục chuyên biệt vào đúng vị trí số thứ tự (ví dụ `§2.9`, `§4.7`, `§6.8`);
   - thêm các dòng `Acceptance bắt buộc` của overlay vào `§6.7`;
   - lấy bảng `FE Flow Impact mặc định` của overlay làm điểm khởi đầu cho `§6.3`, rồi **sửa theo
     task thật** — không copy máy móc;
   - đọc mục `Cấm` của overlay và tôn trọng nó.
3. File plan cuối cùng phải **tự chứa**: không viết "xem overlay" thay cho nội dung thật, vì plan là
   engineering record đọc lại sau nhiều tuần.

Một yêu cầu **một** file plan. Không tạo plan thứ hai cho cùng một yêu cầu — sửa chính file đó.
Không sửa `_PLAN_CORE.md`, `_overlay_*.md`, hay file rules này trong project.

---

### Khi nào KHÔNG cần plan

- Câu hỏi read-only, giải thích code, tra cứu, đọc log.
- **Chạy tool để quan sát** (lint / sim / synth — tự chạy tay, không có skill scan/orchestrate) khi output chỉ nằm trong
  `synth/` `syn/` `sim/` `prj/` và không sửa constraint / RTL / testbench / script ngoài các thư mục
  đó. Hook **không chặn** các thư mục này, vì đó là artifact do tool sinh ra, không phải design
  intent. Vẫn phải báo số kèm **đường dẫn report đã đọc**.
- Sửa đổi mà tôi đã mô tả tường tận trong chính prompt — khi đó nói rõ là làm trực tiếp, không plan.

Ngược lại, vẫn **cần plan**: đổi `.xdc`/`.sdc` nằm ngoài `synth/`, đổi Makefile / `library.mk` /
flow config, thêm waiver, và mọi thay đổi trong `rtl/` `lib/` `tb/` `ddoc/`.

### Plan weight

`FULL` là mặc định. `LITE` **chỉ** dùng cho `FLOW` thăm dò (xem `_overlay_flow.md`): rút ngắn danh
sách mục phải điền, nhưng **không** nới quy tắc evidence — số nào cũng phải có path report thật.

## 2. Status lifecycle

```text
DRAFT
  ↓                (agent đang viết)
READY_FOR_REVIEW   ← đưa tôi đọc, DỪNG LẠI ở đây
  ↓                (tôi nói "làm đi" / "ok làm" / "approve")
APPROVED
  ↓
IN_PROGRESS        (đang implement)
  ↓
VERIFICATION       (code xong, đang chạy lint / sim / synth)
  ↓
COMPLETED
```

Nhánh khác: `NEEDS_REVISION` (tôi yêu cầu sửa plan) · `SUPERSEDED` (bị plan khác thay) ·
`ABANDONED` (bỏ).

- Agent **chỉ** được đổi sang `APPROVED` khi tôi nói rõ. **Không tự approve thay tôi.**
- Hook chỉ cho ghi source khi `Status` ∈ {`APPROVED`, `IN_PROGRESS`, `VERIFICATION`}.
- `COMPLETED` = đóng plan. Yêu cầu mới → plan mới, không mở lại plan cũ.
- `PENDING_REVIEW` ≡ `READY_FOR_REVIEW`, `DONE` ≡ `COMPLETED` (tương thích plan cũ).

---

## 3. Evidence-Based Completion

> Không bao giờ đánh dấu một bước là `PASS` dựa trên suy luận, kỳ vọng, hay đọc source code,
> khi bước đó cần tool kiểm chứng.

| Bước | Bằng chứng bắt buộc |
|---|---|
| RTL compile / elaborate | log compile |
| Simulation | log sim + token `[FINISH] PASS`, số TC pass/total |
| Lint | lint report / output tool |
| CDC | CDC report |
| Synthesis | `synth/<module>/utilization.rpt`, `synth_report.md` |
| Timing / STA | `synth/<module>/timing_summary.rpt` |
| Area / Power | `utilization.rpt` / `power.rpt` |

- **"Should pass" không phải "PASS". "Looks correct" không phải "PASS". "Implemented" không phải
  "Verified".**
- Lúc lập plan, mọi status kiểm chứng phải là `PENDING` hoặc `NOT_RUN` — **không** được có `PASS`.
- Mỗi `PASS`/`FAIL` phải kèm **đường dẫn artifact** (log / report) đã thật sự đọc.
- Tool không có trên máy → `NOT_RUN` + ghi thiếu tool gì, và stage tương ứng
  `✗ BLOCKED (tool unavailable: <tool>)`. Tuyệt đối không bịa output.

```text
PLAN → PENDING → EXECUTION → EVIDENCE → RESULT
```

---

## 4. Traceability

Mọi requirement phải có đường đi tới chỗ nó được verify:

```text
Requirement → Design change → RTL file → Test case → Simulation → Lint/CDC → Synthesis → Timing → Acceptance
```

- Mỗi `REQ-NNN` trace về một câu trong `User Request (verbatim)` hoặc một mục trong
  `ddoc/<module>_req.md`. **Không tự sinh requirement.**
- Mỗi step trong Implementation Plan phải ghi nó cover `REQ` nào.
- Mỗi `REQ` phải xuất hiện trong Traceability Matrix và có **ít nhất một** phương pháp verify.
  `REQ` không có cách verify = plan chưa đủ tốt.

---

## 5. FE Flow Impact — phải giải thích khi SKIP

Không phải task nào cũng chạy full flow. Nhưng **không được im lặng bỏ qua** một stage: mỗi stage
phải có `YES` / `NO` / `MAYBE` **kèm lý do**.

- `NO — không có clock crossing nào thay đổi` ✔
- (để trống) ✘

CDC và STA: bộ skill không có công cụ CDC riêng, và không skill nào chạy synth — toàn bộ
lint/sim/CDC/synth/STA/power là **MANUAL**, do tôi tự chạy tool rồi dán report lại. STA lấy từ
`timing_summary.rpt` do tool synth sinh ra (không phải do skill). Nếu stage không chạy được vì
thiếu tool → `NO — tool unavailable: <tool>`.

---

## 6. Mục không áp dụng

Giữ **nguyên** 8 mục và số thứ tự của template. Mục/bảng không áp dụng → ghi
`N/A — <lý do cụ thể>`, **không xoá heading**, không xoá dòng. Lý do kiểu "không cần" là không đủ.

---

## 7. Plan là engineering record

Plan không chỉ phục vụ lúc bắt đầu task. Sau khi xong, nó là hồ sơ để vài tuần sau trả lời được
"tại sao block này được thiết kế như vậy". Vì vậy:

- Không xoá phần phân tích cũ khi sửa plan — ghi thay đổi vào `Revision History`.
- Lệch plan lúc làm → ghi vào `Deviations From Approved Plan`, không âm thầm sửa lại phần kế hoạch.
- Xong thì điền `Final Result` đầy đủ, thêm một dòng vào `plans/INDEX.md`, và link chéo với file
  session trong `memory_bank/`.

---

## 8. Ràng buộc của project RTL (`rtl-design-flow-skills`)

Theo `~/.claude/skills/shared/` và từng `SKILL.md` dưới `~/.claude/skills/<stage>/`:

- 9 stage: `01` Spec (`spec-analyzer`) · `02` Architecture (`rtl-architect`) · `03`
  Microarchitecture (`microarchitecture-designer`) · `04` Logic Diagram (`logic-diagram-designer`) ·
  `05` Implementation Plan (`implementation-planner`) · `06` RTL (`rtl-implementer`) · `07` RTL
  Review (`rtl-reviewer`) · `08` Verification Plan (`vplan-designer`) · `09` Testbench
  (`tb-generator`). Song song: điều phối bởi `design-flow-manager`; cross-cutting
  `change-impact-analyzer`, `design-reviewer`; chiều ngược `rtl-documenter` (RTL có trước, doc
  sinh sau).
- Traceability chain: `Requirement → Architecture → Microarchitecture → Logic Diagram →
  Implementation Plan → RTL → RTL Review → VPlan → Testbench/Test`.
- **Bộ skill dừng ở testbench.** Không skill nào gọi tool bên ngoài, và không skill nào được
  phép báo một kết quả tool mà nó chưa thật sự đọc. Simulation, lint, CDC, synthesis, STA/timing,
  power, regression, waveform debug, tool-specific debug đều **MANUAL/USER-OWNED** — tôi tự chạy
  tool, agent không tự suy ra hay bịa kết quả.
- Token trạng thái stage: `✓ DONE` · `⚠ IN PROGRESS (n/m)` · `✗ PENDING` · `✗ BLOCKED (<reason>)` ·
  `✗ ERROR (<reason>)`.
- `rtl/` + `lib/` phải **Verilog-2005**; chỉ `tb/` được dùng SystemVerilog (trừ khi
  `config/coding_style.md` của project ghi đè).
- Reset **đồng bộ**, active-low mặc định, dùng `RS_LV` khi cấu hình được.
- `rtl/<ip>_top` skeleton thuộc stage 02 `rtl-architect` — các stage sau không tạo/ghi đè file này.
- `rtl/<module>` backbone với `//@` markers do stage 05 `implementation-planner` sinh ra; stage 06
  `rtl-implementer` **bị cấm** tự sinh backbone, chỉ được điền logic vào marker có sẵn. `//@`
  markers chỉ xoá khi implementation của đúng module đó xong.
- Mỗi module trong `rtl/` phải có doc theo `ModuleDocContract.md` (12 mục, lifecycle
  INTENDED → AS-BUILT); module trong `lib/` có doc cùng thư mục.
- Microarchitecture chi tiết thuộc artifact của stage 03 — plan chỉ ghi quyết định và delta,
  không copy lại nguyên văn.
- Pattern-scan / reflex-trigger và no-fabricated-numbers (`TBD`/`UNKNOWN`/`USER_INPUT_REQUIRED`/
  `ASSUMP-###`) áp dụng theo `DesignPatterns.md` và từng `SKILL.md` — không tự điền số khi chưa có
  bằng chứng.
- **Checkpoint discipline:** không bỏ qua checkpoint, duyệt từng module một (không batch-approve),
  và phân biệt "result" của skill (xử lý theo rule của stage) với "abort" (dừng, đánh dấu
  BLOCKED, in nguyên output, không tự tiếp tục).
