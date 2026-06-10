# Domain Pack: ESP32 Firmware

Domain-specific guidance for ESP32 projects (ESP-IDF or Arduino-ESP32). Load this
pack **only** when the project matches the detection signals below; otherwise the
platform-independent core (`references/test-architecture.md`) is enough.

## Detection signals

Treat the project as ESP32 firmware if any of these are present:

- Files: `sdkconfig`, `sdkconfig.defaults`, `platformio.ini`, `partitions*.csv`,
  `idf_component.yml`, `CMakeLists.txt` with `idf_component_register`.
- Code/symbols: `esp_`, `ESP_LOG`, `nvs_`, `esp_wifi`, `esp_mqtt`, `esp_ota`,
  `NimBLE`/`esp_ble`, `tinyusb`/`tusb_`, `app_main`, FreeRTOS tasks.
- Description mentions: ESP32 / ESP32-C3/S3/etc., ESP-IDF, Arduino-ESP32, a named
  dev board, or the chip's peripherals.

## Layer profile (for FSD §2.4)

Embedded layer contents — apply the core ownership rule (library/managed client to
an external service = foundation; hand-written decoder/driver/handler = interface):

- **L0 — Foundation / transport**: WiFi, VPN (e.g. WireGuard), the MQTT *client*,
  NVS/flash, the RTOS (boot + scheduling), `esp_http_client`/server stacks. Tested
  transitively.
- **L1 — Interfaces**: hand-written protocol decoders (DLMS/OBIS, custom UART
  framing), bus drivers (Modbus, I²C/SPI device drivers), device HTTP handlers
  (captive portal, status portal, OTA receiver), provisioning client. Pure
  converter/policy core on the host tier; wire/flow on target/bench.
- **L2 — Application logic**: control loops, state machines, scheduling/decision
  logic. Pure functions on the host tier.

## Test tiers (for the §x.0 Test Architecture, in the V&V chapter)

| Tier | Runs on | Speed | Catches |
|---|---|---|---|
| **host** | Dev machine, plain `gcc`/host build | ms, every commit | Pure logic: parsing, encoding, math, lookups, bounds. No ESP-IDF, no hardware. |
| **target** | The ESP32, real ESP-IDF | seconds, pre-merge | UART/I²C/SPI timing, NVS, the HTTP server, flash writes, RTOS behaviour. |
| **bench** | Device + real peers (sensors, broker, server) | minutes, pre-release | End-to-end; recovery, reconnection, timing. |
| **other** | — | — | Non-firmware (server/CI/silicon) or review-only. |

Extract each interface's pure core as a free function (e.g. a size/range
predicate separate from its HTTP/flash handler) so it is host-testable.

## Standard test libraries

Conditionally include standard test cases based on detected features. Scan the FSD
and source for the patterns, then fold the matching spec's tests into the test-spec
files that mirror the affected chapters (the generated traceability matrix picks
them up automatically).

| Feature | Detection Patterns | Test Spec | Include |
|---------|-------------------|-----------|---------|
| **WiFi STA** | `WiFi.begin`, `esp_wifi_connect`, "STA mode" | `esp32/wifi-test-spec.md` | WIFI-001–005, EC-100–101, EC-110–111, EC-115 |
| **Captive Portal** | `WiFi.softAP`, "captive portal", "AP mode" | `esp32/captive-portal-test-spec.md` | AP-001–006, CP-001–006, TC-CP-100–102 |
| **MQTT** | `PubSubClient`, `esp_mqtt`, "MQTT broker" | `esp32/mqtt-test-spec.md` | MQTT-001–031, TC-MQTT-100–103 |
| **BLE** | `NimBLE`, `esp_ble`, `BLEDevice`, "BLE", "GATT" | `esp32/ble-test-spec.md` | BLE-001–032, TC-BLE-100–103 |
| **BLE NUS** | `NUS`, `6E400001`, "Nordic UART" | `esp32/ble-test-spec.md` | BLE-020–023, TC-BLE-101 |
| **OTA** | `esp_ota`, `httpUpdate`, "firmware update", "OTA" | `esp32/ota-test-spec.md` | OTA-001–013, TC-OTA-100–102 |
| **USB HID** | `tinyusb`, `tusb_`, "HID", "keyboard", "USB device" | `esp32/usb-hid-test-spec.md` | HID-001–022, TC-HID-100–103 |
| **NVS** | `Preferences`, `nvs_`, "NVS", "stored credentials" | `esp32/nvs-test-spec.md` | NVS-001–024, TC-NVS-100–103 |
| **Watchdog** | `esp_task_wdt`, `TWDT`, "watchdog" | `esp32/watchdog-test-spec.md` | WDT-001–022, TC-WDT-100–102 |
| **Logging** | `ESP_LOG`, `udp_log`, "UDP logging", "serial log" | `esp32/logging-test-spec.md` | LOG-001–026, TC-LOG-100–103 |
| **Ethernet** | `W5500`, `ETH.begin`, "dual network" | `esp32/wifi-test-spec.md` | TEST-001–005, EC-100 |

### Workflow

1. Scan the FSD requirements and source code for the detection patterns above.
2. For each detected feature, read the corresponding `references/domains/esp32/*.md`.
3. Copy relevant requirements, functional tests, and edge cases into the FSD.
4. Update project-specific placeholders (SSIDs, IPs, timeouts, etc.).
5. Place the included tests in the spec files mirroring their chapters; the
   generated traceability matrix will reflect them (do not hand-edit the matrix).

### Spec files

All under `references/domains/esp32/`:

| File | Coverage |
|------|----------|
| `wifi-test-spec.md` | WiFi STA connection, signal, DHCP, ethernet test mode |
| `captive-portal-test-spec.md` | AP mode, captive portal, provisioning, credential change |
| `mqtt-test-spec.md` | Broker connection, pub/sub, QoS, LWT, reconnect, buffering |
| `ble-test-spec.md` | BLE advertising, GATT, NUS, pairing, coexistence |
| `ota-test-spec.md` | OTA download, rollback, integrity, power loss recovery |
| `usb-hid-test-spec.md` | USB enumeration, keyboard layouts, latency, stuck key prevention |
| `nvs-test-spec.md` | Config persistence, factory reset, corruption recovery, credentials |
| `watchdog-test-spec.md` | Software/hardware WDT, memory watchdog, false trigger prevention |
| `logging-test-spec.md` | Serial logging, UDP logging, log levels, crash capture |
