<!-- OVERLAY: VERIFY. Ap len _PLAN_CORE.md. KHONG sua file nay. -->

# Overlay: `VERIFY`

## Khi nào dùng

Xây hoặc sửa **verification**: scaffold bench, viết/sửa testcase, stimulus driver, scoreboard,
assertion, coverage, directed / random test, chạy regression.

Dấu hiệu: "viết testbench…", "thêm testcase cho…", "cover trường hợp…", "chạy regression".
Nếu mục tiêu là **sửa bug** thì đó là `DEBUG`; nếu là **sửa RTL** thì `IMPLEMENT`/`REFACTOR`.

## §1 Metadata — giá trị mặc định

| Field | Value |
|---|---|
| Task type | `VERIFY` |
| Flow phase(s) | Stage 08 Verification Plan · Stage 09 Testbench |
| Skills / Agents | `vplan-designer` (verification plan, mỗi dòng một test dự kiến) · `tb-generator` (scaffold bench + testcase theo vplan) · **chạy test = MANUAL, không có skill** |
| Related artifacts | `tb/<ip>/{Makefile,script/,clock.vh,dut.vh}`, `sti_<iface>.sv`, `sco_<iface>.sv`, `tb_<name>.sv`, `tc_list.md`, `issue/<ip>/run_summary.md` |

## Đổi tên trong core

- §2.5 `Requirements` → requirement ở đây là **điều cần được verify** (trace về `_req.md` / `doc/`),
  không phải chức năng mới.

## Chèn thêm

### §2.9 Test strategy & coverage goals

- Mức test: unit (`tb/<module>/`) hay IP-level (`tb/<ip>/`) — và vì sao.
- Cơ chế điều khiển: bench là **CPU-bus-only** (testcase chỉ được drive qua CPU BFM, không chọc
  trực tiếp tín hiệu nội bộ) — xác nhận tuân thủ hoặc nêu ngoại lệ.
- Self-check bằng gì: scoreboard `sco_<iface>`, expected value tính trong TC, assertion.
- Mục tiêu cover: chức năng nào, edge case nào, corner reset/back-pressure/overflow nào.
- Random hay directed; nếu random thì seed được ghi lại thế nào.

### §4.7 Bench architecture

| Thành phần | File | Vai trò | New / Modify |
|---|---|---|---|
| clock / reset gen | `clock.vh` | | |
| reg-map + DUT + CPU BFM | `dut.vh` | | |
| stimulus | `sti_<iface>.sv` | | |
| scoreboard | `sco_<iface>.sv` | | |
| testcase | `tb_<name>.sv` | | |
| filelist | `script/rtl.f` | | |

### §5.4 Testcase list changes

| Testcase | New / Modify | Requirement cover | Verdict token |
|---|---|---|---|
| `tb_<name>.sv` | New | REQ-001 | `[FINISH] PASS` |

`tc_list.md` sẽ được cập nhật: [ ] có

### §6.8 Coverage matrix

| Requirement | TC | Loại (directed/random) | Edge case cover | Status |
|---|---|---|---|---|
| REQ-001 | TC-001 | directed | | PENDING |

Requirement nào **chưa** có TC → ghi rõ ở đây, đừng để trống rồi coi như xong.

## Acceptance bắt buộc thêm

- [ ] Mỗi TC **self-check**, không cần tôi đọc waveform mới biết pass/fail
- [ ] Mỗi TC kết thúc bằng đúng token `[FINISH] PASS` / `[FINISH] FAIL`
- [ ] TC chỉ điều khiển qua CPU bus (hoặc ngoại lệ đã khai ở §2.9 và tôi duyệt)
- [ ] Mỗi TC mới đã **chạy thật** và có log (không chỉ compile) — **MANUAL**, tôi tự chạy và dán log
- [ ] `tc_list.md` cập nhật đúng trạng thái từng TC
- [ ] Không sửa `rtl/` trong plan này (nếu buộc phải sửa → dừng, mở plan `DEBUG`/`IMPLEMENT`)
- [ ] TC mới **fail đúng cách** khi DUT sai (đã thử làm nó fail, hoặc giải thích vì sao không thử được)

## FE Flow Impact mặc định

| Stage | Required? | Reason |
|---|---:|---|
| RTL | NO | VERIFY không sửa RTL |
| Testbench | YES | chính là task này |
| Simulation | YES | TC phải được chạy thật — **MANUAL**, không skill nào tự chạy |
| Lint | MAYBE | YES nếu lint có cover `tb/` |
| CDC / Synthesis / Timing / Area / Power | NO | không đổi RTL |
| Documentation | MAYBE | YES nếu `tc_list.md` / doc test cần cập nhật |

## Cấm

- Không sửa RTL để TC pass.
- Không viết TC "luôn pass" (không có self-check, hoặc check điều luôn đúng).
- Không đánh dấu TC PASS khi chỉ compile được.
