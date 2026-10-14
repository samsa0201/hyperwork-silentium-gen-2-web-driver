# Protocol notes — Silentium Gen 2

> Trạng thái: mới có HID descriptor của receiver; **chưa biết lệnh nào**. Mọi điều dưới mục "Giả thuyết" cần kiểm chứng.

## Receiver 2.4 GHz — `32c2:0012` "2.4G Wireless Receiver"

Nguồn: [`captures/receiver-32c2-0012.json`](../captures/receiver-32c2-0012.json) (Chrome / macOS, 2026-10-14).

| Top-level collection | Report ID | Hướng | Kích thước | Nội dung | WebHID |
|---|---|---|---|---|---|
| `0xFFB5:0x0001` (vendor) | `0xB5` | in + out | 7 byte | 7 × u8, usage `0xFFB5:0x0002` | **Dùng được** — kênh lệnh cấu hình |
| `0x0001:0x0002` Mouse → Pointer | `0x02` | in | 6 byte | 8 nút, X/Y 12-bit (±2047), wheel s8, AC Pan s8 | Bị chặn |
| `0x000C:0x0001` Consumer | `0x01` | in | 2 byte | 1 usage consumer 16-bit (0x001–0x3FF) | Dùng được |
| `0xFFBC:0x0088` (vendor) | `0x04` | in | 1 byte | mảng 1 usage `0xFFBC:0x00–0xFF` | Dùng được |

Không có feature report nào → mọi trao đổi đi qua **output report `0xB5` (gửi) và input report `0xB5` (nhận)**.

Bố cục report chuột `0x02` (sau byte report ID):

```
byte 0     : nút 1–8 (bitmask)
byte 1..3  : X (12 bit) | Y (12 bit), little-endian
byte 4     : wheel (s8)
byte 5     : AC Pan / cuộn ngang (s8)
```

## Chip

- VID `0x32c2` được ghi nhận ở thiết bị khác với tên **"HS6209 2.4G Wireless Receiver / Mouse"** (lsusb của người dùng
  trên V2EX, linux-hardware.org cho `32c2:0010`). HS6209 = OnMicro OM6209: 8051, 2.4 GHz GFSK, USB, **16 KB OTP**.
- Nếu receiver đúng là HS6209 thì firmware receiver **không cập nhật được** (OTP). Chuột Gen 2 có Bluetooth nên MCU
  của chuột gần như chắc chắn là chip khác (HS6209 không có BLE) — công cụ cập nhật của hãng nhiều khả năng nạp
  firmware cho **chuột**, có thể chuyển tiếp qua receiver bằng report `0xB5`.

## Giả thuyết (chưa kiểm chứng)

1. Gói lệnh = 1 byte report ID `0xB5` + 7 byte payload; chuột/receiver trả lời bằng input `0xB5` cùng độ dài.
2. Report `0x04` (`0xFFBC`) là sự kiện phím đặc biệt (ví dụ nút DPI, chuyển kênh) hoặc trạng thái kết nối.
3. Report Consumer `0x01` mang zoom in/out (AC Zoom In `0x22D` / Zoom Out `0x22E`) khi nghiêng/nhấn nút giữa.

## Việc cần làm

- [ ] Export descriptor của mục "2.4G Mouse" (và chế độ cắm cáp nếu có).
- [ ] Chỉ nghe (không gửi): ghi log input `0xB5`, `0x04`, `0x01` khi bấm nút DPI, nghiêng con lăn, nhấn nút giữa,
      tắt/bật chuột, đổi kênh 2.4G/BT, cắm sạc.
- [ ] Bắt pcap công cụ cập nhật firmware chính hãng trên Windows (USBPcap + Wireshark; lọc theo
      `usb.device_address` của receiver, tìm các gói 8 byte bắt đầu bằng `b5`) để có các lệnh `0xB5` thật.
