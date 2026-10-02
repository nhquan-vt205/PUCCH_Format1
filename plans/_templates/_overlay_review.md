<!-- OVERLAY: REVIEW. Ap len _PLAN_CORE.md. KHONG sua file nay. -->

# Overlay: `REVIEW`

## Khi nào dùng

**Đánh giá**, không sửa: review RTL / testbench / doc / flow, audit style, đối chiếu RTL vs
`_req.md` vs `doc/`, đánh giá testability, soát sẵn-sàng-synth.

Dấu hiệu: "review…", "kiểm tra xem có đúng…", "module này ổn chưa", "soát lại…".
Deliverable là **findings**, không phải patch. Muốn sửa → plan `IMPLEMENT`/`REFACTOR`/`DEBUG` riêng.

## §1 Metadata — giá trị mặc định

| Field | Value |
|---|---|
| Task type | `REVIEW` |
| Flow phase(s) | Stage 07 RTL Review *(không đẩy stage nào tiến lên)* |
| Skills / Agents | `rtl-reviewer` (stage 07) · `design-reviewer` (cross-cutting) · đọc `CodingStyle.md`, `ModuleDocContract.md`, `HierarchyFilelist.md` |
| Target version | commit / mtime của file được review |
| Deliverable | §8.5 Findings *(+ `doc/review_<target>.md` nếu tôi yêu cầu)* |

## Đổi tên trong core

- §2 `Objective & Requirements` → **`Objective & Review Criteria`**
- §4 `Technical Analysis` → **`Review Method`**
- §5 `Implementation Plan` → **`Review Plan`**
- §6 `Verification Plan` → **`Verification of the Review`** (kiểm tra chính review, không phải RTL)

## Chèn thêm

### §2.9 Question to answer

Review này trả lời **câu hỏi nào** của tôi. Một câu, cụ thể.

### §2.10 Review criteria

Mỗi criterion phải có verdict ở §8.5 — không được để trống.

| ID | Criterion | Nguồn chuẩn | Áp dụng? |
|---|---|---|---|
| C-001 | Coding style | `CodingStyle.md` | |
| C-002 | Verilog-2005 cho `rtl/`+`lib/` | `~/.claude/skills/shared/CodingStyle.md` | |
| C-003 | Doc contract, `doc/<m>.md` khớp RTL | `ModuleDocContract.md` | |
| C-004 | Khớp requirement | spec của stage 01 (`spec-analyzer`) | |
| C-005 | Khớp proposal | microarchitecture proposal của stage 03 | |
| C-006 | Reset đồng bộ, active-low / `RS_LV` | `~/.claude/skills/shared/CodingStyle.md` | |
| C-007 | `//@` marker lifecycle | `CodingStyle.md` | |
| C-008 | Testability / coverage | `tc_list.md` | |
| C-009 | Filelist / hierarchy | `HierarchyFilelist.md` | |

### §4.7 Evidence rules & severity

- Mỗi finding **phải** có `file:line` (hoặc bảng/dòng trong doc) + trích dẫn. Không bằng chứng =
  không được ghi là finding.
- Phân biệt: **defect** (sai chuẩn/spec) · **risk** (có thể sai) · **suggestion** (gu cá nhân).
- Không đoán ý định tác giả; chỗ mơ hồ → câu hỏi ở §4.6.

| Severity | Nghĩa |
|---|---|
| `BLOCKER` | sai chuẩn / sai spec, không được để nguyên |
| `MAJOR` | rủi ro thật về function / timing / verify |
| `MINOR` | lệch style, doc thiếu |
| `INFO` | ghi nhận |

### §6.8 Completeness checks

- [ ] Mỗi criterion §2.10 có verdict `PASS` / `FAIL` / `N/A — lý do`
- [ ] Mỗi file trong scope đã đọc hết phần liên quan (không đọc lướt rồi kết luận)
- [ ] Mỗi finding có `file:line` + severity + trích dẫn + đề xuất
- [ ] Không finding nào chỉ dựa trên suy đoán
- [ ] Không báo lại issue đã biết (§3.5) như phát hiện mới

### §8.5 Findings *(thay phần Final Result của core bằng bảng này + tổng kết)*

| ID | Severity | File:line | Criterion | Phát hiện (có trích dẫn) | Đề xuất xử lý | Follow-up plan? |
|---|---|---|---|---|---|---|
| F-001 | MAJOR | `rtl/<m>.v:42` | C-002 | | | cần plan `IMPLEMENT` |

**Verdict theo criterion**

| Criterion | Verdict | Ghi chú |
|---|---|---|
| C-001 | PENDING | |

**Tổng kết:** `BLOCKER` n · `MAJOR` n · `MINOR` n · `INFO` n — và trả lời §2.9.

## Acceptance bắt buộc thêm

- [ ] Trả lời được đúng câu hỏi §2.9
- [ ] **Không file nào** trong `rtl/` `lib/` `tb/` `ddoc/` bị sửa
- [ ] Findings xếp theo severity, có tổng kết
- [ ] Đã nêu rõ follow-up plan nào cần tạo

## FE Flow Impact mặc định

| Stage | Required? | Reason |
|---|---:|---|
| RTL review | YES | chính là task này |
| Lint (read-only) | MAYBE | YES nếu cần tìm warning chưa biết |
| Simulation | NO | review không chạy sim, trừ khi tôi yêu cầu |
| CDC | NO | không có tool CDC trong flow hiện tại |
| Synthesis / Timing / Area / Power | NO | không sửa RTL nên không chạy lại |
| Documentation | MAYBE | YES nếu deliverable là file report trong `doc/` |

## Cấm

- **Không sửa** `rtl/` `lib/` `tb/` `ddoc/` `script/` — kể cả lỗi hiển nhiên; ghi finding và mở plan
  khác.
- Không đưa patch/diff vào plan review (đề xuất bằng lời, không viết code thay).
- Không kết luận "ổn" cho criterion chưa thực sự soi.
