# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Universal Embedded Workbench

Raspberry Pi-based test instrument for ESP32 firmware: serial proxy (RFC2217), WiFi AP/STA, GPIO control, HTTP relay, all via REST API.

## Tech Stack

- **Runtime**: Python 3.9+ (Pi), Python 3.11 (devcontainer)
- **Frameworks**: stdlib `http.server` (no Flask), pyserial (RFC2217), hostapd/dnsmasq (WiFi), bleak (BLE), mosquitto (MQTT)
- **Testing**: pytest, ruff, mypy
- **Hardware**: Raspberry Pi Zero W (eth0 + wlan0), USB hub for serial slots

## Commands

```bash
# Run tests (instrument self-tests, no DUT required)
pip install -r requirements-dev.txt
pytest pytest/

# Run a single test
pytest pytest/workbench_test.py::TestBasicProtocol::test_wt100_ping_response

# Run tests against a real Pi (instrument tests only)
pytest pytest/ --wt-url http://192.168.0.87:8080

# Run tests that need a DUT plugged in
pytest pytest/ --wt-url http://192.168.0.87:8080 --run-dut

# Lint / type-check
ruff check .
mypy --strict .

# Run portal manually (on Pi)
python3 pi/portal.py

# Discover USB slot keys (on Pi, after plugging in devices)
rfc2217-learn-slots

# Install on Pi
cd pi && bash install.sh
```

## Architecture

`portal.py` is the single-process HTTP server that owns all instrument state. It imports four sub-module controllers as libraries (not subprocesses):

- `debug_controller.py` — OpenOCD lifecycle, chip auto-detection, probe allocation (GDB)
- `wifi_controller.py` — hostapd/dnsmasq AP, wpa_supplicant STA, HTTP relay
- `ble_controller.py` — bleak BLE scan/connect/write (runs its own asyncio loop in a daemon thread)
- `mqtt_controller.py` — mosquitto broker lifecycle for MQTT client testing

The HTTP handler is a single `Handler(BaseHTTPRequestHandler)` class that dispatches requests by path to `_handle_*` methods. All API responses use `{"ok": true/false, ...}`. The `WorkbenchDriver` in `pytest/workbench_driver.py` is the Python client wrapping every API call for test scripts.

### Slot model

A **slot** is the permanent identity for a physical USB hub port. Slots are created at startup (from `workbench.json` config or auto-detected from hub topology) and persist regardless of whether a device is plugged in. Each slot has a stable TCP port for RFC2217. The slot state machine:

```
absent → idle → resetting / monitoring / download_mode / debugging
                         ↘ flapping → recovering
```

USB hotplug events flow: udev rule → `/api/hotplug` POST → `portal.py` starts/stops `plain_rfc2217_server.py` subprocess per slot.

### Key data flows

- **Serial access**: esptool/pyserial → TCP port (RFC2217) → `plain_rfc2217_server.py` → `/dev/ttyUSBx`
- **Serial logging**: `plain_rfc2217_server.py` feeds a per-slot ring buffer (`SERIAL_BUF_MAXLEN=1000`) readable via `/api/serial/output`
- **UDP debug logs**: ESP32 firmware → UDP port 5555 → `_udp_log` deque → `/api/udplog`
- **Debug**: `/api/debug/start` → `debug_controller` → OpenOCD subprocess → GDB port (3333+)
- **OTA firmware**: binary uploaded via `/api/firmware/upload` → stored in `/var/lib/rfc2217/firmware/` → served over HTTP to ESP32

### Config

Hardware layout in `/etc/rfc2217/workbench.json` (or `pi/config/workbench.json` as template):
- `gpio_boot`, `gpio_en`: BCM GPIO pins for ESP32 EN/BOOT control
- `slots`: USB prefix → TCP port + GDB port + OpenOCD telnet port mapping
- `debug_probes`: external JTAG probes (ESP-Prog / FT2232H)

## Project Structure

```
pi/
  portal.py               # HTTP server, slot manager, proxy supervisor (main entry)
  wifi_controller.py      # WiFi instrument (AP, STA, scan, HTTP relay)
  debug_controller.py     # OpenOCD/GDB manager, chip auto-detection
  ble_controller.py       # BLE proxy (bleak, asyncio in daemon thread)
  mqtt_controller.py      # mosquitto broker lifecycle
  plain_rfc2217_server.py # RFC2217 server subprocess (one per slot)
  serial_proxy.py         # RFC2217 proxy with traffic logging (alternative)
  signal_generator.py     # Unified RF source: Si5351 (I2C) + PE4302 attenuator + GPCLK fallback
  si5351.py / pe4302.py / gpclk.py / bcm_gpio.py  # Hardware drivers
  morse.py                # Backend-agnostic Morse keyer
  sniffer.py              # Passive serial sniffer
  config/workbench.json   # Hardware config template (slots, GPIO, probes)
  config/signalgen.json   # Signal generator config
  install.sh              # Pi installer (deps, systemd, udev)
  udev/                   # Hotplug rules → POST /api/hotplug
  systemd/                # rfc2217-portal.service
pytest/
  workbench_driver.py     # WorkbenchDriver — HTTP client wrapping all API calls
  conftest.py             # pytest fixtures (workbench, wifi_network); --wt-url / --run-dut flags
  workbench_test.py       # Instrument self-tests (WT-xxx)
test-firmware/            # ESP-IDF reference firmware (OTA, BLE NUS, captive portal)
debug-test/               # ESP-IDF firmware for debug/GDB tests
docs/
  Embedded-Workbench-FSD.md  # Full functional specification (authoritative)
```

## Code Style

- Python: ruff for linting, mypy strict, `ruff format` for formatting
- `snake_case` for functions and variables
- REST API endpoints under `/api/` namespace
- All API responses: `{"ok": true, ...}` or `{"ok": false, "error": "..."}`

## Key Conventions

- Always release GPIO pins after use: `gpio_set(pin, "z")` (switches to input with pull-up)
- BCM GPIOs 16–27 are safe for DUT control; 2/3 = I2C, 5/6 = GPCLK, 6/12/13 = PE4302
- `SERIAL_PI=192.168.0.87` is set in the devcontainer; `WORKBENCH_URL=http://$SERIAL_PI:8080`
- Tests marked `@pytest.mark.requires_dut` are skipped unless `--run-dut` is passed
- Deploy portal: `scp pi/portal.py pi@192.168.0.87:/tmp/portal.py && ssh pi@192.168.0.87 'sudo cp /tmp/portal.py /usr/local/bin/rfc2217-portal && sudo systemctl restart rfc2217-portal'`
- Deploy debug_controller: `scp pi/debug_controller.py pi@192.168.0.87:/tmp/ && ssh pi@192.168.0.87 'sudo cp /tmp/debug_controller.py /usr/local/bin/debug_controller.py && sudo systemctl restart rfc2217-portal'`

## Gotchas / Do Not

- Do NOT SSH into the Pi to interact with the workbench — always use the HTTP API at `:8080`. SSH is only for deploying code updates to `/usr/local/bin/`.
- Do NOT add Flask or other web frameworks — the server uses stdlib `http.server` intentionally (Pi Zero memory constraints).
- Slot identity is USB-path-based, not device-serial-based. Never key slot state on VID/PID or serial number.

## Specifications

`docs/Embedded-Workbench-FSD.md` is authoritative for all functional behavior — slot auto-detect, flashing, GPIO API, signal generator, WiFi modes, GDB debug, RFC2217 semantics, flap detection, etc.

## Host Access

See `remote-connections` skill for SSH, InfluxDB, Grafana, and Docker details.
