# HyperWork Silentium Gen 2 — Web Driver

Web driver (chạy trên trình duyệt, dùng Web Bluetooth / WebHID) cho chuột HyperWork Silentium Gen 2 (MS01G2).

**Trạng thái:** đã có trang cập nhật firmware qua Web Bluetooth (`tools/ble-ota/`, dựa trên dịch ngược công cụ OTA
chính hãng — xem [docs/OTA.md](docs/OTA.md)); đã kiểm thử với thiết bị giả lập, **chưa thử trên chuột thật**.
Cấu hình DPI/nút lúc chạy vẫn chưa có giao thức. Xem thêm [docs/RESEARCH.md](docs/RESEARCH.md).

## Cấu trúc

- `docs/OTA.md` — phân tích gói cập nhật `MS01_G2_update.rar`: chip OnMicro OM6621, giao thức Nordic Secure DFU qua BLE.
- `docs/FIRMWARE.md` — dịch ngược firmware: CRC của gói, xử lý nút, bảng macro, bảng tổ hợp nút → action (nền tảng cho việc patch đổi nút).
- `tools/ble-ota/` — trang Web Bluetooth cập nhật firmware (thay BLEOTATool chỉ có trên Windows).
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

## Chạy ble-ota (cập nhật firmware)

```sh
python3 -m http.server 8000
# mở http://localhost:8000/tools/ble-ota/ bằng Chrome hoặc Edge desktop
```

1. Giải nén `MS01_G2_update.rar` của HyperWork, chọn file `MS01G2UPDATE.zip` trong trang.
2. Chuyển chuột sang Bluetooth, giữ nút kết nối 3 giây (chế độ ghép nối, **không** ghép với máy).
3. **Chọn chuột…** → chọn đúng tên chuột → tick xác nhận → **Bắt đầu cập nhật**.

Trên macOS cần cho Chrome quyền Bluetooth (System Settings → Privacy & Security → Bluetooth).
