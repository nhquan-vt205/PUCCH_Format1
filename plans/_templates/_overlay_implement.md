<!-- OVERLAY: IMPLEMENT. Ap len _PLAN_CORE.md. KHONG sua file nay. -->

# Overlay: `IMPLEMENT`

## Khi nào dùng

Thiết kế đã tương đối rõ, việc còn lại là **thực hiện**: điền backbone thành RTL hoàn chỉnh, thêm
module / feature / port / register / FSM theo spec, tích hợp IP có sẵn từ `lib/`.

Dấu hiệu: "implement…", "thêm enable/reset vào…", "viết module theo `_req.md`", "tích hợp `lib/x`".
Nếu kiến trúc còn phải quyết → `DESIGN` trước. Nếu code đổi mà behavior phải giữ → `REFACTOR`.

## §1 Metadata — giá trị mặc định

| Field | Value |
|---|---|
| Task type | `IMPLEMENT` |
| Flow phase(s) | Stage 05 Implementation Plan · Stage 06 RTL *(kèm Stage 07 review nếu §6.3 nói YES)* |
| Skills / Agents | `implementation-planner` (`//@` backbone, stage 05) · `rtl-implementer` (điền logic, stage 06 — **không tự chạy lint/sim**) |
| Related artifacts | microarchitecture proposal, `rtl/<m>.v` backbone + implementation, `doc/<m>.md`, `tb/<module>/` |

## Chèn thêm

### §2.9 Requirement → Design Mapping

Mỗi requirement phải chỉ ra nó được hiện thực **ở đâu** trong thiết kế đã chốt:

| REQ | Mục trong microarchitecture proposal | Block / signal trong RTL | `//@` marker liên quan |
|---|---|---|---|
| REQ-001 | | | |

Requirement **không** có chỗ trong proposal → hoặc proposal thiếu (phải cập nhật, ghi ở §8.4), hoặc
requirement nằm ngoài scope (ghi ở §2.4). Không tự ý implement thứ proposal không nói.
Nếu module chưa có `rtl/<m>.v` backbone với `//@` markers → đó là việc của stage 05
`implementation-planner` trước, `rtl-implementer` (stage 06) **bị cấm tự sinh backbone**.

### §5.4 New files & integration points

| File mới | Vai trò | Được instantiate ở đâu | Cần thêm vào `script/rtl.f`? |
|---|---|---|---|
| | | | |

Integration point: module cha nào phải sửa, `tb/<ip>/dut.vh` có phải cập nhật, filelist có đổi.

## Acceptance bắt buộc thêm

- [ ] Mọi `//@` marker của module đã xử lý xong và đã xoá
- [ ] Lint sạch (có log) — **MANUAL**, không skill nào tự chạy lint; agent chỉ đọc report tôi dán
- [ ] Unit sim `tb/<module>/` → `[FINISH] PASS` (có log) — **MANUAL**, không skill nào tự chạy sim
- [ ] `doc/<m>.md` cập nhật khớp RTL sau khi implement
- [ ] `script/rtl.f` / hierarchy còn đúng nếu có file mới
- [ ] Không đổi interface ngoài §2.6 đã khai

## FE Flow Impact mặc định

| Stage | Required? | Reason |
|---|---:|---|
| RTL | YES | chính là task này |
| Testbench | MAYBE | YES nếu module chưa có unit tb, hoặc cần TC mới cho REQ mới |
| Simulation | YES | phải chứng minh logic đúng, không chỉ compile — **MANUAL**, tôi tự chạy |
| Lint | YES | RTL thay đổi — **MANUAL**, tôi tự chạy |
| CDC | NO | *(sửa nếu task tạo clock crossing mới)* |
| Synthesis / Timing | MAYBE | YES nếu đổi logic depth / thêm path tới đường critical |
| Area / Power | MAYBE | YES nếu §2.8 có target |
| Documentation | YES | doc phải khớp RTL mới |

## Cấm

- Không "sửa luôn cho đẹp" phần ngoài §2.4 — refactor là plan riêng.
- Không đổi `ddoc/<m>_req.md` (đổi yêu cầu) mà không khai ở §2.4 và báo tôi.
- Không đánh dấu xong khi mới compile: `implemented` ≠ `verified`.
