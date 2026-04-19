# Car data ports & protocols — overview

**Date:** 2026-04-19
**Scope:** What the different connectors, buses, and application protocols in a modern car are and how they relate, for orienting the drivelog logger design.

---

## Physical ports

| Port | What it is |
|---|---|
| **OBD-II** | Mandatory 16-pin diagnostic port on all cars since ~1996 (US) / 2001 (EU gas) / 2004 (EU diesel). Standardized location under dash. The port itself is just a connector — what's *on* the pins depends on the car. |
| **DLC** | "Data Link Connector" — generic name for the OBD-II connector. Same thing. |
| **J1939 (9-pin Deutsch)** | Trucks/heavy equipment. Different connector, different protocol. Not relevant for passenger cars. |
| **Manufacturer-specific ports** | Behind dash, under seat, in engine bay — access to internal buses not exposed at OBD-II (e.g. GR86's ASC connector). |

## Low-level buses (the physical wires)

| Bus | Speed | Used for |
|---|---|---|
| **CAN** (Controller Area Network) | 125 kbit/s (LS-CAN) or 500 kbit/s (HS-CAN) | Dominant in-vehicle bus since ~2008. Broadcast, multi-master, differential pair (CAN-H / CAN-L). |
| **CAN-FD** | up to 5 Mbit/s data phase | Newer cars (mostly 2020+). Backward-ish compatible frame format, bigger payloads, higher speed. |
| **LIN** | 19.2 kbit/s, single-wire | Cheap slave devices (window switches, seat modules). Subordinate to a CAN node. |
| **FlexRay** | 10 Mbit/s, dual-channel | High-end chassis/x-by-wire (some BMW, Audi). Declining — CAN-FD and Ethernet replacing it. |
| **MOST** | 25–150 Mbit/s, fiber | Infotainment audio/video (older BMW, Mercedes). Being replaced by Automotive Ethernet. |
| **K-Line / ISO 9141 / KWP2000** | 10.4 kbit/s, single-wire | Pre-CAN diagnostics. Legacy; some cars kept it alongside CAN into the 2000s. |
| **Automotive Ethernet** (100BASE-T1 / 1000BASE-T1) | 100 Mbit/s – 1 Gbit/s | Modern cameras, ADAS, zonal architectures, OTA. Single twisted pair. Requires a media converter to talk to from a PC/Pi. |

**Rule of thumb:** For a data logger on a 2010–2020s passenger car, CAN first, CAN-FD if newer, OBD-II as a gateway. LIN / FlexRay / MOST rarely matter.

## Application-level protocols (what rides on CAN)

| Protocol | Purpose |
|---|---|
| **OBD-II (SAE J1979)** | Emissions diagnostics. Standardized PIDs, modes 01–0A. Request/response. What every ELM327 dongle speaks. |
| **UDS (ISO 14229)** | Unified Diagnostic Services. Manufacturer-specific diagnostics, flashing, coding. Request/response. Much richer than OBD-II. |
| **KWP2000 (ISO 14230)** | Older UDS ancestor. Still on some cars. |
| **ISO-TP (ISO 15765-2)** | Transport layer for fragmenting >8-byte OBD-II / UDS messages over CAN. Invisible to users; library handles it. |
| **J1939** | Truck / heavy equipment. Defines standardized IDs and signals on CAN for diesel engines, transmissions. Not used on passenger cars. |
| **CCP / XCP** | Calibration / measurement (tuners, dyno shops). High-rate ECU variable read/write. |
| **Manufacturer broadcasts** | Raw periodic CAN frames the ECUs send to each other — no protocol layer, just "ID 0x139 every 20 ms". Decoded with a DBC. *This is where the 100 Hz telemetry lives.* |

## OBD-II transport layers (the "5 protocols")

When "OBD-II supports 5 protocols" is said, this is what's meant — how OBD-II is physically carried:

1. **SAE J1850 PWM** — old Ford, pre-2003.
2. **SAE J1850 VPW** — old GM, pre-2003.
3. **ISO 9141-2** — European/Asian, pre-2004.
4. **ISO 14230 (KWP2000)** — transitional.
5. **ISO 15765 (CAN)** — everything 2008+ (US mandate) and most cars before that.

Any ELM327 auto-detects which one the car uses.

## What drivelog actually cares about

- **Passive CAN sniffing** on HS-CAN at 500 kbit/s — where the high-rate driving signals live.
- **ISO 15765 / OBD-II** over CAN — for standardized emissions PIDs and DTCs.
- **UDS** — optional, for manufacturer-specific data or ECU IDs.
- **CAN-FD** — only if the target car uses it (newer cars; GR86 does not).
