# HyperWork Silentium Gen 2 — Web Driver

Web driver (chạy trên trình duyệt, dùng WebHID) cho chuột HyperWork Silentium Gen 2 (MS01G2).

**Trạng thái:** đang research — chưa có giao thức của thiết bị. Xem [docs/RESEARCH.md](docs/RESEARCH.md).

## Cấu trúc

- `docs/RESEARCH.md` — tổng hợp thông tin thiết bị, nền tảng WebHID, kiến trúc đề xuất, quy trình reverse-engineer.
- `tools/hid-probe/` — trang WebHID để dò HID descriptor, đọc feature report, log input report và export JSON.
- `docs/PROTOCOL.md` — phân tích descriptor và giao thức đã biết.
- `captures/` — JSON descriptor / pcap bắt được từ thiết bị thật.

## Chạy hid-probe

```sh
python3 -m http.server 8000
# mở http://localhost:8000/tools/hid-probe/ bằng Chrome hoặc Edge desktop
```

Bấm **Kết nối thiết bị…** và chọn một mục của chuột (hộp chọn của Chrome chỉ cho chọn một mục mỗi lần, mỗi mục đã gồm mọi interface của thiết bị đó). Nếu có nhiều mục (ví dụ "2.4G Wireless Receiver" và "2.4G Mouse"), bấm kết nối lại để thêm từng mục, rồi **Export JSON** (xuất tất cả) và lưu vào `captures/`.
