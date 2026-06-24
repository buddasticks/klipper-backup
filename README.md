# Sovol SV08 Stealth Changer — 4-Tool Dragon Burner Build

> **Status:** Active Development | **Last Updated:** June 2026

A fully custom multi-tool StealthChanger conversion of the Sovol SV08, replacing the stock toolhead system with four independent **Dragon Burner** toolheads, each running **Sherpa Mini** extruders and **Bambu Lab clone** hot ends. Toolheads are managed via **EBB36 v1.2** CAN boards, bridged through a **Hexa Distro Fusion** board at the rear of the printer.

---

## Table of Contents

- [Hardware Overview](#hardware-overview)
- [Electronics & Wiring](#electronics--wiring)
- [Firmware & Software](#firmware--software)
- [Toolhead Details](#toolhead-details)
- [Bed & Probe](#bed--probe)
- [Calibration & Tuning](#calibration--tuning)
- [Known Issues & Notes](#known-issues--notes)
- [Credits & Resources](#credits--resources)

---

## Hardware Overview

| Component | Specification |
|-----------|---------------|
| **Base Printer** | Sovol SV08 (Stock frame, motion system) |
| **Tool Changer** | StealthChanger kinematic mount system |
| **Toolheads** | 4× Dragon Burner |
| **Extruders** | 4× Sherpa Mini |
| **Hot Ends** | 4× Bambu Lab clone (all-metal, ~300°C capable) |
| **Toolhead Boards** | 4× BTT EBB36 v1.2 (CAN bus) |
| **CAN Bridge** | Hexa Distro Fusion board (rear-mounted) |
| **Main PSU** | 350W — powers stock mainboard, bed heater, and frame electronics |
| **Tool PSU** | 500W — powers Hexa Distro Fusion and all four toolheads |
| **Bed** | Upgraded Ram3n Graphite Bed |
| **Probe** | Cartographer v4 (mounted on shuttle) |
| **Main Board** | Stock SV08 mainboard (via USB-to-CAN bridge) |

### Key Modifications

- **Stock toolheads removed** — replaced with Dragon Burner + Sherpa Mini + Bambu clone hot end assemblies
- **Stock extruder system removed** — each toolhead is now fully independent with its own Sherpa Mini
- **Dual PSU setup** — 350W for mainboard/bed, 500W dedicated to toolheads via Hexa Distro Fusion
- **CAN bus topology** — EBB36 boards communicate over CAN, bridged to the stock mainboard via Hexa Distro Fusion
- **Probe on shuttle** — Cartographer v4 rides on the tool shuttle, not fixed to any single toolhead

---

## Electronics & Wiring

### Dual PSU Power Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        AC INPUT                              │
│                         │                                   │
│            ┌────────────┴────────────┐                     │
│            ▼                         ▼                     │
│      ┌──────────┐               ┌──────────┐               │
│      │  350W    │               │  500W    │               │
│      │   PSU    │               │   PSU    │               │
│      │ (Main)   │               │ (Tools)  │               │
│      └────┬─────┘               └────┬─────┘               │
│           │                          │                     │
│    ┌──────┴──────┐            ┌──────┴──────┐             │
│    ▼             ▼            ▼             ▼             │
│ SV08 Main    Bed Heater   Hexa Distro   (spare/aux)      │
│  Board                    Fusion Board                      │
│                              │                             │
│                         ┌────┼────┐                        │
│                         │    │    │                        │
│                        EBB  EBB  EBB  EBB                  │
│                        T0   T1   T2   T3                   │
│                       (Dragon Burner toolheads)             │
└─────────────────────────────────────────────────────────────┘
```

### PSU Breakdown

| PSU | Wattage | Powers | Notes |
|-----|---------|--------|-------|
| **Main PSU** | 350W | SV08 mainboard, bed heater (Ram3n), frame fans, display | Stock or upgraded; handles bed + motion |
| **Tool PSU** | 500W | Hexa Distro Fusion → 4× EBB36 → hot ends, Sherpa Minis, part fans | Dedicated; prevents main PSU overload during multi-tool heating |

> **Why dual PSUs?** Four 40W hot ends + four Sherpa Mini steppers + 8+ fans can pull 250W+ simultaneously. A single stock PSU would struggle. The 500W tool PSU ensures stable power for all toolheads, especially during multi-material prints where rapid tool swaps demand quick heat-up.

### CAN Bus Topology

```
Stock SV08 Mainboard (USB)
         │
         ▼
  Hexa Distro Fusion
  (USB-to-CAN Bridge, rear-mounted)
  (Powered by 500W Tool PSU)
         │
    ┌────┼────┐
    │    │    │
   EBB  EBB  EBB  EBB
   T0   T1   T2   T3
  (Dragon Burner toolheads)
```

### EBB36 v1.2 Configuration

Each EBB36 is configured with:
- **CAN Address:** Unique per toolhead (e.g., `canbus_uuid` assigned per tool)
- **Heater:** Bambu Lab clone hot end cartridge (powered by 500W tool PSU via Hexa Distro)
- **Thermistor:** Standard NTC 100k (verify type in config)
- **Part Cooling:** 2× 4010 or 5015 fans (depending on Dragon Burner variant)
- **Hotend Cooling:** 1× 3010 or 4010 heatsink fan
- **Probe:** *Not* on EBB — Cartographer is on the shuttle

### Hexa Distro Fusion

- Mounted at the **rear of the printer**
- Acts as the **central CAN bridge** between USB (mainboard) and the CAN bus
- **Powered by the 500W tool PSU** — distributes 24V to all four EBB36 boards
- Ensure proper 120Ω termination at both ends of the CAN bus

### Power Considerations

| Load | Est. Draw | Notes |
|------|-----------|-------|
| 4× Hot ends (40W ea) | ~160W | Peak when all heating simultaneously |
| 4× Sherpa Mini steppers | ~40W | Active during extrusion |
| 4× Heatsink fans | ~8W | Always on above threshold |
| 8× Part cooling fans | ~40W | Variable, peak during overhangs |
| EBB36 boards (×4) | ~20W | Idle + logic |
| **Total Tool Load** | **~270W** | Well within 500W PSU headroom |
| **Main Load** (bed + motion) | **~200W** | Within 350W PSU headroom |

> **Tip:** Use `SET_HEATER_TEMPERATURE` macros to stagger hot end warm-ups. Heating all four from cold at once creates a ~160W spike; the 500W PSU handles it, but your mains wiring and breaker will appreciate staged heating.

---

## Firmware & Software

### Host: Klipper (Mainsail/Fluidd)

This printer runs **Klipper** on the stock SV08 mainboard, with the following key components:

| Component | Firmware | Power Source |
|-----------|----------|--------------|
| SV08 Mainboard | Klipper (stock MCU) | 350W Main PSU |
| EBB36 v1.2 (×4) | Klipper (CAN bus) | 500W Tool PSU via Hexa Distro |
| Hexa Distro Fusion | Klipper (USB-to-CAN bridge) | 500W Tool PSU |
| Cartographer v4 | Klipper (probe as secondary MCU or via mainboard) | 350W Main PSU or tool PSU |

### Key Klipper Config Sections

- `[mcu]` — Stock mainboard
- `[mcu can0]` through `[mcu can3]` — EBB36 boards (one per tool)
- `[mcu cartographer]` — Cartographer probe MCU (if separate)
- `[extruder]`, `[extruder1]`, `[extruder2]`, `[extruder3]` — Four tool extruders
- `[heater_bed]` — Ram3n graphite bed
- `[probe]` or `[cartographer]` — Cartographer v4 probe configuration
- `[tool]` / `[toolchanger]` — Tool change macros and parking

### Toolchanger Plugin

This build uses a **toolchanger plugin** (e.g., `ktc-easy` or standard Klipper toolchanger) to manage:
- Tool parking positions
- Z-offset per tool
- Temperature management during swaps
- Fan control per tool

> **Note:** Tool offsets (X/Y/Z) must be calibrated per toolhead and stored in the tool configuration.

---

## Toolhead Details

### Dragon Burner

The Dragon Burner is a compact, high-flow toolhead designed for Voron-style printers. Key specs for this build:

- **Hot end:** Bambu Lab clone (all-metal, bi-metallic heatbreak recommended)
- **Extruder:** Sherpa Mini (geared, ~3.5:1 ratio, lightweight)
- **Cooling:** Dual part cooling + single heatsink fan
- **Mount:** StealthChanger kinematic dock interface

### Sherpa Mini

- **Motor:** NEMA 14 pancake (typically 10–14mm stack)
- **Gear ratio:** 3.5:1 or 5.44:1 (variant dependent)
- **Filament path:** Short, direct drive
- **Weight:** ~120–150g assembled (lightweight for fast tool changes)

### Bambu Lab Clone Hot End

- **Max temp:** ~300°C (verify thermistor and heater cartridge ratings)
- **Nozzle:** V6-style or proprietary (check your clone variant)
- **Heatbreak:** All-metal, bi-metallic recommended for reliability
- **Cooling:** Requires active heatsink fan above ~240°C

---

## Bed & Probe

### Ram3n Graphite Bed

- **Material:** Composite graphite (excellent thermal conductivity and flatness)
- **Surface:** Typically PEI or bare graphite (verify your variant)
- **Heating:** Stock SV08 bed heater, powered by **350W main PSU**
- **Thermal expansion:** Very low — excellent for mesh bed leveling

### Cartographer v4 Probe

- **Mounting Location:** On the **tool shuttle** (not on any individual toolhead)
- **Function:** Mesh bed leveling, Z-homing, bed tilt compensation
- **Advantage:** Single probe for all tools — no per-tool probe offset needed
- **Integration:** Communicates via CAN or dedicated MCU connection

> **Important:** Since the probe is on the shuttle, ensure your tool change macros move the shuttle (with a tool docked or bare) to probe positions. The probe must be active regardless of which tool is loaded.

---

## Calibration & Tuning

### Required Calibrations

1. **CAN Bus Setup**
   - Flash each EBB36 with unique CAN UUIDs
   - Verify all four toolheads appear in `ls /dev/serial/by-id/` or via `canbus_query.py`

2. **Tool Offsets**
   - Measure X/Y/Z offset for each tool relative to Tool 0 (or the probe)
   - Store in `[tool T0]` through `[tool T3]` config sections
   - Use a calibration cube or dial indicator for precision

3. **Z-Offset Per Tool**
   - Each Bambu clone hot end may sit at a slightly different Z height
   - Calibrate using the Cartographer probe or feeler gauge

4. **Extruder Calibration**
   - E-steps/mm for each Sherpa Mini (they should be close, but verify)
   - Rotation distance in Klipper config

5. **PID Tuning**
   - Run `PID_CALIBRATE` for each hot end
   - Run `PID_CALIBRATE` for the bed (Ram3n graphite may have different thermal mass)

6. **Bed Mesh**
   - Generate bed mesh with Cartographer v4
   - Save and load in `PRINT_START` macro

7. **Tool Change Macros**
   - Test parking/docking for all four tools
   - Verify filament cutting/purging (if equipped)
   - Tune wipe tower or prime line settings for multi-material prints

---

## Known Issues & Notes

| Issue | Status | Notes |
|-------|--------|-------|
| **Dual PSU ground reference** | ⚠️ Verify | Ensure both PSUs share a common ground. Tie DC grounds together if not already bonded through the frame. |
| **500W tool PSU load** | ✅ Good | Handles ~270W peak tool load comfortably. ~230W headroom for inrush/spikes. |
| **350W main PSU load** | ✅ Good | Bed (~200W) + mainboard/motors (~50W) = ~250W. ~100W headroom. |
| **CAN bus termination** | ✅ Critical | Verify 120Ω resistors at both ends of the CAN bus (Hexa Distro + last EBB36). |
| **Tool offset drift** | ⚠️ Check | Kinematic mounts can drift. Re-check offsets after crashes or maintenance. |
| **Cartographer on shuttle** | ✅ Working | Ensure probe macros account for shuttle position, not toolhead position. |
| **Bambu clone nozzle compatibility** | ⚠️ Verify | Some clones use proprietary nozzles; verify you have spares. |
| **Sherpa Mini gear wear** | 🔧 Maintenance | Check gear tension periodically; lightweight extruders can slip if loose. |

### Tips

- **Label your CAN cables AND PSU rails** — With four EBB36 boards and two PSUs, tracing wires is much easier with labels.
- **Use a CAN bus analyzer** (or `canbus_query.py`) to debug communication issues.
- **Keep a spare EBB36 flashed** — Having a hot-swappable spare saves downtime.
- **Document your offsets** — Save them in this README or a separate `offsets.md`.
- **Monitor PSU temps** — The 500W PSU at the rear may need airflow. Consider a 40mm fan if it runs warm.

---

## File Structure (Repo)

```
sovol-sv08-stealth-changer/
├── README.md                 # This file
├── config/
│   ├── printer.cfg           # Main Klipper config
│   ├── moonraker.conf        # Moonraker/Fluidd config
│   ├── macros/
│   │   ├── print_start.cfg   # PRINT_START, PRINT_END
│   │   ├── toolchange.cfg    # Tool change macros
│   │   ├── probe.cfg         # Cartographer probe macros
│   │   └── calibration.cfg   # PID, bed mesh, etc.
│   └── tools/
│       ├── tool_t0.cfg       # EBB36 T0 config
│       ├── tool_t1.cfg
│       ├── tool_t2.cfg
│       └── tool_t3.cfg
├── docs/
│   ├── wiring_diagram.md     # Detailed wiring notes
│   ├── psu_notes.md          # Dual PSU setup and load calculations
│   ├── offsets.md            # Measured tool offsets
│   └── tuning_notes.md       # PID, pressure advance, etc.
├── stl/
│   └── (custom mounts, brackets, etc.)
└── scripts/
    └── (helper scripts for calibration, backup, etc.)
```

---

## Credits & Resources

- **Sovol SV08** — Base printer platform
- **StealthChanger** — Kinematic tool changer system
- **Dragon Burner** — Toolhead design community
- **Sherpa Mini** — Annex Engineering / community extruder design
- **BTT EBB36** — BigTreeTech CAN toolhead board
- **Hexa Distro Fusion** — CAN distribution board
- **Cartographer** — Probe system by Cartographer3D
- **Ram3n** — Graphite bed upgrade
- **Klipper** — 3D printer firmware by KevinOConnor
- **Klipper-Backup** — GitHub backup utility for Klipper configs

### Useful Links

- [Klipper Documentation](https://www.klipper3d.org/)
- [StealthChanger GitHub](https://github.com/StealthChanger)
- [Cartographer3D Documentation](https://docs.cartographer3d.com/)
- [BTT EBB36 GitHub](https://github.com/bigtreetech/EBB)
- [Klipper-Backup-Install](https://github.com/Staubgeborener/klipper-backup)

---

## License

Hardware modifications and configurations are shared as-is for educational and personal use. Respect the licenses of individual projects (StealthChanger, Dragon Burner, etc.) when redistributing derived works.

---

> **Maintainer:** [Your Name/GitHub Handle]  
> **Questions?** Open an issue or discussion on this repo.
