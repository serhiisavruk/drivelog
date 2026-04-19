# CAN bus: extractable data, throughput, multi-bus access

**Date:** 2026-04-19
**Questions:**
1. What can we extract from CAN? List all options.
2. How many datapoints per second can we record?
3. Car has 3–5 CAN buses — can we tap all of them? If MCP2515 is a bottleneck, can we switch to USB?

---

## 1. What can be extracted from CAN

Depends heavily on vehicle make/model/year and which bus is tapped (HS-CAN powertrain, MS-CAN body, CAN-FD, OBD-II gateway).

### Powertrain / engine
- Engine RPM
- Engine load (calculated / absolute)
- Throttle position (pedal + throttle body)
- Manifold absolute pressure (MAP)
- Mass airflow (MAF)
- Intake air temperature
- Coolant temperature
- Oil temperature / oil pressure (some vehicles)
- Fuel level, fuel rail pressure, fuel trims (short/long term)
- Lambda / O2 sensor voltages
- Ignition timing advance
- Turbo boost pressure (if equipped)
- Exhaust gas temp (some)
- DPF / regen status (diesels)
- AdBlue / DEF level (diesels)

### Transmission / drivetrain
- Gear position (current / requested)
- Transmission fluid temp
- Torque converter lockup state
- Clutch position (manuals with sensor)
- Drive mode (Eco/Sport/etc.)

### Vehicle dynamics
- Vehicle speed (wheel-speed-derived)
- Individual wheel speeds (4x)
- Steering wheel angle + rate
- Yaw rate
- Lateral / longitudinal acceleration
- Brake pressure, brake pedal switch
- ABS / ESC / TCS activity flags
- Parking brake status

### Fuel & economy
- Instantaneous fuel consumption
- Trip fuel / distance counters
- Range remaining
- Average speed / average consumption

### Electrical / hybrid / EV
- Battery voltage (12V)
- Alternator load
- HV battery SoC, SoH, pack voltage/current, cell temps
- Regen braking power
- Motor RPM / torque
- Charging state, charge rate, charger type

### Body / comfort
- Door / hood / trunk open states
- Seatbelt status
- Window positions
- HVAC settings, cabin temp, outside air temp
- Interior / exterior lights state
- Wiper state, rain sensor
- Horn, turn signals, hazards

### Driver assistance / safety
- Cruise control state + set speed
- ACC target distance
- Lane-keep / lane-departure events
- Collision warnings, AEB activations
- Blind-spot detections
- Parking sensor distances
- Airbag / crash status
- TPMS pressures + temps per tire

### Identification / diagnostics
- VIN
- Odometer
- Ignition state (off/acc/run/start)
- Key presence / remote ID
- DTCs (current + pending + permanent) via UDS/OBD-II
- Freeze-frame data
- ECU software/hardware part numbers
- Service interval counters

### Raw / meta
- Any arbitrary CAN ID payload (for custom decoding / reverse engineering)
- Bus load, frame timing, error frames
- Network management messages

### Caveats
- Most non-OBD PIDs need a DBC file or reverse-engineering per vehicle — they're not standardized.
- OBD-II Mode 01 PIDs are standardized but limited (mostly emissions-related).
- Manufacturer-specific data usually requires UDS requests on a diagnostic CAN ID.
- Some modern cars firewall sensitive buses behind a gateway — only a subset is visible without bypass.

---

## 2. Throughput — how many datapoints per second

### Bus physical limits (what's on the wire)

| Bus type | Bitrate | Max frames/sec (theoretical) | Typical real load |
|---|---|---|---|
| LS-CAN (body/comfort) | 125 kbit/s | ~2,000 | 500–1,500 fps |
| HS-CAN (powertrain) | 500 kbit/s | ~8,500 | 2,000–5,000 fps |
| HS-CAN high-load | 500 kbit/s | ~8,500 | up to ~6,000 fps |
| CAN-FD | 2–5 Mbit/s data phase | 20,000–40,000+ | 5,000–15,000 fps |

### Frames vs. signals vs. datapoints

One frame ≠ one datapoint. A single 8-byte frame commonly packs **4–10 signals** (RPM, coolant temp, throttle, etc.).

On a typical HS-CAN at ~3,000 fps this is **~15,000–30,000 signal updates/sec** — but most signals repeat at fixed cycle times, so unique-value updates are lower.

### Per-signal cycle times (typical)

| Signal class | Cycle | Rate |
|---|---|---|
| Engine RPM, throttle, wheel speeds | 10 ms | 100 Hz |
| Steering angle, yaw, accel | 10–20 ms | 50–100 Hz |
| Coolant temp, fuel level | 100–1000 ms | 1–10 Hz |
| Odometer, VIN, ambient temp | 1000 ms+ | ≤1 Hz |
| OBD-II polled PIDs | request/response | ~20–50 Hz total budget |

**OBD-II Mode 01 polling is slow** — request/response serialized. Expect **20–50 PIDs/sec total** across all IDs. Passive sniffing of the vehicle's own broadcasts is ~100× faster but requires a DBC.

### Pi-side interface limits

| Interface | Practical ceiling |
|---|---|
| MCP2515 via SPI (waveshare hat) | ~4,000–5,000 fps before drops |
| USB-CAN (CANable, PCAN-USB, Kvaser) | full bus rate, no issue |
| RP1/native SocketCAN on CM4/Pi 5 + transceiver | full bus rate |
| Two MCP2515s on one SPI bus | ~half each, interrupts contend |

SocketCAN + `candump -L` or python-can can handle full 500 kbit/s HS-CAN on a Pi 4/5 easily. MCP2515 is the common bottleneck.

### Storage cost

Raw logging with timestamp + ID + 8 bytes ≈ **24–32 bytes/frame**:

| Rate | Per hour | Per 8-hr day |
|---|---|---|
| 1,000 fps | ~100 MB | ~800 MB |
| 3,000 fps | ~300 MB | ~2.4 GB |
| 6,000 fps | ~600 MB | ~4.8 GB |

Compressed (gzip/zstd) typically 5–10× smaller because CAN payloads repeat heavily.

### Practical targets for drivelog

- **Passive sniffing on one HS-CAN**: plan for **2,000–5,000 frames/sec** sustained.
- **Decoded signals** (if a DBC is available): **10,000–30,000 signal samples/sec**.
- **OBD-II polling only (no DBC)**: **20–50 datapoints/sec total**, which is what most consumer dongles do.
- **GPS**: separate, usually 1–10 Hz.

---

## 3. Multi-bus access and MCP2515 → USB

### Tapping all 3–5 buses

Physically yes, but:

- **Access**: Only *one* bus (usually HS-CAN powertrain) is exposed on the OBD-II port, and on many post-2018 cars even that is gatewayed/filtered. To reach MS-CAN, body-CAN, chassis-CAN, infotainment-CAN, splicing into wiring behind the dash or at specific modules is typically required. That's invasive and can void warranty / trip intrusion detection.
- **Gateway**: Most modern cars have a central gateway ECU that firewalls the buses. The OBD port often only shows diagnostic traffic, not raw broadcast frames. Bypassing requires either a known gateway unlock (some VAGs, Fords) or tapping upstream of the gateway.
- **One transceiver per bus**: each bus needs its own CAN controller + transceiver pair. Can't share.
- **Bus termination**: don't add a 120 Ω terminator when splicing into an existing bus — it would create three terminators and distort signals. High-impedance tap only.

### Switching MCP2515 → USB

Usually the right call for multi-bus logging:

| Option | Channels | Notes |
|---|---|---|
| CANable 2.0 / CANtact (slcan/candleLight) | 1 | ~$30, works as SocketCAN, handles full 1 Mbit/s |
| Innomaker USB2CAN | 1 | SocketCAN native, solid on Pi |
| PCAN-USB (Peak) | 1 or 2 | industry standard, $$$ |
| Kvaser Leaf / U100 | 1 | pro-grade, expensive |
| **Multi-channel**: Kvaser USBcan Pro, PCAN-USB Pro FD, CL2000 | 2–4 | best for logging several buses simultaneously |
| **CAN-FD capable**: CANable 2.0, PCAN-USB FD, Kvaser U100 | 1+ | needed for newer vehicles |

For a Pi logger, pragmatic setup:

- 1–2× CANable 2.0 (cheap, SocketCAN) on USB, or
- 1× multi-channel USB adapter for 3+ buses from one device.

USB bandwidth is nowhere near the limit — USB 2.0 full-speed is ~12 Mbit/s, more than enough for several 500 kbit/s CAN buses combined. The real bottleneck shifts to **kernel interrupt handling + logging code**, not transport. Pi 4/5 handle multiple SocketCAN interfaces at full rate without issue.

### Alternatives to MCP2515 (if staying HAT-based)

- **MCP2518FD** (CAN-FD capable, SPI) — faster, larger FIFOs, still SPI but much better than MCP2515.
- **Pi 5 / CM4 with native CAN via RP1 + external transceiver** — no SPI bottleneck.
- **Dual/quad CAN HATs** (Waveshare, Seeed, Copperhill) — usually still MCP2515-based, so same ceiling per channel.

### Recommendation for drivelog

- Start with **1× USB CAN-FD adapter (CANable 2.0)** on the OBD-II port — covers 80% of cars and all the interesting powertrain/chassis data.
- Design the logger to accept **N SocketCAN interfaces** (`can0`, `can1`, …) from day one, so adding a second bus later is config, not rewrite.
- Only splice into additional buses if there's a specific data gap that can't be filled from the OBD-II-exposed bus.
