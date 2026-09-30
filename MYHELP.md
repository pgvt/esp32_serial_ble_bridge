# ESP32-C3 → iPhone Wireless UART Console: Complete Build Summary

A working BLE-UART bridge that lets your iPhone act as a serial terminal, replacing a physical FTDI cable.

---

## 1. Hardware Used

- **ESP32-C3** dev board (any variant)
- **iPhone / iPad** as the console
- **Jumper wire** for the loopback test
- **3 wires** to the target device (TX, RX, GND)

---

## 2. Software Used

| Component | Version / Choice |
|---|---|
| IDE | VS Code |
| Extension | ESP-IDF Extension (Espressif) |
| **ESP-IDF version** | **v5.3.6** (do NOT use 6.x) |
| Target | `esp32c3` |
| iOS app | **BLE Serial Pro** (~$5.74 CAD) |
| Firmware repo | `https://github.com/riozebratubo/esp32_serial_ble_bridge` |
| License | CC BY-NC 4.0 |

---

## 3. Installation Steps (VS Code)

1. Install **VS Code**.
2. Install the **ESP-IDF** extension by Espressif.
3. Run the extension's **Express Install**:
   - **Select version: v5.3.6** (not 6.x)
   - Let it install its **own bundled Python** (do NOT reuse old system Python)
   - Let it install the **xtensa toolchain** for ESP32-C3
   - Default install path (e.g. to `C:\esp` + `C:\Espressif`) is fine — **no spaces in path**
4. Installation is ~7 GB and takes 30–60 minutes. Let it run untouched.

---

## 4. Clone and Open the Project

```
git clone https://github.com/riozebratubo/esp32_serial_ble_bridge
```

Open the folder in VS Code. The **ESP-IDF: Explorer** panel (shown in your screenshot) is your control center.

---

## 5. Code Fixes Required for ESP32-C3

Open `esp32_serial_ble_bridge.c` and change the default UART pins. The repo defaults (GPIO 17/18) are for classic ESP32 and **do not exist on the C3**.

```c
app_settings_t app_settings = {
    .complete_service_uuid = "0000FFE0-0000-1000-8000-00805F9B34FB",
    .complete_characteristic_uuid = "0000FFE1-0000-1000-8000-00805F9B34FB",
    .device_name = "ESPSERIALBLE",
    .manufacturer_bytes = {0x12, 0x23, 0x45, 0x56},
    .desired_ble_mtu = 400,
    .force_mac_address = "",
    .uart_pin_tx = 4,        // <-- CHANGED (was 17)
    .uart_pin_rx = 5,        // <-- CHANGED (was 18)
    .uart_buffer_size = 2048,
    .uart_baud_rate = 115200
};
```

**Why 4 and 5?** They're free GPIOs on the C3 that don't collide with USB-JTAG (20/21) or strapping pins (2, 8, 9).

> ⚠️ **Build issue**: Project was originally for ESP32 and ESP-IDF 5.3.0  
 On ESP32-C3 with ESP-IDF 5.3.6, the project's calls to legacy BLE advertising APIs  
 (esp_ble_gap_start_advertising, esp_ble_gap_config_adv_data) failed to link because  
 IDF 5.3.6 gates those functions behind CONFIG_BT_BLE_42_FEATURES_SUPPORTED,  
 and that option defaults to off on the C3  
 (which defaults to BLE 5 extended advertising instead). The repo's sdkconfig.defaults did not set it.  
   
 > **Fix**: add CONFIG_BT_BLE_42_FEATURES_SUPPORTED=y to sdkconfig.defaults (and the active sdkconfig), then rebuild. This restores the two GAP symbols at link time.  

In the VS Code ESP-IDF Explorer, click **SDK Configuration Editor (menuconfig)**  
and search for `BT_BLE_42` / `BT_BLE_50`
1. **`Enable BLE 4.2 features`** → ✅ (check!)
2. **`Enable BLE 4.2 DTM test`** → ✅ (fine, harmless)
3. **`Enable BLE 4.2 advertising`** → ✅ (required — this is the one that fixes the link error)
4. **`Enable BLE 4.2 scan`** → ✅ (fine, harmless)

Then **scroll down** in the same "Bluedroid Options" section and find these BLE 5.0 options — **uncheck all of them**:

5. **`Enable BLE 5.0 features`** → ❌ **uncheck**

---

## 6. Build & Flash — Using the ESP-IDF Explorer Panel

Match your screenshot's commands in order:

| Step | Panel Command | Notes |
|---|---|---|
| 1 | **Select Current Project workspace folder** | Point at the cloned repo |
| 2 | **Select current ESP-IDF version** | Pick **v5.3.6** |
| 3 | **Select Flash Method** | Choose **UART** (or "USB Serial/JTAG" if your board exposes it) |
| 4 | **Select Port to Use (COM, tty, usbserial)** | Pick the COM port of your C3 |
| 5 | **Select Monitor Port to Use** | Same COM port |
| 6 | **Set Espressif Device Target (IDF_TARGET)** | Select **esp32c3** |
| 7 | **SDK Configuration Editor (menuconfig)** | Optional — see §9 |
| 8 | **Full Clean** | Do this once after changing target or pins |
| 9 | **Build Project** | Compiles the firmware |
| 10 | **Flash Device** | Uploads to the C3 |
| 11 | **Monitor Device** | Opens the serial console — you'll see the boot log |

**Do NOT use "Launch Debug" or "OpenOCD Server"** — those require JTAG drivers that aren't present, and are not needed for this project.

---

## 7. First Boot — What to Look For in the Monitor

A healthy boot log shows:

```
I (24)  boot: ESP-IDF v5.3.6 2nd stage bootloader
I (25)  boot: chip revision: v0.4
...
I (479) SERIALBLE_SETTINGS: - Uart pin tx: 4
I (489) SERIALBLE_SETTINGS: - Uart pin rx: 5
I (489) SERIALBLE_SETTINGS: - Uart buffer size: 2048
I (499) SERIALBLE_SETTINGS: - Uart baud rate: 115200
I (519) BLE_INIT: Bluetooth MAC: 88:56:a6:64:7b:fa
I (589) SERIALBLE_BLE: REGISTER_APP_EVT, status 0, app_id 0
I (599) SERIALBLE_BLE: SERVICE_START_EVT, status 0, service_handle 40
I (609) SERIALBLE_BLE: ADD_CHAR_EVT, status 0, attr_handle 42, service_handle 40
I (619) uart: queue free spaces: 50
I (619) uart: queue free spaces: 50
```

**Two harmless warnings you'll see** (ignore them):

- `E SPIFFS: spiffs partition could not be found` — no SPIFFS partition in the table; `save`/`restore` won't persist, defaults are used. Bridge works fine.
- `W spi_flash: Detected size(4096k) larger than ... (2048k)` — your board has 4MB flash but the build assumes 2MB. Cosmetic.

---

## 8. Connecting from the iPhone

1. Install **BLE Serial Pro** on the iPhone.
2. Open the app, scan for BLE devices.
3. Connect to **`ESPSERIALBLE`**.
4. The app should expose the service and characteristic.
5. **Enable notifications** on characteristic `FFE1` (in the app this is usually a "listen" or "subscribe" toggle).

Once the ESP32 log prints:

```
ESP_GATTS_CONNECT_EVT, conn_id 0, remote ...
ESP_GATTS_MTU_EVT, MTU 400
update connection params status = 0
GATT_WRITE_EVT, conn_id 0, trans_id 3, handle 43
notify enable
```

…you're linked.

---

## 9. Proving the Round Trip — The Loopback Test

Before connecting to any real target, do the loopback:

1. **Short GPIO4 to GPIO5** with a jumper wire.
2. In BLE Serial Pro, type `hello` and send.
3. You should see `hello` echo back in the app.

Expected ESP32 monitor output:

```
SERIALBLE_BLE: GATT_WRITE_EVT, conn_id 0, trans_id 4, handle 42
SERIALBLE_BLE: Received (len 5): hello
SERIALBLE_UART: Sending data to BLE: [mtu: 400, read size: 5, data: hello]
```

That `Received` + `Sending data to BLE` pair **proves the entire bidirectional chain is working**.

---

## 10. Wiring to the Real Target Device

Remove the loopback jumper. Connect:

| ESP32-C3 pin | Goes to target |
|---|---|
| **GPIO4 (TX)** | Target **RX** |
| **GPIO5 (RX)** | Target **TX** |
| **GND** | Target **GND** |

**Critical rules:**

- **Cross TX/RX** — never straight-through.
- **Common GND is mandatory** — without it, garbage or silence.
- **3.3V logic only.** If the target is 5V TTL, add a level shifter or resistor divider on the ESP32's RX line. 5V into GPIO5 will kill the C3 over time.

---

## 11. Baud Rate

Default is **115200, 8N1** on the UART side. This is set in `esp32_serial_ble_bridge.c` as `.uart_baud_rate`.

If your target uses a different baud:

- **Easiest**: edit `.uart_baud_rate` in the source, rebuild, re-flash.
- **Alternative**: send `set uart_baud_rate="9600"` followed by `save` over the BLE terminal — but **`save` won't persist** because there's no SPIFFS partition. You'd need to add one (see §12).

---

## 12. Optional Improvements (Not Required)

**Add SPIFFS partition** so `save`/`restore` work:

Create `partitions.csv` in the project root:

```
# Name,   Type, SubType, Offset,   Size,  Flags
nvs,      data, nvs,     0x9000,   0x6000,
phy_init, data, phy,     0xf000,   0x1000,
factory,  app,  factory, 0x10000,  0x100000,
spiffs,   data, spiffs,  0x110000, 0xF0000,
```

Then in `SDK Configuration Editor (menuconfig)`, set:

- `CONFIG_PARTITION_TABLE_CUSTOM=y`
- Point to `partitions.csv`

**Silence the flash-size warning** — in menuconfig:

- `CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y`
- `CONFIG_ESPTOOLPY_FLASHSIZE="4MB"`

---

## 13. Final State Achieved

| Layer | Status |
|---|---|
| ESP-IDF v5.3.6 toolchain | ✅ |
| ESP32-C3 target | ✅ |
| Firmware build & flash | ✅ |
| BLE advertising (`ESPSERIALBLE`) | ✅ |
| Service `0000FFE0` / Char `0000FFE1` | ✅ |
| MTU negotiated (400) | ✅ |
| Notify enabled from iPhone | ✅ |
| Phone → ESP32 write path | ✅ |
| ESP32 → UART TX out (GPIO4) | ✅ |
| UART RX in (GPIO5) → ESP32 | ✅ |
| ESP32 → Phone notify path | ✅ |
| Loopback echo confirmed on iPhone | ✅ |

---

## 14. Reference Links

- Firmware repo: `https://github.com/riozebratubo/esp32_serial_ble_bridge`
- ESP-IDF VS Code extension docs: `https://github.com/espressif/vscode-esp-idf-extension`
- OpenOCD troubleshooting FAQ (not needed for this project): `https://github.com/espressif/openocd-esp32/wiki/Troubleshooting-FAQ`

---

**You now have a functional wireless UART console on an iPhone for roughly the cost of the BLE Serial Pro app.** Wire it to any 3.3V TTL serial target and debug away.