# ESP32-C6 Zigbee Coordinator + ESP32-C3 WiFi Bridge

Turn a ~$5 **ESP32-C6** into a network-attached Zigbee coordinator, using an
**ESP32-C3** running ESPHome as a WiFi-to-UART bridge. Lets you run a **second,
independent Zigbee2MQTT network** in Home Assistant without touching your
existing one.

```
┌──────────────┐  UART 115200  ┌──────────────┐   WiFi    ┌──────────────────┐
│  ESP32-C6    │  TX22 ──► RX20│  ESP32-C3    │  TCP 6638 │  Home Assistant  │
│  ZBOSS NCP   │  RX23 ◄── TX21│  ESPHome     │ ◄───────► │  Zigbee2MQTT #2  │
│  coordinator │  GND ──── GND │ stream_server│           │  adapter: zboss  │
└──────┬───────┘               └──────────────┘           └──────────────────┘
       │ 802.15.4
   Zigbee devices
```

## Why

- **Second Zigbee network.** A separate coordinator, channel and PAN — useful for
  isolating a noisy/untrusted device class, testing, or splitting a large network.
  Your existing coordinator is untouched.
- **Place the coordinator where the radio is good,** not where the HA box is.
  The C6 doesn't need to be near the server — only near mains power and WiFi.
- **The C6 is a genuinely capable radio:** 802.15.4 + an integrated PA for up to
  **+20 dBm**, and the firmware supports 200 direct children.

### Why not just use the C6's own WiFi?

The coordinator firmware has an experimental `wifi-coex` build that joins WiFi
directly and exposes TCP itself — no C3 needed. It is explicitly **experimental**:
the C6 has a single 2.4 GHz radio time-shared between WiFi and 802.15.4, and it
requires a custom Zigbee2MQTT image with longer timeouts.

This two-chip design keeps the radio dedicated to Zigbee. Upstream also notes the
UART transport avoids the USB-CDC `inReset` quirks entirely, so it is the more
robust path.

## Hardware

| Part | Notes |
|---|---|
| ESP32-C6 board | Any. **4 MB flash required** (app partition is 1900 K). |
| ESP32-C3 board | Any; a C3 SuperMini is tiny and cheap. |
| 3 jumper wires | Both boards are 3.3 V logic — no level shifter. |
| 2 USB power supplies | Or one, if you power the C6 from the C3 (see below). |

## Wiring

Three wires. Both chips are 3.3 V, so TX/RX connect directly.

```
      ESP32-C3 (bridge)              ESP32-C6 (coordinator)
     ┌──────────────────┐           ┌──────────────────┐
     │  GPIO21  (TX) ───┼──────────►┤ GPIO23  (RX)     │
     │  GPIO20  (RX) ◄──┼───────────┤ GPIO22  (TX)     │
     │  GND ────────────┼───────────┤ GND              │
     └──────────────────┘           └──────────────────┘
```

| ESP32-C3 | | ESP32-C6 |
|---|---|---|
| GPIO21 (TX) | → | GPIO23 (RX) |
| GPIO20 (RX) | ← | GPIO22 (TX) |
| GND | ↔ | GND |

**TX goes to RX, crossed.** If the coordinator never answers, swap those two
wires — that is the single most common mistake.

### Power — read this before connecting 5V

> **If each board has its own USB supply, connect GND and the two data lines
> ONLY. Do not connect the 5V pins.** Tying two powered 5V rails together
> backfeeds one supply into the other.

Connect `5V ↔ 5V` **only** when a single USB supply feeds the C3 and the C6 draws
its power from it (or vice versa). In that case it is 4 wires: 5V, GND, TX, RX.

### Board pin labels

On a **Seeed XIAO ESP32-C6**, GPIO22/GPIO23 are silkscreened **D4/D5**. Generic
devkits label them GPIO22/GPIO23 directly. The UART pins are configurable in the
firmware via `idf.py menuconfig` → `NCP_UART_*`.

## Step 1 — Flash the ESP32-C6 coordinator

Firmware: [tostmann/esp-coordinator](https://github.com/tostmann/esp-coordinator)
(ZBOSS NCP). Easiest route is the web flasher at
<https://install.busware.de/zboss/>, which auto-detects the chip.

From the CLI instead:

```bash
pip install esptool

# Download factory.bin + SHA256SUMS from the latest release, then verify:
shasum -a 256 -c SHA256SUMS --ignore-missing

esptool --chip esp32c6 -p /dev/ttyACM0 erase-flash
esptool --chip esp32c6 -p /dev/ttyACM0 write-flash 0x0 esp-coordinator-esp32c6-<version>-factory.bin
```

The erase is not strictly required, but it clears any stale NVS/`zb_storage` from
previous firmware that could confuse the ZBOSS stack.

> Use the **stable** release, not the `wifi-coex` branch. Since v1.3.x a single
> image serves USB **and** UART simultaneously — there is no separate "UART
> build". Replies follow whichever interface last delivered a valid NCP frame, so
> a probe on the USB side cannot steal the link from Zigbee2MQTT.

### Verify it booted

On reset the firmware emits a ZBOSS boot frame. Read it on the C6's USB port:

```bash
python3 - <<'PY'
import serial, time
s = serial.Serial("/dev/ttyACM0", 115200, timeout=0.2)
s.dtr = False; s.rts = True; time.sleep(0.15)   # pulse reset
s.reset_input_buffer(); s.rts = False
data = b""; t0 = time.time()
while time.time() - t0 < 4:
    data += s.read(256)
print(data.hex(" "))
PY
```

Expected:

```
de ad 05 00 06 01 8f                             <- boot-ready
de ad 0e 00 06 c0 ...                            <- NCP reset indication
```

## Step 2 — Flash the ESP32-C3 bridge (ESPHome)

```bash
cp esphome/secrets.yaml.example esphome/secrets.yaml   # then edit it
```

Copy `esphome/zigbee-bridge.yaml` and the `esphome/components/` directory into
your ESPHome config dir, keeping them side by side — the YAML references the
component with `type: local`, `path: components`.

```
esphome/
├── zigbee-bridge.yaml
├── secrets.yaml
└── components/
    └── stream_server/
```

Then compile and flash (first flash over USB; later ones via OTA):

```bash
esphome run zigbee-bridge.yaml
```

The `stream_server` component is vendored because upstream does not build on
ESPHome 2026.x — see [`esphome/components/stream_server/README.md`](esphome/components/stream_server/README.md).

**Give the C3 a DHCP reservation.** Zigbee2MQTT addresses it by IP; if it moves,
the Zigbee network goes down.

### Verify the full path

With both boards wired and powered, connect to the bridge and reset the C6. The
same boot frame should arrive over TCP — proving C6 UART → C3 → WiFi → network:

```bash
python3 - <<'PY'
import socket, time
s = socket.create_connection(("BRIDGE_IP", 6638), timeout=5)
s.settimeout(0.3)
data = b""; t0 = time.time()
print("connected — now reset the C6")
while time.time() - t0 < 15:
    try: data += s.recv(512)
    except Exception: pass
print(data.hex(" "))
PY
```

Silence here means the C6→C3 direction is wrong: swap TX/RX.

## Step 3 — Second Zigbee2MQTT add-on (Home Assistant)

The official add-on can only run once, so install a second copy as a **local**
add-on. On the HA host:

```bash
mkdir -p /addons/zigbee2mqtt_2
```

Copy [`addon/config.json`](addon/config.json) there. It is the upstream
`hassio-zigbee2mqtt` manifest with three changes:

- `slug` / `name` / `panel_title` — made unique so it installs alongside the original
- `data_path` → `/config/zigbee2mqtt_2` — separate state directory
- `ports` set to `null` — no host ports published, avoiding a clash with the
  original's 8485. The frontend is reached through HA ingress.

Reload the add-on store, then install **Zigbee2MQTT #2 (ESP32-C6)**.

Copy [`zigbee2mqtt/configuration.example.yaml`](zigbee2mqtt/configuration.example.yaml)
to `/config/zigbee2mqtt_2/configuration.yaml` and set `BRIDGE_IP` and
`MQTT_BROKER_IP`:

```yaml
serial:
  adapter: zboss
  port: tcp://BRIDGE_IP:6638
  baudrate: 115200
mqtt:
  server: mqtt://MQTT_BROKER_IP:1883
  base_topic: zigbee2mqtt_2      # MUST differ from the first instance
advanced:
  channel: 25                    # keep clear of your other Zigbee net and WiFi
  pan_id: GENERATE
  ext_pan_id: GENERATE
  network_key: GENERATE
  transmit_power: 20
```

Pick a `channel` well away from your existing Zigbee network *and* your WiFi
channel. `base_topic` **must** differ from the first instance or the two will
publish over each other.

## Step 4 — Start and verify

```bash
ha addons start local_zigbee2mqtt_2
ha addons logs local_zigbee2mqtt_2
```

A healthy first start:

```
zh:zboss: ZBOSS Adapter starting
zh:zboss:uart: Socket ready
zh:zboss:driv: Driver connected
zh:zboss:driv: Form network
z2m: zigbee-herdsman started (reset)
z2m: Coordinator firmware version: '{"meta":{...,"revision":"1.5.0.79","stack":"3.15.0.0"},"type":"zboss"}'
z2m: Connected to MQTT server
```

Zigbee2MQTT rewrites `configuration.yaml` at this point, replacing the `GENERATE`
placeholders with the real PAN ID, extended PAN ID and network key.

### Confirm the network persists across a reboot

Worth doing **before** pairing anything. Reset the C6, restart the add-on, and
check the log:

```
z2m: zigbee-herdsman started (resumed)     <-- correct
```

`(resumed)` means the coordinator restored its network from NVRAM. If you see
`(reset)` and `Form network` again, it formed a *new* network — which would
silently orphan every paired device. Resolve that before pairing.

## Troubleshooting

| Symptom | Cause |
|---|---|
| No boot frame over TCP | TX/RX swapped. Swap the two data wires. |
| Add-on connects, no coordinator reply | Only one direction is wired — check both TX→RX runs and the common GND. |
| `Failed to get services from Supervisor API` | Benign. The add-on asks Supervisor for a registered MQTT service; harmless when the broker is configured explicitly. |
| Network re-forms on every boot | Coordinator NVRAM not persisting — update the firmware before pairing. |
| Add-on won't start, port conflict | A `ports` entry collides with the original add-on. Set them `null` and use ingress. |
| Bridge unreachable after a while | C3's DHCP lease moved. Set a reservation. |

## Notes and limitations

- **Backup/restore reports `NOT_SUPPORTED`** on the stock Zigbee2MQTT. Pairing and
  routing are unaffected; only automated NVRAM snapshots are missing. For
  hardware-swap-without-re-pairing, upstream offers
  `ghcr.io/tostmann/zigbee2mqtt-esp32` — not currently packaged as an HA add-on.
- The `zboss` adapter is marked **experimental** in Zigbee2MQTT.
- One host at a time. Both USB and UART stay live, but two hosts talking at once
  is unsupported.
- **Never publish a compiled ESPHome binary** — it embeds your WiFi credentials
  and API key. `.gitignore` covers `*.bin`.

## Repository layout

```
esphome/
  zigbee-bridge.yaml              ESP32-C3 bridge config
  secrets.yaml.example            template — copy to secrets.yaml (gitignored)
  components/stream_server/       vendored, patched (GPL-3.0, see its README)
addon/
  config.json                     HA local add-on manifest for the 2nd instance
zigbee2mqtt/
  configuration.example.yaml      redacted Z2M config template
```

## Credits

- [tostmann/esp-coordinator](https://github.com/tostmann/esp-coordinator) — ZBOSS NCP coordinator firmware
- [oxan/esphome-stream-server](https://github.com/oxan/esphome-stream-server) — serial-to-WiFi ESPHome component (GPL-3.0)
- [zigbee2mqtt/hassio-zigbee2mqtt](https://github.com/zigbee2mqtt/hassio-zigbee2mqtt) — the add-on this manifest derives from

## License

GPL-3.0-or-later — see [LICENSE](LICENSE). This project redistributes a modified
copy of `stream_server` (© 2020-2023 Oxan van Leeuwen, GPL-3.0-or-later), so the
project as a whole carries that license.
