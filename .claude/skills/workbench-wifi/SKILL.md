---
name: workbench-wifi
description: Use this skill whenever you need to control the workbench's WiFi radio for testing — starting a SoftAP for DUTs to connect to, joining a DUT's captive portal as a station, scanning for networks, relaying HTTP requests to devices on the WiFi network, or provisioning DUT WiFi credentials. Essential for any test that involves WiFi connectivity, captive portal flows, or HTTP communication with devices on the isolated test network (192.168.4.x). Triggers on "wifi", "AP", "station", "scan", "provision", "captive portal", "enter-portal", "HTTP relay", "wifi test", "SoftAP".
---

# ESP32 WiFi & Provisioning

Base URL: `http://workbench.local:8080`

## Step 0: Discover Workbench

Before using any workbench API, ensure `workbench.local` resolves:

```bash
curl -s http://workbench.local:8080/api/info
```

If that fails, run the discovery script from the workbench repo:

```bash
sudo python3 .claude/skills/esp-idf-handling/discover-workbench.py --hosts
```

## Operating Modes

The workbench has two WiFi operating modes:

| Mode | wlan0 usage | WiFi endpoints |
|------|-------------|----------------|
| **wifi-testing** (default) | Test instrument (AP/STA/scan) | Active |
| **serial-interface** | Joins WiFi for additional LAN | Disabled |

```bash
# Check current mode
curl http://workbench.local:8080/api/wifi/mode

# Switch to wifi-testing mode
curl -X POST http://workbench.local:8080/api/wifi/mode \
  -H 'Content-Type: application/json' \
  -d '{"mode": "wifi-testing"}'

# Switch to serial-interface mode (joins a WiFi network)
curl -X POST http://workbench.local:8080/api/wifi/mode \
  -H 'Content-Type: application/json' \
  -d '{"mode": "serial-interface", "ssid": "MyNetwork", "pass": "password"}'
```

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/wifi/ap_start` | Start WiFi AP (for DUT to connect to) |
| POST | `/api/wifi/ap_stop` | Stop WiFi AP |
| GET | `/api/wifi/ap_status` | Current AP state and connected clients |
| POST | `/api/wifi/sta_join` | Join an existing WiFi network as station |
| POST | `/api/wifi/sta_leave` | Disconnect from WiFi network |
| GET | `/api/wifi/scan` | Scan for nearby WiFi networks |
| POST | `/api/wifi/http` | HTTP relay — make HTTP requests via workbench's network |
| GET | `/api/wifi/events` | Long-poll for STA_CONNECT / STA_DISCONNECT events |
| POST | `/api/enter-portal` | Ensure device is on workbench AP — provision via captive portal if needed |
| GET | `/api/wifi/ping` | Quick connectivity check |
| POST | `/api/wifi/mode` | Set mode: `wifi-testing` or `serial-interface` |
| GET | `/api/wifi/mode` | Get current mode |

## WiFi AP (Access Point)

```bash
# Start AP
curl -X POST http://workbench.local:8080/api/wifi/ap_start \
  -H 'Content-Type: application/json' \
  -d '{"ssid": "TestAP", "pass": "testpass123", "channel": 6}'

# Check AP status and connected clients
curl http://workbench.local:8080/api/wifi/ap_status

# Stop AP
curl -X POST http://workbench.local:8080/api/wifi/ap_stop
```

AP and STA are mutually exclusive — starting one stops the other.

## WiFi STA (Station)

```bash
# Join a network
curl -X POST http://workbench.local:8080/api/wifi/sta_join \
  -H 'Content-Type: application/json' \
  -d '{"ssid": "MyNetwork", "pass": "password", "timeout": 15}'

# Disconnect
curl -X POST http://workbench.local:8080/api/wifi/sta_leave
```

## WiFi Scan

```bash
curl http://workbench.local:8080/api/wifi/scan
```

## WiFi On/Off Testing

To test a device's behavior when WiFi connectivity is lost and restored:

```bash
# 1. Ensure device is connected to workbench AP
curl -X POST http://workbench.local:8080/api/wifi/ap_start \
  -H 'Content-Type: application/json' \
  -d '{"ssid": "TestAP", "pass": "testpass123"}'

# 2. Stop AP — device loses WiFi
curl -X POST http://workbench.local:8080/api/wifi/ap_stop

# 3. Monitor device behavior (serial or UDP logs)
# ... wait for desired duration ...

# 4. Restart AP — device should reconnect
curl -X POST http://workbench.local:8080/api/wifi/ap_start \
  -H 'Content-Type: application/json' \
  -d '{"ssid": "TestAP", "pass": "testpass123"}'

# 5. Wait for device to reconnect
curl "http://workbench.local:8080/api/wifi/events?timeout=30"
```

## HTTP Relay

**IMPORTANT:** Devices on the workbench AP (192.168.4.x) are NOT directly reachable from the development machine. Always use this relay to make HTTP requests to device endpoints (e.g. `/status`, `/ota`). The response body is base64-encoded — decode it to get the actual JSON.

```bash
# GET request to device
curl -X POST http://workbench.local:8080/api/wifi/http \
  -H 'Content-Type: application/json' \
  -d '{"method": "GET", "url": "http://192.168.4.2/status", "timeout": 10}'

# POST with base64-encoded body
BODY=$(echo -n '{"key":"value"}' | base64)
curl -X POST http://workbench.local:8080/api/wifi/http \
  -H 'Content-Type: application/json' \
  -d "{\"method\": \"POST\", \"url\": \"http://192.168.4.2/config\", \"headers\": {\"Content-Type\": \"application/json\"}, \"body\": \"$BODY\", \"timeout\": 10}"

# Decode the base64 response body
curl -s -X POST http://workbench.local:8080/api/wifi/http \
  -H 'Content-Type: application/json' \
  -d '{"method": "GET", "url": "http://192.168.4.x:8080/endpoint", "timeout": 10}' \
  | python3 -c "import json,sys,base64; r=json.load(sys.stdin); print(base64.b64decode(r['body']).decode())"
```

## WiFi Events

Long-poll for STA_CONNECT / STA_DISCONNECT events:

```bash
curl "http://workbench.local:8080/api/wifi/events?timeout=30"
```

## Enter-Portal (Captive Portal Provisioning)

Provisions a device via its captive portal SoftAP, then starts the workbench AP so the device can connect back.

```bash
curl -X POST http://workbench.local:8080/api/enter-portal \
  -H 'Content-Type: application/json' \
  -d '{"portal_ssid": "iOS-Keyboard-Setup", "ssid": "TestAP", "password": "testpass123"}'
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `portal_ssid` | yes | — | Device's captive portal SoftAP name to join |
| `ssid` | yes | — | WiFi SSID to provision (also used for the workbench AP in step 4) |
| `password` | no | `""` | WiFi password |
| `portal_ip` | no | `192.168.4.1` | Device's portal IP |
| `form_path` | no | `"/connect"` | Endpoint on the device portal to POST to |
| `form_fields` | no | `null` | Full form body dict. `null` = use `{"ssid": ..., "password": ...}` |

**Procedure:**
1. Workbench joins the device's captive portal SoftAP (`portal_ssid`, open network)
2. POSTs `form_fields` (or default `ssid`/`password`) to `http://{portal_ip}{form_path}`
3. Disconnects from the device's SoftAP
4. Starts workbench AP (`ssid`/`password`) so the device can connect back

**All values must come from the project FSD** — never guess them.

Monitor progress via `GET /api/log`.

### jbdBridge example

jbdBridge uses `/save` as the form endpoint and a different set of field names:

```bash
curl -X POST http://workbench.local:8080/api/enter-portal \
  -H 'Content-Type: application/json' \
  -d '{
    "portal_ssid": "JBDBridge-Setup",
    "ssid": "TestAP",
    "password": "testpass123",
    "form_path": "/save",
    "form_fields": {
      "wifi_ssid": "TestAP",
      "wifi_pass": "testpass123",
      "mqtt_uri": "mqtt://192.168.4.1:1883",
      "mqtt_prefix": "jbd/",
      "poll_interval": "30"
    }
  }'
```

## Common Workflows

1. **Ensure device is connected to workbench AP:**
   - `POST /api/enter-portal` with all three values
   - `GET /api/wifi/ap_status` — verify device appears as connected client

2. **Test device WiFi connectivity:**
   - `POST /api/enter-portal` — ensure device is on workbench AP
   - `POST /api/wifi/http` — relay HTTP to device's IP to verify it responds

3. **Test WiFi disconnect/reconnect behavior:**
   - `POST /api/wifi/ap_stop` — device loses WiFi
   - Monitor device via serial (see workbench-logging)
   - `POST /api/wifi/ap_start` — device should reconnect
   - `GET /api/wifi/events` — confirm reconnection

## Troubleshooting

| Problem | Fix |
|---------|-----|
| AP won't start | Check that mode is `wifi-testing` via `GET /api/wifi/mode` |
| STA join timeout | Verify SSID/password; increase timeout |
| HTTP relay fails | Ensure workbench is on same network as target (AP or STA) |
| enter-portal "already running" | Previous run still active; wait for it to finish |
| No events from long-poll | DUT may not have connected yet; increase timeout |
| WiFi endpoints return "disabled" | System is in serial-interface mode; switch to wifi-testing |
