# Guitar-MFX Hardware Design Files

| Field | Value |
|---|---|
| Document status | Index of board design sources and fabrication outputs |
| Revision | 0.3 |
| Date | 2026-10-14 |
| Design tool | EasyEDA |

This folder holds PCB design sources and the exact files sent for manufacture. Requirements, board scope, and selections live in [docs/01-Hardware.md](../docs/01-Hardware.md); rationale in [docs/90-DecisionRecord.md](../docs/90-DecisionRecord.md); every bring-up and test result in [docs/03-Measurements.md](../docs/03-Measurements.md).

## Boards

Three product boards, selected 2026-10-14 ([90-DecisionRecord.md](../docs/90-DecisionRecord.md) ADR-013), plus one test fixture.

| Board | Folder | Purpose | Status |
|---|---|---|---|
| Front-panel board | `front-panel/` | ESP32-S3-WROOM-1-N16R8 module, display and touch FPC connectors, backlight driver, local 3.3 V rail, I2C I/O expander, encoder, footswitch and power-button connections | Selected; not started |
| Audio board | `audio-board/` | Guitar input, PGA, codec, Main out L/MONO + R (6.35 mm) and headphone (3.5 mm) outputs, Master and Phones potentiometers, mute relays and mute gate | Selected; not started; gated by P0-GATE |
| Main board | `main-board/` | 12 V DC input, power tree, push-button power controller, power-fail detector, Pi 5 power, panel USB-C, external footswitch and expression jacks, signal routing between the boards | Selected; not started; gated by P3-GATE |
| Display bring-up adapter | `display-adapter/` | Test fixture: plug-on adapter for the 44-pin ESP32-S3 development board that runs the display gate | Planned first PCB; whether front-panel rev A runs the display gate instead is open |

## Folder layout per board

```text
<board>/
├── README.md        purpose, revision history, links to docs and measurement rows
├── easyeda/         EasyEDA project export (Pro: .epro; Standard: JSON export)
├── fabrication/
│   └── rev-A/       exactly what was sent to the PCB / assembly house: Gerber zip, BOM, pick-and-place
└── outputs/         schematic PDF, PCB PDF, 3D renders
```

## Rules

- Folder names are lowercase with hyphens and match the board names in [01-Hardware.md](../docs/01-Hardware.md) §2.1.
- Revisions are letters: A, B, C. When a revision is ordered, copy the files sent to the manufacturer into `fabrication/rev-X/` and tag the commit `<board>-rev-X` (for example `front-panel-rev-A`).
- Never edit files inside an ordered `fabrication/rev-X/` folder. Any change after ordering is the next revision.
- Vendor datasheets for parts used on a board go in [docs/references/](../docs/references/README.md), not in the board folder.
- Measured results are recorded only in [03-Measurements.md](../docs/03-Measurements.md); a board README links to the rows.
