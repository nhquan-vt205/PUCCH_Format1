<!-- OVERLAY: DOC. Ap len _PLAN_CORE.md. KHONG sua file nay. -->

# Overlay: `DOC`

## Khi nào dùng

Viết / cập nhật / tổng hợp **tài liệu**: `doc/<module>.md` cho module, `doc/<ip>.md` cho cả IP,
`ddoc/<m>_req.md` khi tôi yêu cầu soạn requirement, README / mô tả reg-map.

Dấu hiệu: "viết doc cho…", "tổng hợp tài liệu IP", "doc đang lệch với RTL", "mô tả reg-map".
Nếu task là **sửa RTL** rồi mới cập nhật doc kèm theo → đó là `IMPLEMENT`, doc là một step trong đó.

## §1 Metadata — giá trị mặc định

| Field | Value |
|---|---|
| Task type | `DOC` |
| Flow phase(s) | `rtl-documenter` (chiều ngược: RTL có trước, sinh doc sau) |
| Skills / Agents | `rtl-documenter` — dùng khi repo đã có RTL nhưng thiếu/doc lệch; đọc theo `ModuleDocContract.md` |
| Related artifacts | `doc/<m>.md`, `doc/<ip>.md`, `ddoc/<m>_req.md`, `ddoc/<m>_proposal.md`, `~/.claude/skills/shared/ModuleDocContract.md` |

## Đổi tên trong core

- §2.5 `Requirements` → requirement ở đây là **mục nào của doc contract phải có**, trace về
  `ModuleDocContract.md` và về §2.1.
- §5 `Implementation Plan` → **`Documentation Plan`**

## Chèn thêm

### §2.9 Doc contract targets

| Mục theo `ModuleDocContract.md` | Có trong doc hiện tại? | Sẽ viết / sửa |
|---|---|---|
| Overview / chức năng | | |
| Port list (dir, width, mô tả) | | |
| Parameter list | | |
| Reg-map (nếu có CPU bus) | | |
| Clock / reset | | |
| FSM / timing diagram | | |
| Usage / integration note | | |

Mục tiêu: doc **đủ để dùng module mà không cần đọc RTL**.

### §4.7 Source of truth precedence

Khi các nguồn lệch nhau, thứ tự tin cậy để viết doc:

```text
RTL thực tế  >  ddoc/<m>_proposal.md  >  ddoc/<m>_req.md
```

- Doc phải mô tả **RTL đang có**, không mô tả ý định.
- Phát hiện RTL lệch `_req.md` → **không sửa RTL, không sửa doc cho khớp ý định**: ghi vào bảng dưới,
  báo tôi, và đề xuất plan `DEBUG`/`IMPLEMENT` riêng.

| # | Điểm lệch | RTL nói gì (`file:line`) | `_req`/`_proposal` nói gì | Xử lý |
|---|---|---|---|---|
| X1 | | | | báo tôi / follow-up plan |

### §5.4 Doc file map

| Doc file | Module / IP | New / Modify | Nguồn nội dung |
|---|---|---|---|
| `doc/<m>.md` | `rtl/<m>.v` | | RTL + proposal |
| `doc/<ip>.md` | cả IP | | gộp từ `doc/<m>.md` (`rtl-documenter`) |

### §6.8 Doc ↔ RTL consistency matrix

Kiểm tra bằng cách **đối chiếu thật**, không bằng cảm giác.

| Hạng mục | Cách đối chiếu | Kết quả | Status |
|---|---|---|---|
| Port list (tên, dir, width) | so với `module` header trong RTL | | PENDING |
| Parameter (tên, default) | so với khai báo `parameter` | | PENDING |
| Reg-map (offset, field, reset) | so với `dut.vh` / decode logic | | PENDING |
| Reset polarity / đồng bộ | so với always block reset | | PENDING |
| Hành vi mô tả | so với FSM / datapath | | PENDING |

## Acceptance bắt buộc thêm

- [ ] Mọi module trong scope có `doc/<module>.md` (module trong `lib/` có `<module>.md` cùng thư mục)
- [ ] Doc theo đúng `ModuleDocContract.md`
- [ ] §6.8 đối chiếu xong, không còn sai lệch chưa giải thích
- [ ] **Không sửa** `rtl/` `lib/` `tb/` trong plan này
- [ ] Điểm lệch RTL vs `_req.md` đã báo tôi (§4.7), không tự chọn bên nào
- [ ] `doc/<ip>.md` (nếu có) nhất quán với các `doc/<m>.md` thành phần

## FE Flow Impact mặc định

| Stage | Required? | Reason |
|---|---:|---|
| RTL | NO | DOC không sửa RTL |
| Testbench | NO | |
| Simulation / Lint / CDC / Synthesis / Timing / Area / Power | NO | không đổi RTL nên không cần chạy lại |
| Documentation | YES | chính là task này |

## Cấm

- Không sửa RTL để khớp doc.
- Không viết doc theo `_req.md` khi RTL làm khác — mô tả RTL thật, và báo điểm lệch.
- Không ghi "đã kiểm tra khớp" khi chưa mở file RTL ra đối chiếu.
