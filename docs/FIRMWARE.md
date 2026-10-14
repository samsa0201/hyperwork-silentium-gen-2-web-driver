# Phân tích firmware `firmware.bin` (MS01G2UPDATE.zip)

> Phân tích tĩnh bằng capstone (ARM Thumb-2). Địa chỉ dưới đây là địa chỉ chạy (file offset + `0x403000`).
> Các nhận định đánh dấu *(giả thuyết)* cần kiểm chứng trên chuột thật.

## Toàn vẹn & khả năng patch

- `firmware.dat` (24 byte) **không có chữ ký số**. Các trường (u32 LE):
  `0x01bfdf55` (magic), `3`, `0x00010000`, **`0xd22e0000` → u16 tại offset 14 = CRC16-XMODEM của toàn bộ `firmware.bin`**,
  `0x6621e100` (mã chip), `0x00013550` (kích thước bin).
- Trong `firmware.bin` không thấy checksum riêng. → Sửa bin rồi cập nhật CRC16 (và kích thước nếu đổi) trong dat là đủ để
  gói hợp lệ về mặt định dạng.
- Ảnh firmware: vector table ở đầu, chạy tại `0x00403000`; bảng scatter-load tại `0x4030d0`:

| Nguồn (flash) | Đích (RAM) | Dài |
|---|---|---|
| `0x41025c` | `0x200000` | `0xcc` |
| `0x410328` | `0x2000cc` | `0x80c` |
| `0x410b34` | `0x2008d8` | `0x594c` |
| `0x416480` | `0x20006230` | `0xd0` |
| — (zero) | `0x20006300` | `0x23c8` |

  Đổi địa chỉ RAM `0x2008d8…0x206224` về file: `file = addr − 0x2008d8 + 0x410b34 − 0x403000`.
- Các lời gọi `bl 0x61xxxx` / `0x32xxxx` là hàm trong ROM của chip (ngoài ảnh firmware).

## HID qua BLE

Report map tại `0x40f974` (file `0xc974`): ID 1 chuột (8 nút, X/Y 12-bit, wheel, AC Pan), ID 2 bàn phím, ID 3 consumer
(16-bit), ID 4 system control (sleep/power).

Hàm gửi report `0x411568(type, data)`: type 0 = nút (2 byte: nút, nghiêng), 1 = X/Y, 2 = wheel/pan (cộng dồn),
3 = bàn phím (9 byte), 4–6 = 16 byte.

## Xử lý nút — `0x41388c(mask)`

- `os = rom_0x61a75c()`; giá trị **4 và 6 là thiết bị Apple** *(giả thuyết: 4 = macOS, 6 = iOS/iPadOS)*.
- Bit nút: `0x01` trái, `0x02` phải, `0x04` giữa, `0x08`/`0x10` nút 4/5 (Back/Forward) *(giả thuyết)*,
  `0x20`/`0x40` nghiêng con lăn trái/phải (đảo chiều trên Apple).
- Trên Apple, bit `0x08`/`0x10` không gửi nút 4/5 mà phát **macro bàn phím** (xoá bit rồi gọi `0x415db4(macro, 4)`):
  macOS → `Cmd+[` / `Cmd+]`; os 6 → `Cmd+;` / `Cmd+'`.
- **Nút giữa (`0x04`) trong bản firmware này đi thẳng ra nút chuột 3** → trên Mac là "mở tab mới". Bản cũ (gửi
  Mission Control) có lẽ phát macro `Ctrl+↑` ở đây — macro đó vẫn còn trong bảng.

## Bảng macro (RAM `0x206066`, file `0x0132c2`, mỗi macro 40 byte = 4 bước × 10 byte)

Bước = `[len][type=3 bàn phím][modifier][0][key]…`; 4 bước: nhấn modifier → nhấn phím → nhả phím → nhả hết.

| RAM | Phím | Dùng ở |
|---|---|---|
| `0x206066` | GUI + `0x33` (`;`) | nút 4, os 6 |
| `0x20608e` | GUI + `0x34` (`'`) | nút 5, os 6 |
| `0x2060b6` | GUI + `[` | nút 4, macOS (Back) |
| `0x2060de` | Ctrl + ↑ | action 8 / 29 (Mission Control) |
| `0x206106` | GUI + `]` | nút 5, macOS (Forward) |
| `0x20612e` | Ctrl+GUI + `F` | action 15 trên Apple (toàn màn hình) |
| `0x206156` | GUI + `D` | action 9 / 29 (Windows: Show Desktop) |
| `0x20617e` | Shift+GUI + `S` | dispatcher (Windows: chụp màn hình) |
| `0x2061a6` | GUI + Tab | action 8 trên Windows (Task View) |

## Bảng tổ hợp / nhấn giữ (RAM `0x205fe0`, file `0x01323c`, 7 mục × 12 byte)

`[mask u32][kiểu u16: 1/2 = nhấn, 3 = giữ][thời gian giữ u16, tick ≈ 8 ms][action u8][pad 3]`

| # | Mask | Kiểu | Giữ | Action | Ghi chú |
|---|---|---|---|---|---|
| 0 | `0x080` | 2 | 12 | 3 | *(giả thuyết: nút DPI)* |
| 1 | `0x100` | 1 | 0 | 6 | nút kết nối |
| 2 | `0x100` | 2 | 0 | 2 | nút kết nối |
| 3 | `0x100` | 3 | 375 (~3 s) | 1 | giữ nút kết nối 3 s → ghép nối (khớp hướng dẫn PDF) |
| 4 | `0x006` | 3 | 375 | 14 | giữ phải + giữa 3 s |
| 5 | `0x200` | 2 | 0 | 26 | |
| 6 | `0x800` | 2 | 12 | 9 | |

## Bộ phân phối action `0x413f6c(action)`

Bảng nhảy `tbh` 29 mục tại `0x413f7c`. Đã biết: action 8 = Mission Control (`Ctrl+↑`) trên Mac / Task View
(`Win+Tab`) trên Windows; action 1 = vào ghép nối. Các action 5, 11, 17, 21–23 không làm gì.

## Hướng patch để đổi nút (chưa thực hiện)

1. **Chỉ sửa dữ liệu (rủi ro thấp nhất):** đổi phím trong macro (ví dụ nút 4/5 trên Mac), hoặc đổi mã action trong
   bảng tổ hợp. Code giữ nguyên.
2. **Sửa code nhỏ:** trong `0x41388c`, cho nút giữa (hoặc nút khác) phát macro / action thay vì nút chuột.
3. Sau khi sửa: tính lại CRC16-XMODEM vào `firmware.dat[14..15]`, đóng gói zip, nạp bằng `tools/ble-ota`.

Rủi ro: DFU chạy trong chính firmware ứng dụng. Nếu bản patch khởi động được nhưng hỏng logic BLE, có thể không còn
cập nhật lại được qua Bluetooth (chưa biết chip có bootloader dự phòng hay không).
