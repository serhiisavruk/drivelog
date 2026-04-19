# 2022+ Toyota GR86 / Subaru BRZ (ZN8/ZD8) — CAN bus access

**Date:** 2026-04-19
**Scope:** Verify claims about CAN access on the ZN8 platform and survey what's actually loggable, for a Pi-based data logger (drivelog).

---

## Claim verification

Each original claim assessed against web sources. Status levels: **VERIFIED**, **PARTIALLY VERIFIED**, **UNVERIFIED**, **CONTRADICTED**.

### ✅ Claim 1 — OBD-II port does not expose full CAN on 2022+
**Status: VERIFIED · Confidence: high**

- timurrrr/ft86 (gen2 CAN doc): "OBD-II port no longer exposes high-rate CAN data except OBD-II requests on 2022+ models."
- Ansix Auto: "2022+ BRZ/GR86 does not have CAN bus lines run to the onboard OBD-II port."
- Corroborated by multiple vendors (Ansix, Hachi, DauntlessOBD, Petrel, RaceCapture) all selling non-OBD taps explicitly for this reason.

**Nuance:** OBD-II still works for request/response (Mode 01/06/09 PIDs), just not for passive broadcast sniffing. Useful for DTCs and slow polling; useless for 100 Hz telemetry.

### ✅ Claim 2 — ASC connector is a valid tap; Ansix and Hachi sell harnesses
**Status: VERIFIED · Confidence: high**

- Ansix: tees off the Active Sound Control connector, TE 1376106-1, inside dash right of glovebox.
- Hachi Electronics: "2022+ GR86/BRZ ASC CAN Adapter" at $40.
- timurrrr confirms ASC as a tap point.

**Nuance:** "Full-rate" is an overstatement. The ASC bus is missing some IDs that are visible on the EC CAN (body) bus — notably **wheel speeds (0x13A)**, 0x143, and 0x2D2. The ASC bus is one logical domain; the gateway filters what crosses domains.

### ✅ Claim 3 — ASC harnesses can optionally delete the fake engine sound
**Status: VERIFIED · Confidence: high**

- Hachi sells two SKUs: "ASC Delete" (unplugs module, loses sound) and "ASC Retention" (pass-through, keeps sound).
- Ansix is pass-through/retention-only per their site.

### ⚠ Claim 4 — Connector behind passenger glovebox near cabin filter, used as CAN tap
**Status: PARTIALLY VERIFIED · Confidence: medium–high**

- timurrrr/ft86 confirms a DCM connector behind the glovebox near the cabin filter and 12V socket. Pinout: green=CAN-H, white=CAN-L. Accessible via the Toyota radio harness middle pins.
- Hachi refers to an "EC CAN connector behind the cigarette lighter" specifically for the wheel-speed data missing from ASC.

**Critical caveat (market-dependent):** This connector is populated only on **GR86 and non-US BRZ**. **US BRZs ship without the DCM/telematics module** — the connector is absent. Hachi sells a workaround via "GJP Designs DCM Delete" for US BRZ owners.

Terminology is inconsistent across sources ("DCM connector", "EC CAN", "connector behind cigarette lighter") — likely the same harness branch.

### ⚠ Claim 5 — Hidden 12V outlet in glovebox (Tom YouTube video, 2023-06-05)
**Status: PARTIALLY VERIFIED · Confidence: medium**

- Outlet exists: YouTube short https://www.youtube.com/shorts/bBJUHI566wo titled "Hidden Lighter Plug in the 2022+BRZ/GR86". Multiple forum sources confirm the 15A outlet in the glovebox.
- **Creator name "Tom" and exact date 2023-06-05 not independently verified** — YouTube metadata was not scrapable through WebFetch. One search result mentioned June 2023 without the specific day.

Conclusion: outlet is real and usable; the video attribution details are secondary and unconfirmed.

### ❌ Claim 6 — Network Gateway connector H62 on GR Zoo forum
**Status: CONTRADICTED — wrong car · Confidence: high**

- The GR-Zoo thread "CAN Bus Reverse Engineering" describes H62 as "empty plug Toyota generously left us by the RH footwell" connected to "Bus Buffer ECU (Bus 7)."
- **That thread is about the GR Yaris / GR Corolla, not GR86.** The GR Corolla equivalent is labeled I174. The only GR86 mention in the thread is a cross-link to an AIM thread that contains no H62 info.

**Do not assume H62 exists on a ZN8.** The GR86's equivalents are the ASC connector and the DCM/EC CAN connector behind the glovebox.

### ✅ Claim 7 — AIM Solo 2 DL and RaceCapture use a non-OBD tap on this car
**Status: VERIFIED · Confidence: high**

- Ansix product copy explicitly targets RaceCapture/Track and RaceCapture/Pro users.
- gr86.org forum has a thread "SOLVED: AIM Solo 2 DL ECU mapping in 2023 GR86 Premium" indicating real-world use (body gated behind tollbit; title visible).
- DauntlessOBD offers a hybrid OBD dongle + optional body-CAN adapter.

Some users pair an OBD dongle (for DTCs/config) alongside a CAN tap.

---

## CAN capabilities on GR86 (ZN8/ZD8)

### Topology and bitrate
- **Classical CAN @ 500 kbps** (community-consistent; not confirmed in an authoritative vendor doc — **measure before trusting**).
- **Multiple buses, minimum two**: Powertrain CAN (visible at ASC) and a Body/EC CAN (visible at DCM connector, carries wheel speeds). timurrrr distinguishes "B_CAN" vs EC CAN.
- **No evidence of CAN-FD** on this platform in any source reviewed.

### Signals on the powertrain bus (via ASC)
Source: timurrrr/ft86 gen2 doc (single-source; treat as reverse-engineering, validate in-car).

| Signal | CAN ID | Rate |
|---|---|---|
| RPM, throttle, neutral switch | 0x40 | 100 Hz |
| AT gear / mode | 0x48 | 100 Hz |
| Steering angle, yaw rate | 0x138 | 50 Hz |
| Vehicle speed, brake pressure | 0x139 | 50 Hz |
| Wheel speeds (4x) | 0x13A | 50 Hz — **not on ASC bus, only on EC CAN** |
| Lat / long G | 0x13B | 50 Hz |
| Gear, clutch (MT) | 0x241 | 20 Hz |
| Oil / coolant temp | 0x345 | 10 Hz |
| TPMS | 0x6E2 | 1 Hz |

**Gen2 packets include counter + checksum** (unlike gen1 FR-S / BRZ). Any logger must validate these before trusting values.

### Public DBC availability
- **Not in commaai/opendbc** — no GR86/BRZ/ZN8/ZD8 entry in CARS.md. openpilot does not support this platform.
- **RaceChrono equations** (not a formal DBC) in timurrrr/RaceChronoDiyBleDevice at `can_db/ft86_gen2.md`.
- **Autosport Labs / RaceCapture** has presets (referenced on Ansix page) but not a standalone DBC download.
- **No well-maintained public .dbc exists** — assemble one from timurrrr's tables.

### OBD-II coverage
- Standard Mode 01 PIDs work (RPM, speed, coolant, throttle, MAF, etc.).
- DauntlessOBD Enhanced exposes extra "BRZ/GR86 data channels" via UDS/manufacturer-specific requests and Mode 06. Full PID map is proprietary / not published.
- Key limitation: request/response only. Practical rates **~5–20 Hz total** on OBD polling, vs **~100 Hz** on passive CAN sniffing.

### Gateway / bypass specifics
- **US BRZ lacks the DCM** → no EC CAN connector behind the glovebox. Workaround: GJP Designs DCM Delete harness.
- **Wheel speeds (0x13A)** are filtered off the ASC bus. Only accessible via EC CAN.
- IDs 0x143 and 0x2D2 also missing from ASC.

### Vendor pinout publication
- **Ansix / Hachi**: plug-and-play black-box; no public pinout.
- **timurrrr/ft86 (community)** is the only public pinout source: ASC TE 1376106-1, blue=CAN-H, white=CAN-L; DCM middle pins, green=CAN-H, white=CAN-L.
  - ⚠ **Color conflict**: the verification agent reported blue=CAN-H for ASC, while some community material uses other conventions. Verify with a meter before wiring.
- **MoTeC, AiM, RaceCapture, HP Tuners**: none publish a GR86-specific CAN pinout; they rely on Ansix/Hachi harnesses for the physical tap.

---

## Bottom line for drivelog on a GR86

### Safe defaults (sourced)
1. **Do not rely on OBD-II for high-rate logging.** It's filtered. Use it for PID polling and DTCs only.
2. **Primary tap = ASC connector** (TE 1376106-1). 500 kbps classical CAN. Gets ~100 Hz engine data and 50 Hz chassis data.
3. **If wheel speeds are needed**, add a **second CAN channel on the EC CAN / DCM connector** behind the glovebox. Requires a two-channel SocketCAN setup (or a multi-channel USB-CAN adapter).
4. **Starter harness**: buy a **Hachi ASC pass-through** (~$40) to avoid cutting OEM wiring. Terminate to a connector that feeds the Pi's CAN interface.
5. **Power**: the glovebox 12V outlet is real (15A fuse). Fine for a Pi if shutdown is handled. Outlet is switched with ignition — behavior needs confirming per market.
6. **Decoding**: build a local `.dbc` from timurrrr's signal tables. Validate gen2 counter + checksum fields before trusting any value.

### Must measure in person before committing
- Confirm **the specific car/market** actually has the DCM connector populated (US BRZ does NOT).
- Confirm **bitrate with a scope/analyzer** (500 kbps is community assumption, not an authoritative doc).
- Confirm whether ASC bus and EC CAN are the same logical bus or separate domains bridged by the gateway (wheel-speed absence strongly suggests separate).
- Confirm **CAN-H / CAN-L wire colors** with a meter — color conventions conflict across sources.
- **Verify termination** before driving. Plugging a Pi node into an ASC pass-through can mis-terminate the bus. Measure end-to-end resistance (should be ~60 Ω) with ignition off.
- Sniff for CAN-FD just in case (unlikely but unconfirmed).

### Unresolved gaps
- No authoritative DBC exists — all decoding is community reverse-engineering.
- H62 is a GR Yaris/Corolla concept and does **not** apply to GR86.
- Exact OBD-II enhanced-PID list is vendor-proprietary (DauntlessOBD).
- "Tom" YouTube video date/creator couldn't be independently confirmed beyond the URL and title.

---

## Sources

- timurrrr/ft86 gen2 CAN doc: https://github.com/timurrrr/ft86/blob/main/can_bus/gen2.md
- timurrrr RaceChrono equations: https://github.com/timurrrr/RaceChronoDiyBleDevice/blob/master/can_db/ft86_gen2.md
- Ansix Auto: https://ansixauto.com/2022-brz-gr86-canbus-adapter/
- Hachi Electronics ASC adapter: https://hachielectronics.com/products/2022-gr86-brz-asc-can-adapter
- Petrel Data (reseller/tech notes): https://www.petreldata.com/product/ansix-auto-can-adapter-for-2022-brz-gr86/
- DauntlessOBD Enhanced: https://dauntlessdevices.com/product/dobd-gr/
- GR-Zoo CAN reverse engineering (GR Yaris, not GR86): https://www.gr-zoo.com/threads/can-bus-reverse-engineering.7834/
- Autosport Labs / RaceCapture blog: https://www.autosportlabs.com/toyota-86-subaru-brz-can-bus-data-steering-angle-brake-pressure-oil-temp-and-more/
- YouTube "Hidden Lighter Plug in the 2022+ BRZ/GR86": https://www.youtube.com/shorts/bBJUHI566wo
- commaai/opendbc CARS.md (no GR86 entry): https://github.com/commaai/opendbc/blob/master/docs/CARS.md
