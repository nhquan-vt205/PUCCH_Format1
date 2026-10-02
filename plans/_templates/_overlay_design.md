<!-- OVERLAY: DESIGN. Ap len _PLAN_CORE.md. KHONG sua file nay. -->

# Overlay: `DESIGN`

## Khi nào dùng

Quyết định **thiết kế**, chưa phải viết code hoàn chỉnh: architecture, phân rã submodule,
microarchitecture, interface, FSM, pipeline, clock/reset architecture, memory architecture.
Output là **proposal + doc + backbone**, không phải RTL chạy được.

Dấu hiệu: "thiết kế…", "chia module thế nào", "chọn kiến trúc nào", "định nghĩa interface".
Nếu thiết kế đã rõ và chỉ cần viết code → dùng `IMPLEMENT`.

## §1 Metadata — giá trị mặc định

| Field | Value |
|---|---|
| Task type | `DESIGN` |
| Flow phase(s) | Stage 01 Spec · Stage 02 Architecture · Stage 03 Microarchitecture · Stage 04 Logic Diagram |
| Skills / Agents | `spec-analyzer` (spec/requirement analysis) · `rtl-architect` (architecture + decomposition + `<ip>_top` skeleton) · `microarchitecture-designer` (microarchitecture proposal) · `logic-diagram-designer` (logic diagram) · `design-reviewer` (cross-cutting) |
| Related artifacts | spec/requirement doc, architecture doc, microarchitecture proposal, logic diagram, `doc/<m>.md`, `rtl/<ip>_top` skeleton |

## Chèn thêm

### §4.7 Microarchitecture decision record

Plan **không** chứa microarchitecture đầy đủ (chỗ đó là `ddoc/<m>_proposal.md`). Ở đây chỉ chốt các
quyết định mà proposal phải tuân theo:

| # | Quyết định | Phương án loại bỏ | Ảnh hưởng tới interface / timing / area |
|---|---|---|---|
| MD1 | | | |

### §4.8 Interface contract

Chốt trước khi code, vì module khác sẽ phụ thuộc:

| Port | Dir | Width | Protocol / handshake | Ổn định? |
|---|---|---|---|---|
| | | | | chốt / còn mở |

Reg-map (nếu IP có CPU bus): địa chỉ, field, reset value, access — hoặc `N/A — <lý do>`.

### §5.4 Submodule decomposition

Chỉ dùng khi task ở stage 02 (`rtl-architect`), ngược lại `N/A`.

| Submodule | Vai trò | `ddoc/<m>_req.md` sẽ tạo? | Instance trong `<ip>_top.v` |
|---|---|---|---|
| | | | |

## Acceptance bắt buộc thêm

- [ ] Microarchitecture proposal đã tạo/cập nhật và phản ánh đúng §4.7
- [ ] `doc/<m>.md` theo `ModuleDocContract.md`
- [ ] `<ip>_top` skeleton (stage 02 `rtl-architect`) có đủ wire/instance, **chưa** implement logic
      per-module — logic thuộc stage 05/06
- [ ] Plan `DESIGN` **không** tự sinh `rtl/<module>` backbone với `//@` markers (đó là việc của
      stage 05 `implementation-planner`)
- [ ] Interface §4.8 đã chốt hoặc điểm còn mở đã ghi ở §4.6 và tôi đã trả lời

## FE Flow Impact mặc định

| Stage | Required? | Reason |
|---|---:|---|
| Specification / Architecture | YES | chính là task này |
| RTL | YES (`<ip>_top` skeleton only) | stage 02 tạo skeleton, chưa có `//@` backbone per-module |
| Simulation / Lint | NO | chưa có logic để chạy — sẽ ở plan `IMPLEMENT` |
| Synthesis / Timing / Area / Power | NO | chưa có RTL hoàn chỉnh |
| Documentation | YES | `doc/<m>.md` là output bắt buộc của stage 02/03 |

## Cấm

- Không điền logic implementation trong plan `DESIGN` (đó là việc của `IMPLEMENT` / stage 06
  `rtl-implementer`).
- Không tự sinh `rtl/<module>` backbone với `//@` markers (đó là việc của stage 05
  `implementation-planner`, không phải stage 02/03).
- Không xoá `//@` markers.
- Không ghi `PASS` cho lint/sim vì chưa chạy được.
