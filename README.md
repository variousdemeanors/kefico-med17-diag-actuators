# kefico-med17-diag-actuators

A reference for the **KWP2000 actuator-test diagnostic protocol** on **Kefico MED17.9.8 /
MEDG17.9.8** ECUs (Infineon TC1767, used in several Hyundai/Kia platforms) — the standard
service, framing, and the specific Local Identifiers (LIDs) this ECU accepts for actuator
tests and indicator outputs.

## Provenance / method

Found by **passively observing OBD-II CAN bus traffic** (11-bit IDs, 500 kbit/s, OBD-II pins
6/14) on a Hyundai Veloster 1.6 T-GDI while running a commercial scan tool's (Scanmatik 2 Pro)
actuator-test feature. This is **not** reverse engineering of that tool's software — it's
observation of standard diagnostic traffic on the vehicle's own bus. The service (KWP2000 /
ISO 14230 `0x30` InputOutputControlByLocalIdentifier) is a standardized diagnostic service; the
specific LID assignments are the ECU's/vehicle manufacturer's own diagnostic implementation,
discovered the same way any OBD-II scan tool or diagnostic reference does — by watching what
goes over the wire on a vehicle's own diagnostic port.

## Protocol

- **Service:** KWP2000 `0x30` InputOutputControlByLocalIdentifier. Request `0x7E0` → reply
  `0x7E8`, 11-bit CAN IDs, ISO-TP single frames.
- **Session open:** `10 90` → `50 90` (manufacturer session `0x90`). Some tools send a
  functional `10 81` / `20` first — replay that if the ECU refuses `0x90` directly.
- **Keep-alive:** `3E 00` (TesterPresent) at least every **1 s** while a test is active; also
  kept alive by live-data polling.
- **Live data:** `21 00`…`21 0E` sweep (~14 requests per ~0.44 s sweep).
- **Supported-LID probe:** `30 xx 0A` (`xx` = `00`/`20`/`40`/`60`/`80`) returns a
  PIDs-supported-style **32-bit bitmap** per group — the exact actuator LIDs this ECU accepts.
  Groups confirmed: `01–20`, `21–40`, `41–60`, `61–80`, `81–A0`.
- **Start a test:** `04 30 <LID> 07 <VV>` (ISO-TP single frame). `07` = shortTermAdjustment.
  `VV` = value: **`FF`** = on/max for most LIDs; **`0x50`** for Fan PWM (partial duty);
  O2-heater "additive correction" LIDs use **`7F`/`80`** (center = no adjust).
- **Stop a test:** `04 30 <LID> 07 00`. **Always send an explicit stop** — some tools do not
  send `ReturnControlToECU`; stopping to `00` is what actually releases the actuator.
- **Replies:** positive `03 70 <LID> 07`; refused `03 7F 30 22`
  (`conditionsNotCorrect`, e.g. an injector-disable request with the engine off).
- **ECU latency:** ~4–20 ms per request.

## Actuator LID table (engine OFF unless noted)

| LID | Function | Start value | Notes |
|----|----------|------|------|
| `0x1C` | Fan PWM | `0x50` | only one not `FF`; partial duty |
| `0x23` | Canister Shut-Off Valve (toggling) | `FF` | |
| `0x24` | Canister Shut-Off Valve (leak test) | `FF` | |
| `0x41`–`0x44` | Ignition Coil Enable/Disable #1–4 | `FF` | |
| `0x49`–`0x4C` | Injector Disable Cyl 1–4 | `FF` | **refused engine-off (NRC `0x22`); needs engine running** |
| `0x91` | Turbo Wastegate (WGT) | `FF` | |
| `0x92` | Turbo dump / Recirc valve (RCV) | `FF` | |
| `0x11` | A/C Compressor Relay | `FF` | status label unreliable on the tool used — verify physically |
| `0x12` | Fuel Pump Relay (1st stage) | `FF` | status label unreliable on the tool used — verify physically |
| `0x13` | Fuel Pump Control (1st stage) | `FF` | |
| `0x25` | Canister Purge Valve | `FF` | |
| `0x22` | Fuel Open Loop | `FF` | |
| `0x32` | ETC (throttle) Motor | `FF` | |
| `0x51` / `0x52` | CVVT Intake / Exhaust | `FF` | |
| `0x55` | Oil Control Valve — Intake Bank 1 | `FF` | |
| `0x61` / `0x63` / `0x65` | O2 Heater additive correction (B1S1 / B1S2 / all) | `7F` | trim-type; `80` = center/no-adjust |
| `0x01`/`0x03`/`0x06`/`0x07`/`0x08` | MIL / Cruise Main / Cruise SET / Overspeed / Oil-change lamps | `FF` | indicator outputs |

**Full accepted-LID set from the bitmap probe** (names not yet mapped for the rest — PRs/issues
with confirmed mappings welcome):
`01, 03, 06, 07, 08, 11, 12, 13, 16, 1C, 22, 23, 24, 25, 31, 32, 41–44, 49–4C, 51, 52, 55, 56,
61, 63, 65, 70, 71–74, 79, 7C, 7F, 91, 92`

## ⚠️ Safety

- **Always send the explicit `04 30 <LID> 07 00` stop** on: operator release, timeout, or any
  abort. Nothing should be left energized/commanded.
- **Don't trust an "ok"/"stopped" label from any tool as proof of state** — on the capture this
  was taken from, the A/C relay and fuel pump LIDs reported "stopped" while still physically
  actuating. Verify by live data and/or physical observation.
- Keep `3E 00` (TesterPresent) flowing at least every 1 s while any test is active, or the
  session (and the actuator) can be left in an undefined state.
- Engine-running-only LIDs (injectors `0x49`–`0x4C`) will refuse with NRC `0x22` off-engine —
  that's expected, not a bug.
- Use only on a vehicle you own or are authorized to service. Actuating ignition/fuel/drivetrain
  outputs carries real physical risk — know what you're commanding before you send `FF`.

## Scope

This repo documents the protocol and the LID table only. It does not include, and will not
accept contributions containing, anything about ECU flashing, SecurityAccess/seed-key
authentication, checksum/CRC-correction math, or any write/flash capability — this is a
read/actuate diagnostic reference, nothing more.

## Contributing

If you've confirmed this protocol or additional LID mappings on a MED17.9.8/MEDG17.9.8 vehicle
(same platform or different), please open an issue or PR with: the vehicle/ECU, the LID(s), and
what you observed. Unmapped LIDs from the bitmap above are the most useful gap to fill.

## License

MIT — see `LICENSE`. This is an original write-up of observed, standard-protocol diagnostic
traffic; no proprietary software, database, or third-party source code is included or was
reverse engineered to produce it.
