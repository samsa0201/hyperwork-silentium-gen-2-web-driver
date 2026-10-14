# OTA firmware update — phân tích gói `MS01_G2_update.rar`

> Nguồn: `https://cdn.shopify.com/s/files/1/0591/7006/2395/files/MS01_G2_update.rar?v=1768379874` (HyperWork phát hành,
> file ngày 2026-01-14). SHA-256 của .rar: `6d474ca682d98f21ef5d5c757fef24bd1414df38dbe5fc07a04a460b7b562d4c`.
> Chỉ phân tích tĩnh (giải nén, dịch ngược .NET bằng ILSpy) — không chạy file .exe. **Không commit firmware của hãng vào repo.**

## Nội dung gói

| File | Ghi chú |
|---|---|
| `Hướng dẫn Upgrade Firmware MS01.G2 (Silentium Gen 2) .pdf` | 7 trang hướng dẫn |
| `MS01G2UPDATE.zip` | `manifest.json` + `firmware.dat` (24 byte) + `firmware.bin` (79 184 byte) |
| `OTATOOL Update/BLEOTATool.exe` (+ `.pdb`, `.config`) | .NET Framework 4.8 WPF, "OTAToolForWindow10" v1.0.0.5, comment "6621x OTA", icon `onmicro_logo.ico` |
| `Newtonsoft.Json.dll`, `Windows.winmd` | thư viện |

**Mục đích bản cập nhật (theo PDF):** đổi chức năng Middle Click khi dùng trên **macOS** từ "Mission Control" sang
"Open New Tab" (giống trên Windows). Quy trình: chuyển chuột sang Bluetooth → giữ nút kết nối 3 s để vào chế độ ghép nối
(**không** ghép với máy) → mở BLEOTATool, chọn file zip, chọn thiết bị, Start Upgrade.

## Phát hiện chính

- **Chip của chuột: OnMicro OM6621x** (SoC BLE, ARM Cortex-M). Firmware là ảnh ARM Thumb, vector table ở đầu file:
  SP = `0x200086c8`, reset = `0x00403055` → firmware chạy tại `0x00403000`. Tại offset `0x34` có giá trị `0x6621e100`
  (trùng trong `firmware.dat`) — nhiều khả năng là mã chip/biến thể.
- **Cập nhật qua Bluetooth LE, không qua receiver USB.** Công cụ dùng giao thức **Nordic Secure DFU** (thư viện
  `Plugin.XamarinNordicDFU` được port sang WinRT):

| GATT | UUID |
|---|---|
| DFU service | `0000fe59-0000-1000-8000-00805f9b34fb` |
| Control Point (write + notify) | `8ec90001-f315-4f60-9fb8-838830daea50` |
| Packet (write without response) | `8ec90002-f315-4f60-9fb8-838830daea50` |
| Buttonless DFU (khai báo, **không dùng**) | `8ec90003-f315-4f60-9fb8-838830daea50` |
| Device Information | `0x180A`: Model `2A24`, Serial `2A25`, Firmware `2A26`, Hardware `2A27`, Software `2A28`, System ID `2A23` |

- Firmware ứng dụng của chuột phơi service DFU ngay ở chế độ ghép nối — không cần lệnh chuyển sang bootloader.
- **Web Bluetooth không chặn `0xFE59`** (blocklist chỉ chặn Nordic *legacy* DFU `00001530-…`, HID `0x1812`, và
  characteristic Serial Number `2A25`). → Có thể cập nhật firmware từ trình duyệt, kể cả trên macOS.

## Định dạng gói firmware

`manifest.json`: `{"manifest":{"application":{"bin_file":"firmware.bin","dat_file":"firmware.dat"}}}`.

`firmware.dat` (init packet, không phải protobuf của Nordic — định dạng riêng của OnMicro):

```
55 df bf 01  03 00 00 00  00 00 01 00  00 00 2e d2  00 e1 21 66  50 35 01 00
                                                    ^ 0x6621e100 ^ 0x00013550 = 79184 = kích thước firmware.bin
```

CRC32 (IEEE): init `0x909cd6f2`, firmware `0xefaa88b9`.

## Trình tự DFU của BLEOTATool (đã đọc từ mã dịch ngược)

Mọi lệnh ghi lên Control Point (write with response); phản hồi qua notify dạng `60 <opcode> <status> …`, `status = 01` là
thành công; timeout 5 s mỗi lệnh. Dữ liệu ghi lên Packet (write without response), mỗi gói `MTU − 3` byte, nghỉ 1 ms.

| Opcode | Lệnh | Tham số | Phản hồi thêm |
|---|---|---|---|
| `01` | Create | type (1 = command, 2 = data), size u32 LE | — |
| `02` | Set PRN | u16 LE (công cụ đặt 0) | — |
| `03` | Calculate checksum | — | offset u32, CRC32 u32 |
| `04` | Execute | — | — |
| `06` | Select | type | maxSize u32, offset u32, CRC32 u32 |

1. **Init packet:** `06 01` (Select command) → nếu thiết bị đã có tiền tố hợp lệ thì tiếp tục/bỏ qua → `02 00 00` →
   `01 01 <size>` → gửi `firmware.dat` → `03` so CRC (thử lại tối đa 3 lần) → `04`.
2. **Firmware:** `06 02` (Select data, lấy `maxSize`) → `01 02 min(size, maxSize)` **một lần duy nhất** → nghỉ 400 ms →
   lặp theo từng khối `maxSize`: gửi dữ liệu → nghỉ 400 ms → `03` so CRC lũy kế → `04`. Hết file thì thành công;
   chuột tự khởi động lại và ngắt kết nối.

Khác với Nordic chuẩn, công cụ **không** gửi Create cho các khối sau khối đầu. Trang web làm theo đúng trình tự của
công cụ gốc (đã được hãng dùng với chính file này), chỉ chặt hơn ở chỗ dừng ngay khi CRC sai thay vì lặp tiếp.

## Hệ quả cho web driver

- `tools/ble-ota/` — trang Web Bluetooth cập nhật firmware theo trình tự trên. Đã kiểm thử với thiết bị BLE giả lập
  (Playwright): init packet và firmware nhận đúng từng byte, chuỗi opcode khớp công cụ gốc. **Chưa thử trên chuột thật.**
- Web Bluetooth không cho biết MTU, nên mặc định gói 20 byte (an toàn với MTU 23); có thể chọn gói lớn hơn.
- Receiver 2.4 GHz (`32c2:0012`, chip HS6209) không tham gia cập nhật; collection vendor `0xFFB5` của nó vẫn chưa rõ
  công dụng. Khi bấm nút trên chuột, receiver không gửi gì qua các collection mà WebHID đọc được.
- Firmware không chứa bảng DPI dạng số rõ ràng (1000/1600/…); tuỳ chỉnh DPI/nút lúc chạy vẫn chưa có bằng chứng —
  hiện cách duy nhất hãng dùng để đổi chức năng nút là nạp firmware khác.
