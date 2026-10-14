# HyperWork Silentium Gen 2 — Web Driver

Web driver (chạy trên trình duyệt, dùng WebHID) cho chuột HyperWork Silentium Gen 2 (MS01G2).

**Trạng thái:** đang research — chưa có giao thức của thiết bị. Xem [docs/RESEARCH.md](docs/RESEARCH.md).

## Cấu trúc

- `docs/RESEARCH.md` — tổng hợp thông tin thiết bị, nền tảng WebHID, kiến trúc đề xuất, quy trình reverse-engineer.
- `tools/hid-probe/` — trang WebHID để dò HID descriptor, đọc feature report, log input report và export JSON.
- `captures/` — nơi lưu JSON descriptor / pcap bắt được từ thiết bị thật.

## Chạy hid-probe

```sh
python3 -m http.server 8000
# mở http://localhost:8000/tools/hid-probe/ bằng Chrome hoặc Edge desktop
```

Bấm **Kết nối thiết bị…**, chọn receiver của chuột (chọn tất cả interface), rồi **Export JSON** và lưu vào `captures/`.
