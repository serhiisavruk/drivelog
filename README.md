# drivelog

Always-on Raspberry Pi data logger for cars. Boots with the ignition, records CAN bus + GPS to local storage, syncs to a home server over Wi-Fi when in range.

Designed as a "silent black box" — zero interaction required. Plug it in and forget it.

## Goals

- **Auto on/off with ignition** — no buttons, no apps to open
- **Log everything, always** — every drive, no decisions about what to record
- **Safe shutdown** — no SD card corruption when power drops
- **Offline-first** — logs to local storage; upload is best-effort when Wi-Fi is around
- **Car-agnostic** — CAN bitrate and decoding are config, not code
- **Future-proof data** — raw CAN captured; decoding happens later, never in the car

## Non-goals

- Real-time telemetry or live track-day display (use AiM/RaceChrono for that)
- Lap timing, predictive timing, sector splits
- Cloud telemetry over 4G (optional future add-on, not core)
- In-car dashboard / touchscreen

## Architecture

```
                    ┌──────────────────────────┐
  12V constant ────▶│ Power mgmt board         │
                    │ (MausBerry / UPS HAT)    │
  12V accessory ───▶│  • 12V→5V buck           │
                    │  • ignition sense GPIO   │
                    │  • safe-shutdown trigger │
                    └──────────┬───────────────┘
                               │ 5V + GPIO
                               ▼
                    ┌──────────────────────────┐
   Car CAN bus ────▶│ CAN HAT (MCP2515)        │─▶ can0 (SocketCAN)
                    │  PiCAN2 / Waveshare      │
                    ├──────────────────────────┤
        GPS ───USB──│ Raspberry Pi 4/5         │
                    │                          │
                    │  systemd: drivelog.service│
                    │    • candump -l → .log   │
                    │    • gpspipe → .nmea     │
                    │    • logrotate on exit   │
                    │                          │
                    │  systemd: drivesync.service│
                    │    • rsync when Wi-Fi up │
                    └──────────┬───────────────┘
                               │ Wi-Fi (home SSID)
                               ▼
                    ┌──────────────────────────┐
                    │ Home server              │
                    │  • rsync target          │
                    │  • decode job (DBC)      │
                    │  • InfluxDB + Grafana    │
                    └──────────────────────────┘
```

## Hardware

| Part | Purpose | ~Price (USD) |
|---|---|---|
| Raspberry Pi 4 (4GB) or Pi 5 | Main computer | $55–80 |
| microSD 64GB (A2, high-endurance) | OS + log buffer | $15 |
| PiCAN2 HAT (or Waveshare MCP2515 HAT) | CAN 2.0 interface | $15–50 |
| USB GPS (u-blox 7/8 or equivalent) | Position + time sync | $15–30 |
| MausBerry car power switch (or UPS HAT) | 12V→5V + ignition sense + safe shutdown | $20–40 |
| OBD-II → DB9 CAN cable (or pigtail) | Car ↔ HAT | $10–20 |
| USB SSD (optional, 128GB+) | Durability + write endurance | $25–40 |
| Small enclosure + mounts | Vibration / heat | $10–20 |

**Total: ~$150–250.**

### Notes on car-side CAN access

Different cars expose CAN differently. Examples:

- **2022+ Toyota GR86 / Subaru BRZ:** CAN is **not** on the OBD-II port. Tap the chassis CAN via the Ansix Auto adapter behind the glove box.
- **Most modern cars (2008+):** High-speed CAN is on OBD-II pins 6 (CAN-H) / 14 (CAN-L) directly.
- **Older / weirder cars:** May need a custom tap; project is CAN-generic as long as the HAT can see the bus.

Bitrate is per-car (typically 500 kbps for passenger vehicles) — configured, not hardcoded.

## Runtime behavior

### Boot flow

```
Ignition ON
  → accessory 12V rises
  → power board enables 5V to Pi
  → Pi boots (5–15 s)
  → drivelog.service starts
      - brings up can0 at configured bitrate
      - starts candump -l (rotating .log files)
      - starts gpspipe (NMEA → .nmea files)
      - begins drive session (session id = timestamp)
```

### Shutdown flow

```
Ignition OFF
  → accessory 12V drops
  → power board pulses shutdown-request GPIO
  → drivelog.service stops:
      - flushes and closes current log files
      - appends session manifest (start/end, file list, checksums)
      - optionally gzips the session directory
  → systemd shutdown -h now
  → power board holds 5V until Pi is down (or timeout ~30 s)
  → 5V cut, car drains nothing
```

### Sync flow

Runs independently of ignition, on boot and on NetworkManager "connectivity up" events:

```
Wi-Fi connects
  → if SSID ∈ allowed-list (e.g. home)
    → rsync --partial --append-verify sessions/ → server:drivelog/
    → on success: mark synced; optionally prune after N days
```

## Storage format

One directory per drive session:

```
sessions/
  2026-04-19T10-14-52Z/
    can.00001.log.gz       # candump -l ASCII, rotated
    can.00002.log.gz
    gps.00001.nmea.gz      # raw NMEA from u-blox
    session.json           # start, end, vehicle_id, bitrate, file list, sha256
```

**Why raw?** Decoding in the car means discarding frames we didn't know we cared about. Decoding on the server means we can re-run analyses years later as the DBC grows.

Disk usage rule of thumb: ~1–5 MB/minute for full CAN + GPS, compressed. A 64 GB card holds months of driving.

## Server side

Out of scope for this repo's initial commits, but the contract is simple:

- **Inbox:** rsync target at `drivelog/sessions/`
- **Decode job:** cron/systemd timer runs `cantools decode` with a car-specific DBC → Parquet
- **Query:** InfluxDB (time-series) + Grafana, or DuckDB over Parquet for ad-hoc

The decode DBCs live in `dbc/<vehicle_id>/` and are versioned with the data so old sessions stay decodable.

## Configuration

Single `drivelog.yaml` on the Pi, e.g.:

```yaml
vehicle_id: gr86-2023
can:
  interface: can0
  bitrate: 500000
gps:
  device: /dev/ttyACM0
storage:
  root: /var/drivelog/sessions
  rotate_mb: 32
  gzip_on_close: true
sync:
  allowed_ssids: ["home-5g"]
  target: user@server.local:/srv/drivelog/inbox
  delete_after_sync_days: 30
```

## Status

**Design phase.** This repo currently contains the architecture and BOM only — no code yet. Next milestones:

- [ ] Bench setup — Pi + CAN HAT + USB GPS, confirm SocketCAN comes up
- [ ] `drivelog.service` — candump + gpspipe + rotation, no sync yet
- [ ] Power board integration — ignition-sense GPIO + safe shutdown
- [ ] `drivesync.service` — rsync on Wi-Fi up
- [ ] First real car install — verify boot/shutdown cycle on ignition
- [ ] Server-side decode pipeline
- [ ] DBC for first target vehicle (2023 GR86)

## Prior art worth reading

- [`JonnoFTW/rpi-can-logger`](https://github.com/JonnoFTW/rpi-can-logger) — closest existing project; systemd-based CAN+GPS logger with Wi-Fi upload. Good starting point or reference.
- [AutoPi](https://www.autopi.io/) — commercial all-in-one dongle with OBD-II + 4G + cloud. Relevant for comparison.
- [SavvyCAN](https://github.com/collin80/SavvyCAN) — desktop CAN analyzer / DBC editor for the decode side.

## License

TBD.
