# Guitar-MFX Reference Library

| Field | Value |
|---|---|
| Document status | Index of vendor documents stored in the repository |
| Revision | 0.2 |
| Date | 2026-10-08 |

This folder stores vendor datasheets, application notes, and mechanical drawings for parts the project has selected or purchased. These files are **vendor specifications**, not Guitar-MFX results. Selection decisions belong in [01-Hardware.md](../01-Hardware.md), rationale in [90-DecisionRecord.md](../90-DecisionRecord.md), and measured behaviour in [03-Measurements.md](../03-Measurements.md).

## Layout

```text
references/
└── datasheets/
    └── <category>/          # display, audio-codec, power, mcu, connectors, ...
        └── <PartNumber>_<DocType>_Rev<x.y>.pdf
```

- Keep the vendor's exact part number at the start of the file name.
- Record the vendor document revision in the file name and in the index below.
- When a vendor issues a new revision, add the new file and keep the old one until no design depends on it.

## Index

| Category | Part | Document | Vendor | Doc rev / date | Status | Used for |
|---|---|---|---|---|---|---|
| Display | TL050WVH12CT-B1714A — 5.0" 800×480 IPS RGB TFT with GT911 CTP | [LCD module specification (final)](datasheets/display/TL050WVH12CT-B1714A_LCD-Module-Spec_Rev4.0.pdf) | Shanghai Top Display Optoelectronics (TDO) | 4.0 / 2024-10-18 | Purchased | Local control-surface display driven by the ESP32-S3 |
