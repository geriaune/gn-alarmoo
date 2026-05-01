# gn-alarmoo — Paradox Alarm Wi-Fi & VPN Module

Connect a Paradox alarm panel to Home Assistant over Wi-Fi with optional VPN

> ⚠️ **Important:** Only one module can use the serial interface at a time. Remove any existing IP150 or GSM modules before connecting this one. Make sure your panel is not locked or running firmware with encrypted serial comms (maybe they have fixed this already? Don't update your panel's firmware if this works!).

---

## How It Works

```
Paradox Panel  ──serial──►  ESP32-S3  ──Wi-Fi──►  [Tailscale VPN]  ──►  Home Assistant
                                                                              │
                                                                         PAI App + MQTT
```

The ESP32 exposes the panel's UART over Wi-Fi (port `10000`). The **Paradox Alarm Interface (PAI)** app in Home Assistant connects to it, decodes the data, and pushes it to an MQTT broker — making all zones, sensors and PGMs available as HA entities.

---

## Hardware

| Part | Notes |
|---|---|
| ESP32-S3 N16R8 (16MB flash, 8MB PSRAM) | Min 8MB flash is required for VPN support |
| External Wi-Fi antenna | Optional, helps in poor signal spots |
| 4-pin Molex KK / Dupont connector | Connects to panel serial port |
| DC buck step-down (12V → 5V) | Powers ESP from panel's 12VDC rail |
| USB Type-C connector (with data pins) | For module power and TX/RX |
| 24 AWG wire | Low current draw (~1W total) |

**Panel wiring (confirmed for SP7000; swap TX/RX if no connection):**

```
Serial on Panel         BUCK              ESP32
┌───────┐               
│ Tx   ┌╵   == > == > == > == > == > == > RX
│ Rx   │    == > == > == > == > == > == > TX
│ GND  │    == > IN(-)   ->   OUT(-) == > USB GND 
│ AUX+ └╷   == > IN(+)   ->   OUT(+) == > USB 5V
└───────┘
```

You can get one already built here: - [geriaune](http://link.geriaune.pro/gn-alarmoo)

---

## Software Stack

Install these as **Home Assistant Apps/Add-ons**:

1. **ESPHome** — flashes and manages ESP32 firmware
2. **Mosquitto broker** — MQTT broker
3. **[Paradox Alarm Interface (PAI)](https://github.com/ParadoxAlarmInterface/pai)** — decodes panel data → MQTT
4. **Tailscale** — VPN (optional but recommended if HA is remote)

You also need a free [Tailscale](https://tailscale.com) account, or a self-hosted [Headscale](https://headscale.net) server.

---

## ESPHome Config (`paradox.yaml`)

Key settings — see the full file in this repo for the complete config.

```yaml
esphome:
  name: paradox

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: esp-idf

# Tailscale VPN package
packages:
  tailscale:
    url: https://github.com/Csontikka/esphome-tailscale
    ref: main
    files: [packages/tailscale/tailscale.yaml]

tailscale:
  auth_key: !secret tailscale_auth_key
  hostname: "paradox"
  # login_server: "http://vpn.yourhost.com:8080"  # headscale only

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  # use_address: "100.64.0.X"  # set yours AFTER first flash + successful VPN connect

uart:
  tx_pin: 15    # change if required for your module
  rx_pin: 7     # change if required for your module
  baud_rate: 9600

stream_server:
  port: 10000  # PAI connects here
```

**Secrets to add in ESPHome:**
```yaml
wifi_ssid: "YourSSID"
wifi_password: "YourPassword"
tailscale_auth_key: "tskey-auth-****"   # generate one by adding a linux device in the Tailscale/Headscale
```

After flashing, find the Tailscale IP in the ESP logs or your Tailscale dashboard, then uncomment and set `use_address`.

---

## PAI App Configuration

| Setting | Value |
|---|---|
| `CONNECTION_TYPE` | `IP` |
| `IP_CONNECTION_HOST` | Your Tailscale IP of ESP (`100.XX.0.X`) |
| `IP_CONNECTION_PASSWORD` | Panel password (default: `paradox`) |
| `IP_CONNECTION_BARE` | ✅ Enable (required!) — this hides under "Show unused optional configuration options" |
| `MQTT_ENABLE` | ✅ Enable |
| `MQTT_USERNAME` / `MQTT_PASSWORD` | Match Mosquitto broker user |
| `MQTT_HOST` | HA's local IP (find with `ha network info`) |
| Port (Network section) | `10000` |

> Use the local HA IP (e.g. `192.168.0.222`), not `127.0.0.1` — it won't work inside Docker.

**Healthy PAI log output:**
```
INFO - Connection established
INFO - Panel Identified SP7000 version 7.0
INFO - Authentication Success
INFO - Connection OK
```

---

## ESPHome

Go to **HA Settings → Apps → ESPHome** and enable **"Use ping for status"**.
mDNS doesn't work across subnets (VPN), so ICMP ping is needed to show nodes as online.

---

## VPN IP Note

Every re-flash from PC or re-authentication assigns a new Tailscale IP. When that happens, update the IP in:
- ESPHome device config (`use_address`)
- PAI App (`IP_CONNECTION_HOST`)
- ESPHome integration in HA

---

## Sources

- [ESPHome](https://github.com/esphome/esphome)
- [Paradox Alarm Interface (PAI)](https://github.com/ParadoxAlarmInterface/pai)
- [esphome-tailscale](https://github.com/Csontikka/esphome-tailscale)
- [esphome-stream-server](https://github.com/oxan/esphome-stream-server)
- [Full build article](https://geriaune.pro/howto/2026/05/01/10-paradox-alarm-wi-fi-and-vpn-module/)
- [Hardware](http://link.geriaune.pro/gn-alarmoo)
