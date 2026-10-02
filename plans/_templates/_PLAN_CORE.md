<!-- PLAN CORE - phan dung chung cho moi plan type. KHONG sua file nay.
     Cach dung: copy core nay sang plans/YYYY-MM-DD-HHmm-<short-title>.md, roi ap
     _overlay_<type>.md tuong ung (doi ten muc / chen muc chuyen biet / them acceptance).
     Doc _PLANNING_RULES.md truoc. Giu nguyen 8 muc cap 1 va so thu tu;
     muc khong ap dung -> "N/A - <ly do cu the>", KHONG xoa heading. -->

# Plan (<TYPE>): <short title>

> Đây là **implementation contract** cho task này. Không được bắt đầu thực hiện
> trước khi plan được tôi duyệt (`Status` → `APPROVED`).
>
> Plan type: `<TYPE>` — **lý do chọn type:** <một câu>.

---

## 1. Metadata

| Field | Value |
|---|---|
| **Status** | `READY_FOR_REVIEW` |
| **Plan ID** | `PLAN-YYYYMMDD-NNN` |
| **Title** | |
| **Task type** | `<TYPE>` *(DESIGN / IMPLEMENT / REFACTOR / DEBUG / VERIFY / FLOW / REVIEW / DOC)* |
| **Plan weight** | `FULL` *(`LITE` chỉ dùng cho `FLOW` thăm dò — xem `_overlay_flow.md`)* |
| **Revision** | `1` |
| **Created** | YYYY-MM-DD HH:MM |
| **Updated** | YYYY-MM-DD HH:MM |
| **Project / IP** | `<project>` / `<ip_name>` |
| **Target** | `rtl/<module>.v`, `lib/<module>/`, `tb/<ip>/` … |
| **Flow phase(s)** | *(theo overlay)* |
| **Skills / Agents** | *(theo overlay)* — hoặc *sửa tay* |
| **Request source** | prompt trực tiếp / `ddoc/<m>_req.md` / `issue/<ip>/issue_NNN` / plan trước |
| **Plan file** | `plans/YYYY-MM-DD-HHmm-<short-title>.md` |
| **Session file** | `memory_bank/YYYY-MM-DD-HHmm-<...>.md` |
| **Related artifacts** | `ddoc/`, `doc/`, `tb/<ip>/tc_list.md`, `issue/<ip>/`, `synth/<m>/` |
| **Approved at** | — |

Lifecycle, evidence rule, traceability, bảng chọn type: xem `_PLANNING_RULES.md`.

---

## 2. Objective & Requirements

### 2.1 User Request (verbatim)

> Dán **nguyên văn** prompt của tôi — không diễn giải, không rút gọn, không sửa chính tả.
> Yêu cầu rải nhiều prompt → dán từng đoạn theo thứ tự.

### 2.2 Problem Statement

Vấn đề đang được giải quyết (vì sao cần làm), không phải mô tả lại việc sẽ làm.

### 2.3 Expected Outcome — Definition of Done

Sau khi xong thì điều gì đúng, và **nhìn vào artifact nào** để biết.

### 2.4 Scope

**In scope**
- ...

**Out of scope** — cố tình **không** làm, để tôi kịp phản đối
- ...
- *(mặc định ngoài scope nếu không nói rõ: đổi interface module khác, đổi reg-map, sửa testcase đang PASS, chạy full flow synth/sim)*

### 2.5 Requirements

Mỗi dòng trace về một câu ở §2.1 hoặc một mục trong `ddoc/<m>_req.md`. **Không tự sinh requirement.**

| ID | Requirement | Nguồn | Priority |
|---|---|---|---|
| REQ-001 | | §2.1 câu "…" / `_req.md` §x | Must |

### 2.6 Interface Requirements

| Signal / Interface | Dir | Width | New / Changed / Removed | Requirement | Ảnh hưởng tới |
|---|---|---|---|---|---|
| | | | | REQ-00x | |

Reg-map / `tb/<ip>/dut.vh` có đổi? → ghi rõ, hoặc `N/A — <lý do>`.

### 2.7 Clock / Reset Requirements

**Clock** — tên · tần số mục tiêu · cạnh · có domain crossing mới không
**Reset** — tên · polarity (active-low mặc định) · **đồng bộ** · dùng `RS_LV`? · hành vi khi reset

### 2.8 Performance Requirements

| Metric | Target | Nguồn target |
|---|---|---|
| Frequency (Fmax) | | |
| Latency | | |
| Throughput | | |
| Area | | |
| Power | | |

---

## 3. Current State

Chỉ ghi điều **đã đọc file / đã chạy tool**, kèm bằng chứng. Chưa kiểm tra → ghi "chưa đọc".
Không suy đoán.

### 3.1 Flow status

Nguồn: `flow_status.md` (ghi thời điểm) hoặc tự scan.
Token: `✓ DONE` · `⚠ IN PROGRESS (n/m)` · `✗ PENDING` · `✗ BLOCKED (<reason>)` · `✗ ERROR (<reason>)`.

| Stage | Status | Bằng chứng (file / output đã đọc) |
|---|---|---|
| 01 Spec | | |
| 02 Architecture | | |
| 03 Microarchitecture | | |
| 04 Logic Diagram | | |
| 05 Implementation Plan | | |
| 06 RTL | | |
| 07 RTL Review | | |
| 08 Verification Plan | | |
| 09 Testbench | | |

### 3.2 Relevant files

| File | Role | Liên quan thế nào |
|---|---|---|
| | | |

### 3.3 Current architecture & behavior

Kiến trúc hiện tại ở phần liên quan và **hành vi thực tế quan sát được** (trích RTL / log kèm dòng
cụ thể). Không viết "có lẽ".

### 3.4 Current verification status

| Stage | Status | Evidence (đường dẫn log/report đã đọc) |
|---|---|---|
| RTL compile | NOT_RUN | |
| Simulation | NOT_RUN | |
| Lint | NOT_RUN | |
| CDC | NOT_RUN | |
| Synthesis | NOT_RUN | |
| Timing / STA | NOT_RUN | |
| Area | NOT_RUN | |
| Power | NOT_RUN | |

### 3.5 Existing issues

`issue/<ip>/` đang mở gì (`issue_NNN_*`, `debug_NNN_*`, loại `RTL_BUG` / `TC_BUG` / `DOC_AMBIGUITY`),
`//@` markers còn ở đâu.

### 3.6 Context consulted

| Source | Đã đọc gì | Ràng buộc nó đặt ra |
|---|---|---|
| `memory_bank/<file>.md` | | |
| plan trước `plans/<file>.md` | | |
| `ddoc/<m>_proposal.md` | | |

### 3.7 Constraints

- **Interface:** ... (port / protocol không được đổi)
- **Architecture:** ... (quyết định cũ còn hiệu lực)
- **Tool:** ... (Vivado / xsim / verilator có gì, version)
- **Coding style:** `CodingStyle.md`, Verilog-2005 cho `rtl/`+`lib/`
- **Verification:** `[FINISH] PASS` token, bench CPU-bus-only
- **Compatibility:** testcase / plan / doc nào đang phụ thuộc phần này

---

## 4. Technical Analysis

### 4.1 Technical requirement

Yêu cầu kỹ thuật thực chất phía sau §2.2.

### 4.2 Design analysis

Phân tích hành vi thiết kế liên quan: datapath, FSM, handshake, latency, back-pressure, edge case.
**Không copy micro-architecture** — chỗ đó là `ddoc/<m>_proposal.md`; ở đây ghi phân tích và delta.

### 4.3 Dependencies

Loại: `MUST_CHANGE` · `MAY_CHANGE` · `READ_ONLY` · `GENERATED` · `EXTERNAL`.

| Dependency | Type | Impact |
|---|---|---|
| | | |

### 4.4 Alternatives considered

Có module nào trong `lib/` dùng lại được không (pattern scan theo `DesignPatterns.md` / reflex
triggers như stage 02-03 yêu cầu)? Trả lời rõ.

| Option | Cách làm | Ưu | Nhược | Chọn? |
|---|---|---|---|---|
| A | | | | **CHỌN** |
| B | | | | loại — lý do |

### 4.5 Selected approach

Vì sao phương án được chọn thoả `REQ-001..` và **không** phá gì.

### 4.6 Assumptions & Open Questions

| # | Assumption / Question | Ảnh hưởng nếu sai | Cần tôi trả lời? |
|---|---|---|---|
| A1 | | | không / **CÓ** |

---

## 5. Implementation Plan

### 5.1 Steps

`Gate`: `AUTO` = chạy tiếp · `CHECKPOINT` = dừng đưa tôi xem artifact · `GATE` = chờ tôi quyết.

| # | Step | Skill / Agent / sửa tay | Phase | Files | REQ covered | Artifact ra | Gate | Done |
|---|---|---|---|---|---|---|---|---|
| 1 | | | | | REQ-001 | | AUTO | [ ] |

### 5.2 File Change Manifest

| File | Action | Reason |
|---|---|---|
| | MODIFY | |
| | ADD | |
| | DELETE | |

**Files that must NOT change** (sửa ngoài danh sách trên = phải báo tôi trước khi tiếp tục)
- `rtl/<ip>_top.v` — skeleton thuộc stage 02 `rtl-architect`
- ...

### 5.3 Order & rollback points

Thứ tự thực thi, và chỗ nào dừng an toàn được nếu tôi muốn ngắt giữa đường.

---

## 6. Verification Plan

Mọi status ở mục này khi lập plan phải là `PENDING` / `NOT_RUN` — xem `_PLANNING_RULES.md` §3.

### 6.1 Verification strategy

Chứng minh đúng bằng cách nào, ở mức nào (unit `tb/<module>/` vs IP-level `tb/<ip>/`).

### 6.2 Test matrix

| Test ID | Requirement | Scenario | Expected result | File |
|---|---|---|---|---|
| TC-001 | REQ-001 | | `[FINISH] PASS` | `tb/<ip>/tb_<name>.sv` |

Testcase đang có bị ảnh hưởng (`tc_list.md`): trước → kỳ vọng sau.

### 6.3 FE Flow Impact

Mỗi stage **phải** có `YES` / `NO` / `MAYBE` **kèm lý do**. Không được để trống, không im lặng skip.

| Stage | Required? | Reason | Status |
|---|---:|---|---|
| Specification (stage 01 `spec-analyzer`) | | | PENDING |
| Architecture (stage 02 `rtl-architect`) | | | PENDING |
| Microarchitecture (stage 03) | | | PENDING |
| RTL (stage 05-06 `implementation-planner`/`rtl-implementer`) | | | PENDING |
| RTL review (stage 07 `rtl-reviewer`) | | | PENDING |
| Verification plan (stage 08 `vplan-designer`) | | | PENDING |
| Testbench (stage 09 `tb-generator`) | | | PENDING |
| Simulation — **MANUAL**, không skill | | | PENDING |
| Lint — **MANUAL**, không skill | | | PENDING |
| CDC | NO | không có tool CDC trong flow hiện tại / không đổi clock crossing | N/A |
| Synthesis — **MANUAL**, không skill | | | PENDING |
| Timing / STA (`timing_summary.rpt`) — **MANUAL** | | | PENDING |
| Area | | | PENDING |
| Power | | | PENDING |
| Documentation (`rtl-documenter`, chiều ngược) | | | PENDING |

### 6.4 Tool runs

| Check | Tool | Command | Pass criteria | Artifact sẽ tạo | Status |
|---|---|---|---|---|---|
| Lint | verilator / iverilog | | 0 error, warning phải giải thích | | NOT_RUN |
| Unit sim | xvlog/xelab/xsim | `tb/<module>/` | `[FINISH] PASS`, không X/Z bất thường | log | NOT_RUN |
| IP testcases | make | `tb/<ip>/` → `make <tc>` | tất cả `[FINISH] PASS` | `sim/<tc>/` | NOT_RUN |
| Synthesis | Vivado | `synth/<m>/run_synth.tcl` (chạy tay) | hoàn tất, không unresolved ref | `utilization.rpt`, `synth_report.md` | NOT_RUN |
| Timing | Vivado | (trong flow synth, chạy tay) | setup/hold clean, đạt Fmax §2.8 | `timing_summary.rpt` | NOT_RUN |

### 6.5 Traceability Matrix

| Requirement | Change (file) | Test | Sim | Lint | CDC | Synth | Timing |
|---|---|---|---|---|---|---|---|
| REQ-001 | | TC-001 | PENDING | PENDING | N/A | PENDING | PENDING |

**Mỗi requirement phải có ít nhất một cách verify.** Dòng nào không có = plan chưa đủ.

### 6.6 Contract checks

- [ ] `rtl/` + `lib/` là Verilog-2005 (không construct SystemVerilog); chỉ `tb/` được `.sv`.
- [ ] Coding style theo `~/.claude/skills/shared/CodingStyle.md`.
- [ ] Doc theo `ModuleDocContract.md`; `doc/<module>.md` có và khớp RTL.
- [ ] Filelist / hierarchy theo `HierarchyFilelist.md` (`tb/<ip>/script/rtl.f` còn đúng).
- [ ] Testcase kết thúc bằng `[FINISH] PASS` / `[FINISH] FAIL`.
- [ ] `//@` markers đã xử lý đúng lifecycle.

### 6.7 Acceptance criteria

Chỉ tick mục **áp dụng**; mục không áp dụng ghi `N/A — lý do`. Overlay có thể thêm mục bắt buộc.

- [ ] Tất cả requirement §2.5 đã đạt
- [ ] Tất cả test bắt buộc §6.2 PASS (có log)
- [ ] RTL compile / elaborate sạch
- [ ] Lint đạt (warning còn lại đã giải thích / waive có ghi chép)
- [ ] CDC đạt hoặc `N/A — lý do`
- [ ] Synthesis hoàn tất
- [ ] Timing đạt target §2.8
- [ ] Area / Power đạt target hoặc `N/A — lý do`
- [ ] Không có regression nào chưa giải thích được
- [ ] Artifact bắt buộc đều có và đã đọc
- [ ] `doc/` + `memory_bank/` + `plans/INDEX.md` đã cập nhật

---

## 7. Risks & Decisions

### 7.1 Risks

Nhóm cần soi: functional regression · timing regression · area/power regression · CDC · reset ·
clocking · sim-synth mismatch · interface compatibility · coverage gap.

| Risk | Likelihood | Impact | Mitigation / rollback |
|---|---|---|---|
| | | | |

### 7.2 Key decisions

| # | Quyết định | Lý do | Ai quyết |
|---|---|---|---|
| D1 | | | agent / **tôi** |

### 7.3 Impact

- Module / testcase / doc bị ảnh hưởng ngoài phạm vi sửa trực tiếp: ...
- Có phá interface, reg-map, hay testcase đang PASS? ...
- Phase nào trong `flow_status.md` sẽ lùi về `IN PROGRESS`? ...
- Ước lượng công: ...

### 7.4 Tool availability

| Tool | Có? | Kiểm tra bằng | Thiếu thì stage nào BLOCKED |
|---|---|---|---|
| verilator / iverilog | | | Lint |
| xvlog / xelab / xsim | | | Simulation |
| Vivado | | | Synthesis + Timing |

---

## 8. Approval / Execution / Result

### 8.1 Revision History

| Rev | Date | Tôi yêu cầu sửa gì | Agent đã sửa gì trong plan |
|---|---|---|---|
| 1 | | (bản đầu) | — |

### 8.2 Approval

- [ ] Tôi đã đọc và đồng ý → tôi sẽ nói "làm đi".
- Khi được approve: `Status` → `APPROVED`, điền `Approved at`, rồi `IN_PROGRESS` khi bắt đầu,
  `VERIFICATION` khi chạy tool, `COMPLETED` khi xong. **Không tự approve thay tôi.**
- Không gộp thêm việc ngoài §2.4 trong lúc thực hiện.

### 8.3 Execution Log

Status mỗi step: `PENDING` · `IN_PROGRESS` · `PASS` · `FAIL` · `BLOCKED` · `SKIPPED`.
`PASS` phải kèm evidence. Lint/sim `FAIL` → **dừng**, không đi tiếp phase sau.

| Step | Status | Evidence (log / report path) | Notes |
|---|---|---|---|
| 1 | PENDING | | |

### 8.4 Deviations From Approved Plan

| Date | Step | Deviation | Reason | Impact |
|---|---|---|---|---|
| | | | | |

### 8.5 Final Result

**Final status:** `PENDING`

- **Summary:** ...
- **Verification summary:** (copy status cuối từ §6.4 — chỉ `PASS`/`FAIL` nếu đã chạy thật, kèm path)
- **Changed files:** ...
- **Generated artifacts:** ...
- **Known issues:** ...
- **Follow-up tasks:** ...

### 8.6 Record keeping

- [ ] Đã thêm 1 dòng vào `plans/INDEX.md`
- [ ] Đã cập nhật file session trong `memory_bank/` (và link tới plan này)
- [ ] `flow_status.md` còn đúng (hoặc đã scan lại thủ công theo trạng thái 9 stage)
