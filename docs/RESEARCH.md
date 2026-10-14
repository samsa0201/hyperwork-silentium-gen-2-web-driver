# Research: Web driver cho chuột HyperWork Silentium Gen 2

> Cập nhật: 2026-10-14. Trạng thái: **giai đoạn thu thập thông tin, chưa có giao thức (protocol) của thiết bị**.

## 1. Tóm tắt

- Làm web driver cho Silentium Gen 2 là **khả thi về mặt nền tảng**: trình duyệt Chromium (Chrome, Edge, Opera) có
  WebHID, cho phép trang web nói chuyện với *collection HID vendor-defined* (usage page `0xFF00`–`0xFFFF`) của
  receiver hoặc chuột. Đây là cách các "web hub" của ATK, VXE, MCHOSE… đang làm.
- Điểm nghẽn **không phải là web**, mà là **giao thức cấu hình của chuột chưa được công bố**. HyperWork hiện không
  có phần mềm cấu hình (DPI, gán phím…) cho Gen 2; họ chỉ phát hành **công cụ cập nhật firmware cho Windows** kèm
  hướng dẫn **chuyển đổi chức năng Middle Click**. Việc đổi chức năng bằng cách nạp firmware gợi ý rằng
  firmware **có thể không có lệnh cấu hình lúc chạy** — nếu đúng vậy thì web driver chỉ làm được: đọc thông tin
  thiết bị, (có thể) đọc pin, và nạp firmware.
- Vì vậy bước tiếp theo bắt buộc là **reverse-engineer trên thiết bị thật**: dò HID descriptor (đã có công cụ
  `tools/hid-probe`), phân tích công cụ firmware chính hãng, và bắt gói USB khi công cụ chạy.

## 2. Thông tin thiết bị đã biết

| Mục | Giá trị | Nguồn / độ tin cậy |
|---|---|---|
| Tên / mã | Silentium Gen 2, mã `MS01G2` (`HPW-MS01G2-BLK`, `-WHT`) | Trang hãng + nhà bán lẻ — cao |
| Cảm biến | SG8920 (quang học) | Trang hãng + techspace — trung bình (không tìm thấy datasheet) |
| DPI | 1000 / 1600 / 2400 / 3200 / 4800 (đổi bằng nút) | Trang hãng — cao |
| Kết nối | 2.4 GHz (receiver USB-A và USB-C) + Bluetooth (nhà bán lẻ ghi 5.0), tối đa 3 thiết bị | Nhà bán lẻ — trung bình |
| Cáp | Có cáp trong hộp — **chưa rõ cáp chỉ để sạc hay cũng truyền dữ liệu (chế độ có dây)** | Thế Giới Di Động — cần kiểm tra |
| Pin | Sạc lại, ~70 ngày / "tới 2,5 tháng" | Blog hãng, Hacom |
| Con lăn | Có nghiêng trái/phải (cuộn ngang); "Air Spin"; middle click trái/phải = zoom out/in | Blog hãng + nhà bán lẻ |
| Phần mềm chính hãng | Công cụ **cập nhật firmware (chỉ Windows)** + hướng dẫn **chuyển đổi chức năng Middle Click** | Trang sản phẩm hyperwork.vn |
| Tham chiếu dòng cũ | Silentium Mini MS01M có file `HyperWork_MS01M_Driver_Upgrade.rar` (~345 KB) tại help.hyperwork.vn | help.hyperwork.vn |

**Chưa biết (cần thiết bị thật):** USB VID/PID của receiver và chuột, HID report descriptor, MCU/SoC
(thường gặp ở chuột văn phòng 3-mode: Beken BK34xx/BK36xx, Telink TLSR82xx, Compx, Nordic…), polling rate,
giao thức lệnh, cơ chế bootloader.

> Ghi chú: trong môi trường research này, hyperwork.vn và help.hyperwork.vn bị chặn bởi network policy, nên
> chưa tải và phân tích được công cụ firmware. Thông tin trên lấy từ trích đoạn kết quả tìm kiếm của trang hãng
> và nhà bán lẻ.

## 3. Nền tảng kỹ thuật cho web driver

### 3.1 WebHID — lựa chọn chính

- Hỗ trợ: Chrome / Edge / Opera **desktop** (Windows, macOS, Linux, ChromeOS). **Không** có trên Firefox, Safari,
  và trình duyệt di động.
- Yêu cầu: trang chạy trong secure context (HTTPS hoặc `localhost`); `navigator.hid.requestDevice()` phải được gọi
  từ thao tác người dùng (click).
- **Collection được bảo vệ:** Chrome chặn mọi report của top-level collection mang usage chuột/bàn phím
  (Generic Desktop `0x01` / Mouse `0x02`, Keyboard `0x06`, …). Nghĩa là web driver **không đọc được chuyển động
  chuột**, nhưng vẫn dùng được collection vendor-defined riêng — hầu hết chuột có phần mềm đều có collection này
  để nhận lệnh cấu hình/cập nhật firmware.
- Linux: cần quyền đọc/ghi `hidraw`, ví dụ `/etc/udev/rules.d/70-hyperwork.rules`:
  ```
  KERNEL=="hidraw*", ATTRS{idVendor}=="XXXX", ATTRS{idProduct}=="YYYY", MODE="0660", TAG+="uaccess"
  ```
  (điền VID/PID khi đã biết).
- API dùng: `device.collections` (descriptor đã parse), `sendReport` / `sendFeatureReport` /
  `receiveFeatureReport`, sự kiện `inputreport`, `navigator.hid` `connect`/`disconnect`.

### 3.2 Các hướng khác và lý do loại

| Hướng | Vì sao không phù hợp |
|---|---|
| WebUSB | Interface HID đã bị driver HID của OS chiếm; Chrome chặn class HID trên WebUSB. |
| Web Bluetooth | Service HID over GATT (`0x1812`) nằm trong blocklist của Chrome. |
| WebHID qua Bluetooth | Có thể thấy thiết bị BLE HID trên một số nền tảng, nhưng collection vendor qua BLE không ổn định giữa các OS — chỉ nên thử sau. |

**Kết luận:** ưu tiên kết nối qua **receiver 2.4 GHz** (receiver thường chuyển tiếp lệnh vendor tới chuột) và qua
**cáp** nếu cáp có truyền dữ liệu.

## 4. Kiến trúc đề xuất

```
UI (cấu hình DPI, nút, pin, firmware)
  └─ Device model      : trạng thái + thao tác có kiểu (setDpiStage, getBattery, …)
       └─ Protocol      : mã hoá/giải mã gói lệnh (header, cmd id, length, payload, checksum)
            └─ Transport : WebHID (thật) | Mock (test không cần phần cứng) | Replay (phát lại pcap)
```

- Stack gợi ý: Vite + TypeScript, UI nhẹ (Svelte hoặc React), deploy tĩnh lên GitHub Pages (có sẵn HTTPS).
- Protocol tách riêng, thuần hàm, test bằng các gói bắt được (fixtures) — không cần chuột để chạy CI.
- Mọi lệnh **ghi** (đặc biệt là flash / vào bootloader) phải được khai báo rõ ràng trong code, có xác nhận
  từ người dùng và không bao giờ nằm trong luồng "thử nghiệm" tự động.

## 5. Quy trình reverse-engineer (cần chuột + máy Windows)

1. **Dò descriptor** — mở `tools/hid-probe/index.html` qua `localhost` bằng Chrome/Edge, kết nối receiver (rồi chuột
   qua cáp), bấm *Export JSON* và commit vào `captures/`. Trên Linux có thể bổ sung:
   `lsusb -v -d VID:PID`, `sudo usbhid-dump -d VID:PID`, hoặc `hid-decode /dev/hidrawN` (gói `hid-tools`).
2. **Phân tích công cụ chính hãng** — tải từ trang Gen 2 trên hyperwork.vn / help.hyperwork.vn. Giải nén,
   chạy `strings` / Detect It Easy; nếu là .NET dùng dnSpy/ILSpy, nếu native dùng Ghidra. Tìm: VID/PID,
   report ID, độ dài gói, chuỗi tên chip (`BK`, `TLSR`, `CX`, …), định dạng file firmware.
3. **Bắt gói USB** — USBPcap + Wireshark trên Windows trong khi chạy công cụ cập nhật / chuyển Middle Click.
   Lọc `usbhid` và `usb.transfer_type == 0x02` (control: SET_REPORT/GET_REPORT) hoặc interrupt OUT.
   Bắt thêm khi bấm nút đổi DPI và khi pin yếu để xem chuột có tự báo trạng thái không.
4. **Giải mã cấu trúc lệnh** — so sánh (diff) các lần bắt; xác định lệnh *đọc* (version, pin, cấu hình) trước.
5. **Kiểm chứng an toàn** — chỉ phát lại lệnh *đọc* bằng hid-probe. **Không** phát lại lệnh xoá flash /
   vào bootloader khi chưa hiểu rõ và chưa có cách phục hồi.

## 6. Phạm vi tính năng theo mức độ

| Mức | Tính năng | Điều kiện |
|---|---|---|
| 0 | Nhận diện thiết bị, hiển thị VID/PID, firmware version | Có descriptor + 1 lệnh đọc |
| 1 | Hiển thị % pin, chế độ kết nối | Firmware có lệnh đọc pin |
| 2 | Đặt mức DPI, đổi danh sách DPI, polling rate | Firmware có lệnh ghi cấu hình |
| 3 | Gán lại nút (middle click, tilt, nút bên), macro | Firmware có bảng remap |
| 4 | Cập nhật firmware trên web (thay công cụ Windows, hỗ trợ cả macOS) | Hiểu bootloader + có cách phục hồi; rủi ro cao |

Nếu bước 5 cho thấy firmware **không** có lệnh cấu hình lúc chạy, mức 2–3 chỉ làm được bằng cách nạp các bản
firmware khác nhau (giống cách hãng đang chuyển Middle Click) — khi đó mức 4 trở thành tính năng cốt lõi.

## 7. Rủi ro và lưu ý

- **Brick chuột/receiver** khi flash sai — phải hiểu cơ chế bootloader và cách cứu trước khi viết luồng flash.
- **Bản quyền firmware**: không tự ý phân phối lại file firmware của hãng; tải từ nguồn chính thức hoặc xin phép.
- **Liên hệ HyperWork**: đây là thương hiệu Việt Nam; hỏi trực tiếp tài liệu giao thức / OEM có thể tiết kiệm
  rất nhiều thời gian reverse-engineer.
- Receiver USB-A và USB-C có thể có PID khác nhau; Bluetooth có thể không lộ collection vendor.

## 8. Việc cần làm tiếp

- [ ] Chạy `tools/hid-probe` với receiver USB-A, USB-C và (nếu có) cáp — lưu JSON vào `captures/`.
- [ ] Tải và phân tích công cụ firmware + hướng dẫn Middle Click của Gen 2 (và `HyperWork_MS01M_Driver_Upgrade.rar` để so sánh).
- [ ] Bắt pcap USB khi chạy công cụ trên Windows.
- [ ] Viết tài liệu `docs/PROTOCOL.md` từ dữ liệu bắt được.
- [ ] Dựng skeleton web app (Vite + TS) với transport WebHID + Mock.

## Nguồn

- [HyperWork — Silentium Gen 2 Wireless Mouse (EN)](https://hyperwork.vn/en/products/silentium-gen-2-wireless-mouse)
- [HyperWork — Chuột không dây Silentium Gen 2 (VI)](https://hyperwork.vn/products/chuot-khong-day-silentium-gen-2)
- [HyperWork blog — Silentium Gen 2: con lăn ngang, pin 2,5 tháng](https://hyperwork.vn/en/blogs/fact-tips/silentium-gen-2-mouse-horizontal-scroll-wheel-2-5-month-battery)
- [help.hyperwork.vn — Silentium Mini MS01M](https://help.hyperwork.vn/huong-dan-chuot-van-phong-hyperwork-silentium-mini-ms01m/)
- [help.hyperwork.vn — Silentium MS01](https://help.hyperwork.vn/huong-dan-chuot-van-phong-hyperwork-silentium-ms01/)
- [Techspace — HyperWork Silentium Gen 2](https://techspace.vn/chuot-khong-day-hyperwork-silentium-gen-2)
- [Thế Giới Di Động — Silentium Gen 2](https://www.thegioididong.com/chuot-may-tinh/chuot-sac-bluetooth-hyperwork-silentium-gen-2)
- [Hacom — Silentium Gen 2 (đen)](https://hacom.vn/chuot-khong-day-hyperwork-silentium-gen-2-hpw-ms01g2-blk-den-mehy0009)
- [ThinkPro — Silentium Gen 2 MS01G2](https://thinkpro.vn/chuot/chuot-khong-day-hyperwork-silentium-gen-2-ms01g2)
- [Chrome for Developers — Connect to uncommon HID devices (WebHID)](https://developer.chrome.com/docs/capabilities/hid)
- [WICG — WebHID API spec](https://wicg.github.io/webhid/)
- [WICG/webhid#12 — protected usages](https://github.com/WICG/webhid/issues/12)
- [awesome-webhid](https://github.com/robatwilliams/awesome-webhid)
- [Reverse engineering my USB mouse (hidraw + usbmon)](https://enocstocks.codeberg.page/entries/usb-reverse-engineering.html)
