# HyperWork Silentium Gen 2 — Web Driver

**A browser-based driver for the HyperWork Silentium Gen 2 (MS01G2) wireless mouse**, built on Web Bluetooth and
WebHID, so it runs in Chrome or Edge on any desktop OS instead of a Windows-only vendor tool.
[Tiếng Việt](README.vi.md)

## Status

- ✅ **Firmware update over Web Bluetooth** (`tools/ble-ota/`), based on reverse engineering the official OTA tool
  ([docs/OTA.md](docs/OTA.md)). Tested against a simulated device; **not yet tried on a real mouse**.
- ✅ **HID probe** (`tools/hid-probe/`): reads HID descriptors and feature reports, logs input reports, exports JSON.
- ⏳ Runtime DPI and button configuration: the protocol is not known yet. See [docs/RESEARCH.md](docs/RESEARCH.md).

## What is inside

| Path | Contents |
|---|---|
| `docs/OTA.md` | Analysis of the vendor update package `MS01_G2_update.rar`: OnMicro OM6621 chip, Nordic Secure DFU over BLE |
| `docs/FIRMWARE.md` | Firmware reverse engineering: package CRC, button handling, macro table, button-combo → action table (groundwork for patching button mappings) |
| `docs/PROTOCOL.md` | HID descriptor analysis and the known protocol |
| `docs/RESEARCH.md` | Device facts, the WebHID platform, proposed architecture, the reverse-engineering process |
| `tools/ble-ota/` | Web Bluetooth firmware updater (replaces the Windows-only BLEOTATool) |
| `tools/hid-probe/` | WebHID probe for descriptors and reports |
| `captures/` | Descriptor JSON and pcaps captured from a real device |

## Run the HID probe

```sh
python3 -m http.server 8000
# open http://localhost:8000/tools/hid-probe/ in desktop Chrome or Edge
```

Click **Connect device…** and pick one entry for the mouse (Chrome's chooser selects one entry at a time; each entry
already includes every interface of that device). If there are several entries (for example "2.4G Wireless Receiver"
and "2.4G Mouse"), connect again to add each one, then **Export JSON** and save it to `captures/`.

## Update firmware (ble-ota)

```sh
python3 -m http.server 8000
# open http://localhost:8000/tools/ble-ota/ in desktop Chrome or Edge
```

1. Extract HyperWork's `MS01_G2_update.rar` and choose `MS01G2UPDATE.zip` on the page.
2. Switch the mouse to Bluetooth and hold the connect button for 3 seconds (pairing mode; do **not** pair it with the
   computer).
3. **Choose mouse…** → pick the right device → tick the confirmation → **Start update**.

On macOS, Chrome needs the Bluetooth permission (System Settings → Privacy & Security → Bluetooth).

> Flashing firmware can brick a device. The updater has only been tested against a simulator; use it at your own risk.
