# Guitar-MFX Measurement Record

| Field | Value |
|---|---|
| Document status | Empty measurement ledger; no project result has been recorded yet |
| Revision | 0.5 |
| Date | 2026-10-14 |
| Gate definitions | [02-Architecture.md](02-Architecture.md) §10 and §13 |

## 1. Ownership and recording rules

This document is the single authoritative home for all Guitar-MFX measured results. Targets, vendor specifications, community figures, and design estimates may appear elsewhere when clearly labelled, but a number becomes a **project measurement** only when it is entered here with its conditions.

- Record passes, failures, aborted tests, and retests; do not erase an earlier result when conditions change.
- Use an ISO date (`YYYY-MM-DD`) and identify the person who performed the test.
- Record exact OS image, kernel, PiPedal/plugin version, ESP32 firmware, bridge firmware, and board revision as applicable.
- Describe enough hardware detail to reproduce the setup, including power source, HAT/codec, input/output path, storage, cooler/fan state, cables, loads, and host computer.
- For audio tests, record sample rate, buffer size, period count, and any USB/ASIO buffer in **Buffer settings**. Use `N/A` only when buffering cannot affect the test.
- Put units, method, uncertainty or observed range, and relevant log/artifact paths in **Measured value**.
- Copy the applicable target from the current governing document; do not rewrite a failed result as a relaxed target.
- Use `PASS`, `FAIL`, `INCOMPLETE`, or `NOT RUN` in **Pass/fail**.
- Project documents shall link to a row or section here rather than restating project-measured values.

No tables below contain measured data at revision 0.5.

## 2. Phase verification gates

### 2.1 P0-GATE — Audio feasibility and candidate comparison

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 2.2 P1-GATE — Complete audio engine

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 2.3 P2-GATE — ESP32-S3 control surface

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 2.4 P3-GATE — USB and header-power proof

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 2.5 P4-GATE — Guitar-MFX boards bring-up

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 2.6 P5-GATE — Final assembly and enclosure

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 2.7 P6-GATE — Optional hybrid architecture

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

## 3. Requirement verification gates

### 3.1 V-TONE — Captured-real-amp tone

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 3.2 V-EFFECTS — General effects

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 3.3 V-BEGINNER — Beginner usability

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 3.4 V-TOUCH — Touchscreen

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 3.5 V-FOOT — Foot control

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 3.6 V-OUTPUT — Main and headphone outputs

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 3.7 V-USB — Recording and playback

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 3.8 V-LATENCY — Analog round-trip latency

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 3.9 V-STABILITY — Xruns and soak tests

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 3.10 V-POWER — Power and thermal behavior

Gate definition: [01-Hardware.md](01-Hardware.md) §8.2–8.3. Required rows include: input inrush and rail sequencing; current with the unit off; orderly shutdown time and detection of the Pi 5 3.3 V fall; long-press forced off; power-fail detection delay; relay release versus audio-rail collapse (no audible pop); hold-up time versus the display power-off sequence.

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 3.11 V-STORAGE — Boot and power-loss recovery

Required rows include repeated DC-plug pulls with the read-only root while idle, while saving a preset, and while importing a model, each followed by a boot and a preset-integrity check (UR-PWR-005).

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 3.12 V-BOARD — Board bring-up and regression

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 3.13 V-DISPLAY — Display bring-up and ESP32-S3 display gate

Gate definition: [01-Hardware.md](01-Hardware.md) §6.4. Record the ESP32-S3 module identity (`esptool` chip_id / flash_id output), DCLK frequency and polarity, horizontal/vertical timing, PSRAM and bounce-buffer settings, observed flicker or drift, LVGL benchmark frame rate, touch behaviour, and power-sequence timing as separate rows. Use `N/A` in **Buffer settings** unless audio buffering is involved.

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

## 4. Per-candidate codec and interface measurements

The planned criteria for every candidate are:

1. Stable 24-bit, 48 kHz ALSA operation at the tested small buffers.
2. Analog round-trip latency, including converter filter delay.
3. Dynamic range, idle noise, THD+N, and frequency response.
4. Input full-scale level and gain-calibration range.
5. Clock topology and compatibility with both USB-audio candidates.
6. Pop/mute behavior and driver recovery.
7. Linux ASoC driver behavior and version.
8. PCB, power, clock, and mixed-signal integration implications.

Record separate rows for separate criteria, configurations, buffer settings, or retests.

### 4.1 Pisound v1.2 reference candidate

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 4.2 HiFiBerry DAC2 ADC Pro reference candidate

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 4.3 Production codec/interface candidate A — rev A selection

**Candidate identity:** TI PCM1863 (ADC) + TI PCM5122 (DAC), as on the HiFiBerry DAC2 ADC Pro  
**Datasheet and driver references:** [PCM1863](https://www.ti.com/product/PCM1863), [PCM5122](https://www.ti.com/product/PCM5122); Linux `pcm186x` and `pcm512x` drivers  
**Required rows include:** noise floor with input shorted and with guitar connected but not played, at the normal PGA setting (rev B trigger, [01-Hardware.md](01-Hardware.md) §11.2)

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 4.4 Production codec/interface candidate B — rev B upgrade candidate

**Candidate identity:** Cirrus Logic CS4282P  
**Datasheet and driver references:** [CS4282P datasheet DS1318F1](https://statics.cirrus.com/pubs/proDatasheet/CS4282P_DS1318F1.pdf); Linux `cs530x` driver ([cs530x-i2c.c](https://github.com/torvalds/linux/blob/master/sound/soc/codecs/cs530x-i2c.c))  
**Evaluated only if** the §4.3 noise comparison shows the converter limits the noise floor.

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

## 5. USB architecture comparison measurements

### 5.1 Native Pi 5 UAC2 gadget candidate

Developed first ([02-Architecture.md](02-Architecture.md) §4.4). Required rows include: enumeration with isolated VBUS; four-hour duplex run with three channels to the host and two from it while the NAM A2-Full chain runs, with the xrun count on the local playing path; clock drift and feedback behavior; reconnect after host sleep, cable removal, and MFX reboot; CPU load of the routing service.

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 5.2 Dedicated class-compliant USB audio bridge candidate

Fallback; measured only if §5.1 fails P3-GATE.

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

## 6. Storage candidate measurements

### 6.1 High-endurance microSD

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|

### 6.2 NVMe through Pi 5 PCIe FFC

| Date | OS/kernel/firmware version | Hardware configuration | Buffer settings | Measured value | Target | Pass/fail | Measured by |
|---|---|---|---|---|---|---|---|
