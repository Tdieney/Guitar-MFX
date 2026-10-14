# Guitar-MFX Hardware

| Field | Value |
|---|---|
| Document status | Proposed hardware architecture; production codec, USB path, storage, power implementation, and board layout are measurement-gated |
| Revision | 0.6 |
| Date | 2026-10-14 |
| Related requirements | [00-UserRequirement.md](00-UserRequirement.md) |

## 1. Recommended hardware strategy

Use a Raspberry Pi 5 Model B as both the development and final audio computer. Develop the software and measure candidate audio paths with commercial HATs first, then design the three **Guitar-MFX boards** — an audio board and a main board, connected to the Pi 5 40-pin header by ribbon cable and placed beside the Pi, and a front-panel board behind the display — to replace the commercial HAT in the finished unit. Use an ESP32-S3 (ESP32-S3-WROOM-1-N16R8), soldered on the front-panel board, as a separate control processor for the 5-inch RGB touchscreen and physical controls.

The intended progression is:

1. **Development unit:** Raspberry Pi 5 + HiFiBerry DAC2 ADC Pro (reference for the rev A codec pair, §11.1) + prototype guitar input buffer.
2. **Optional comparison:** Pisound v1.2 as a second reference path; results in [03-Measurements.md](03-Measurements.md).
3. **System validation:** prove neural models, effects, latency, noise, gain calibration, UI communication, power behavior, storage behavior, and both proposed USB-audio architectures.
4. **Final unit:** the same Raspberry Pi 5 SBC plus the custom Guitar-MFX boards using a production codec and supporting circuits selected from comparative measurements.

The Raspberry Pi 5 and the three-board set are user-selected project-scope decisions. They do not validate the production codec, header-powered supply, native USB gadget path, dedicated USB bridge, latency, thermal behavior, or mixed-signal PCB performance. Those remain release gates. The Pi 5 decision was re-examined against Compute Module 5, Rockchip/Allwinner modules, NXP i.MX8M Plus, and an MCU-only design in 2026-10 and retained; see [90-DecisionRecord.md](90-DecisionRecord.md) ADR-005.

## 2. Core processing hardware

| Hardware | Selection | Reason | Status |
|---|---|---|---|
| Development and final audio computer | Raspberry Pi 5 Model B, 4 GB minimum | Runs PiPedal, LV2 plugins, NAM A2-Full, networking, preset storage, and USB-audio services. | Selected by user; audio performance still requires measurement |
| RAM option | Raspberry Pi 5 Model B, 8 GB | Useful if PiPedal and plugins will be compiled locally; extra RAM is not expected to lower real-time audio latency by itself. | Optional |
| Product-specific mixed-signal hardware | Guitar-MFX audio board, main board, and front-panel board (§2.1) | Separates the noise-sensitive analog path from switching power and digital I/O, keeps the ESP32-S3 next to the display, and lets the audio board be built and measured before the main board exists. | Board set selected 2026-10-14; boards not yet designed or validated |
| UI and control computer | ESP32-S3-WROOM-1-N16R8 (16 MB quad flash, 8 MB octal PSRAM, 3.3 V). Development: 44-pin dev board with the same module. Final: module soldered on the front-panel board. | Keeps display, touch, footswitch, and control traffic away from Linux real-time audio. Supervises mute, starts instantly, and shows Linux boot status. | Dev board purchased; display gate in §6.4 must pass before the MCU is final |

Raspberry Pi documents the Pi 5 as a quad-core 2.4 GHz Cortex-A76 platform and recommends a 5 V/5 A supply when full peripheral current is needed. These are **vendor specifications**, not Guitar-MFX measurements. See the [official Raspberry Pi 5 product page](https://www.raspberrypi.com/products/raspberry-pi-5/) and [Raspberry Pi power documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#power-supply).

### 2.1 Guitar-MFX board set

The product-specific hardware is split into three boards, selected by the user on 2026-10-14 ([90-DecisionRecord.md](90-DecisionRecord.md) ADR-013). None is built to the HAT form factor. The audio and main boards connect to the Pi 5 40-pin header by ribbon cable and sit beside the Pi in the enclosure, so the analog section is not placed directly above the Pi 5 switching regulators. The front-panel board sits behind the display.

**Audio board** — noise-critical, 4-layer, expected to be revised more often than the other boards:

- Guitar input jack, protection, 1 MOhm FET-input buffer, software-set PGA, anti-RF filter, single-ended-to-differential drive (§4).
- Audio codec, codec clocking, local decoupling, and local low-noise LDOs fed from a pre-regulated rail supplied by the main board.
- Master and phones volume potentiometers, line driver for Main out L/MONO and R (6.35 mm), headphone amplifier and 3.5 mm headphone jack (§5).
- Two output mute relays placed at the jacks, defaulting to muted when unpowered, with their master-mute gate and coil drivers (§5.5).
- I2S and I2C from the Pi 5 header ribbon, with source-end series resistors on I2S and ground conductors interleaved in the ribbon.
- Test points required by the board bring-up plan in [02-Architecture.md](02-Architecture.md).

**Main board** — power and external I/O:

- DC input, protection, and power tree to the Pi 5 5 V rail, the supply sent to the front-panel board, and the pre-regulated rail sent to the audio board (§8.2).
- Protected external footswitch and expression-pedal jacks, with their signals sent to the front-panel board.
- Routing of the master-mute line from the front-panel board to the audio board, and of the power-state/recovery lines between the Pi 5 and the ESP32-S3.
- Panel USB-C connector routing for the architecture described in §7.

**Front-panel board** — control surface, behind the display:

- ESP32-S3-WROOM-1-N16R8 module soldered directly, so the 20-line RGB bus at 23–27 MHz runs over a short FPC.
- 40-pin display and 10-pin touch FPC connectors, DCLK series damping, and touch I2C pull-ups (§6.1).
- Constant-current backlight boost driver, about 21 V at 40 mA, with PWM dimming.
- Local 3.3 V regulator shared by the ESP32-S3 and display VDD, fed from the main board (§8.2).
- I2C I/O expander, encoder, onboard footswitch connections, and the power button (§6).
- Pi 5 ↔ ESP32-S3 UART, and an ESP32-S3 programming/console port with reset and boot access.

The physical interconnect between the three boards and the Pi 5 header — which cable carries which signals, connector types, and pinouts — is open and is defined with the power, USB, and control-surface design.

**Display bring-up adapter** — a test fixture, not one of the three product boards, and the first PCB of the project (§6.4). It plugs onto the 44-pin ESP32-S3 development board and carries the display and touch FPC connectors, DCLK series damping, the backlight driver, and touch I2C pull-ups.

The Guitar-MFX boards do **not** contain replacements or extensions for Pi 5 HDMI, USB-A, or Ethernet connectors. Those Pi connectors remain physically present; Ethernet is useful during development.

Because the audio board is separate, it can be fabricated, connected to a Pi 5 by ribbon, powered from a bench supply, and measured before the main board exists. The front-panel board can likewise be powered from a bench supply and brought up against a Pi 5 or a PC over UART before the main board exists. Bench phases continue to use commercial audio HATs until the codec and analog architecture are chosen by comparative measurement. Designing the final boards means the project owns the mixed-signal front end, clocking, power integrity, mute behavior, and Linux driver integration. A single measured candidate is therefore insufficient for a PCB commitment; §3.2 and §11 define the required comparison.

The 40-pin header does not expose the Pi 5 USB-C gadget data pair or the PCIe FFC interface. Those connections require separate physical arrangements and are not implied by the header ribbon connection.

## 3. Development audio interfaces

### 3.1 Primary bench path: Pisound v1.2

Use Pisound v1.2 as the first audio HAT. According to **vendor documentation**, it provides:

- Stereo 24-bit conversion at 48, 96, or 192 kHz.
- 110 dB specified input/output SNR at 48 kHz.
- Adjustable analog input gain and output volume.
- ALSA support in the Raspberry Pi kernel.
- 6.35 mm stereo input and output connectors.
- DIN MIDI input and output.
- Pi 5 and Active Cooler compatibility on hardware revision 1.2.

The project will evaluate it at 48 kHz. The vendor specifies 100 kOhm input impedance, so a passive guitar shall not be connected directly for final tone evaluation; the 1 MOhm guitar front end in §4 is required. Vendor specifications are shortlisting inputs only. Actual results belong in [03-Measurements.md](03-Measurements.md). See the [Pisound audio specifications](https://blokas.io/pisound/docs/audio/) and [general specifications](https://blokas.io/pisound/docs/general-specifications/).

### 3.2 Required comparison and optional reference hardware

| Hardware | Purpose | Purchase rule |
|---|---|---|
| HiFiBerry DAC2 ADC Pro | Required second I2S-HAT path for comparing latency, driver stability, noise, frequency response, converter delay, clock behavior, and recovery against Pisound. | Required in Phase 0 because a single measured audio candidate is not enough to select a custom-board codec architecture. |
| Professional USB interface such as MOTU M2 or Focusrite Scarlett 2i2 4th Gen | Independent loopback measurement, calibrated playback/recording, re-amp tests, and recording the device under test. | Strongly recommended measurement equipment; one suitable unit is sufficient. |
| Low-cost deterministic MCU audio board such as Daisy Seed/Pod | Optional perceptual reference for comparing an extremely short deterministic path with the measured Linux path before judging whether the hybrid phase is justified. | Optional de-risking purchase; it is not a production-platform decision or a Phase 0 pass requirement. |

PiPedal's developer guidance says audio-interface dynamic range and input calibration materially affect NAM results. This is **project/community guidance**, not a Guitar-MFX measurement. See [PiPedal's audio-interface guidance](https://rerdavies.github.io/pipedal/FAQ.html).

Pisound and HiFiBerry are bench candidates, not automatic production-codec selections. Their measured results, driver behavior, and circuit constraints inform the production shortlist; at least two production-relevant codec or interface configurations must pass through §11 before the board is frozen.

## 4. Guitar input hardware

### 4.1 Required characteristics

The prototype and Guitar-MFX audio board require a custom mono instrument-input stage with:

- 6.35 mm TS input jack with a switch contact that grounds the input when no plug is inserted.
- At least 1 MOhm input impedance.
- Low input-bias current and low voltage/current noise.
- RF filtering that does not audibly load passive pickups.
- ESD and accidental over-voltage protection with low-capacitance protection parts.
- AC coupling and a controlled bias point where a single-supply circuit is used.
- Sufficient headroom for hot passive humbuckers and active guitars.
- Software-set gain of **0 to +30 dB** (PGA inside the codec or a discrete PGA). 0 dB keeps active pickups and hard playing (up to about 2 V peak) from clipping; +30 dB lifts weak single-coil signals (tens of millivolts) above the converter noise floor. These source levels are design estimates, not measurements.
- Differential drive into the ADC where the selected codec accepts it.
- A hardware or reliably derived clip indication.

An OPA1652 or OPA1642-class FET-input dual op-amp is a prototype candidate, not a selected production part. The exact op-amp, protection network, PGA, coupling capacitors, and anti-alias network shall be selected during schematic design and verified with the actual source, codec, PCB, and power rails.

Layout rule: the node between the input jack and the buffer is the highest-impedance, most noise-sensitive node in the unit. Keep it a few millimetres long, avoid vias where possible, and consider a guard ring.

### 4.2 Gain calibration

Neural amp captures are sensitive to the level presented to the model. The hardware therefore needs a repeatable relationship between jack voltage, ADC full scale, and the software reference.

The design shall support:

1. Applying a known RMS test signal to the input.
2. Setting or measuring analog gain without clipping.
3. Recording the resulting digital level.
4. Saving the calibration offset with the audio-device profile.
5. Confirming the setting using passive, hot passive, and active-pickup test cases.

An uncalibrated gain knob adjusted only by ear is not sufficient for final model evaluation. Results are recorded in [03-Measurements.md](03-Measurements.md).

The analog PGA setting is a **global calibration for the instrument**, set once for the user's guitar. It is not stored per preset. The level presented to each NAM model is adjusted digitally inside PiPedal and stored with the preset. Preset changes therefore never step the analog PGA, avoiding switching clicks while keeping the model drive level repeatable.

## 5. Audio outputs

The output section on the audio board is:

```text
DAC ─ LPF ─┬─ Master pot ─ Line driver ─[Relay 1]─┬─○ Main out L/MONO (6.35 mm)
           │                                       └─○ Main out R      (6.35 mm)
           └─ Phones pot ─ HP amp ─[Relay 2]─○ Headphones (3.5 mm)
Master pot and Phones pot: analog, independent (§5.6)
Relay 1 and Relay 2: muted when unpowered (§5.5)
```

The layout follows common commercial multi-effects units: two main outputs plus a headphone output, with no dedicated guitar-amplifier output. A **2-channel DAC** is sufficient.

### 5.1 Main outputs

Provide two unbalanced 6.35 mm TS jacks, **Main out L/MONO** and **Main out R**, at line level (UR-OUT-001). They serve powered speakers, studio monitors, mixers, and guitar amplifiers. Powered PC speakers with a 3.5 mm or RCA input connect through an adapter cable. This is not a power-amplifier output and shall not drive a passive loudspeaker.

Mono use, the output level setting, and guitar-amplifier use are in §5.3 and §5.4.

The output stage shall provide:

- Defined maximum output level and output impedance.
- AC coupling or DC-servo behavior appropriate to the selected codec.
- Short-circuit tolerance suitable for normal cable insertion.
- Pop-free mute during boot, shutdown, audio restart, and fault recovery.

### 5.2 Headphone output

Provide a separate 3.5 mm stereo headphone output (user choice; use a panel-mounted metal jack so cable strain does not load the PCB) with a dedicated driver and its own analog volume potentiometer ahead of the driver. The potentiometer is the hardware limit for the output closest to the user's ears and is independent of the master potentiometer (§5.6). The driver is sized for headphones up to 300 Ω until the user's headphone model is known. Commercial-HAT output may be used during early development, but the audio board shall be measured with the intended headphone impedances.

OPA1622 and similar low-distortion headphone drivers are candidates, not selected parts. The final choice shall follow output-power, voltage swing at 300 Ω, stability, thermal, noise, package, supply-rail, and load measurements rather than headline SNR alone.

### 5.3 Cabinet setting for guitar amplifiers

There are no output modes. When the main outputs feed a guitar amplifier, the user switches the cabinet IR off in the settings, because the amplifier's real speaker cabinet replaces it; when feeding the amplifier's front input, the user also bypasses the amp model block. Speakers, monitors, and mixers take the main outputs with the cabinet IR on. The previous three-mode design (Full-range / Amp FX return / Amp input) was withdrawn on 2026-10-14.

### 5.4 Output level and mono

- **Output level (Line / Instrument):** a global software setting for the main outputs, as on the [Line 6 HX Stomp](https://line6.com/data/6/0a020a4010c935bb66a4c0c44f/application/pdf/HX%20Stomp%20Manual%20-%20English%20.pdf). Instrument level lets the main outputs feed an amplifier's front input without overdriving it. For the lowest hiss into an amplifier, lower the analog Master potentiometer rather than relying only on digital attenuation; add a switched passive pad later only if measurement shows audible hiss.
- **Mono:** when only L/MONO is connected (for example into a guitar amplifier), it carries a mono sum. Initially a UI setting; automatic detection through the R jack's switch contact is optional.

### 5.5 Mute

- **Relay 1** mutes both main outputs. **Relay 2** mutes the headphone output.
- Both relays sit **at the jacks, after the output drivers**, because the drivers and headphone amplifier themselves produce transients when powered up or down; a relay ahead of them would not stop those transients.
- Both relays are **muted when unpowered**.
- The ESP32-S3 drives one **master-mute GPIO directly** (not through the I/O expander). A logic gate on the audio board, beside the relay coil drivers, combines it with per-output enables from the I/O expander, so the safety path does not depend on the expander. The ESP32-S3 releases mute only after the Pi reports stable audio, and re-mutes on lost heartbeat.
- The gate inputs are pulled to the muted state on the audio board, so an unplugged cable or an unpowered front-panel board leaves the outputs muted (proposed; verified at board bring-up).
- Headphone insertion is sensed through the I/O expander; the ESP32-S3 then opens Relay 1 (UR-OUT-008).

### 5.6 Master volume

Two independent analog potentiometers on the audio board (UR-OUT-002, UR-OUT-003):

- **Master** — in the signal path ahead of the line driver; sets both main outputs.
- **Phones** — in the signal path ahead of the headphone amplifier.

Both keep working when software fails. They are kept separate because inserting headphones mutes the main outputs (UR-OUT-008); with a single shared knob, headphones would start at whatever level the speakers were using.

Design notes:

- Use dual-gang, logarithmic (audio-taper) potentiometers. Channel tracking is worst near the bottom of the rotation, so choose parts with good tracking and set stage gains so normal listening sits mid-rotation.
- AC-couple both potentiometers so no DC flows through the track; DC through a pot makes it crackle when turned.
- The codec does not need its own output-volume control. The digital output level of the audio graph still ramps from silence at `AUDIO_STARTING` ([02-Architecture.md](02-Architecture.md) §6).

The hardware limits when software fails are the relays (§5.5), the two potentiometers, and the volume controls on the speakers and guitar amplifier.

### 5.7 Mute versus true bypass, and ground loops

The baseline safety design remains default mute rather than true bypass: an unprocessed pickup signal sent to headphones or full-range speakers would be quiet and would not contain amp/cab simulation. There is no dedicated amplifier output, so there is no true-bypass path; all outputs default to silent.

Ground-loop handling: connect the chassis and jack sleeves to ground at **one defined point**, and reserve layout space for a ground-lift switch on the main outputs, a USB isolator, or a line-output isolation transformer. Fit them only if the hum test shows a need. Hum is checked by adding the powered speakers, the computer, and the amplifier one at a time (§13 of [02-Architecture.md](02-Architecture.md), V-OUTPUT).

## 6. Display, touch, and physical controls

| Item | Hardware | Notes |
|---|---|---|
| Control MCU | ESP32-S3-WROOM-1-N16R8 | 3.3 V variant only; avoid the 1.8 V `…V` variants, whose GPIO47/48 run at 1.8 V and cannot drive the display. Fallback if the display gate fails: STM32U5G9 ([90-DecisionRecord.md](90-DecisionRecord.md) ADR-015). |
| Display | TDO TL050WVH12CT-B1714A, 5.0-inch 800×480 IPS, parallel RGB | Purchased. Summary in §6.1; datasheet in [references](references/datasheets/display/TL050WVH12CT-B1714A_LCD-Module-Spec_Rev4.0.pdf). |
| Touch | GT911 on the display's touch FPC | I2C; shares the bus with the I/O expander. |
| I/O expander | I2C GPIO expander with interrupt output | Carries slow, non-safety signals (§6.2). Part not selected. |
| Encoder | One detented rotary encoder with push switch | Navigation and parameter editing (UR-UI-007). A/B on direct GPIO, push via expander. |
| Master volume | Dual-gang log analog potentiometer on the audio board | Both main outputs; works without software (§5.6). |
| Phones volume | Dual-gang log analog potentiometer on the audio board | Headphone output; independent of master (§5.2, §5.6). |
| Onboard footswitches | Three soft-touch momentary switches | Proposed, not yet a locked user requirement. |
| External switch input | One protected TRS input supporting a dual momentary footswitch | Required connector; exact wiring and normally-open/closed detection remain to be specified. |
| Expression input | One protected TRS input with calibration and plug detection | Proposed for the first board; confirm before layout. Needs an ADC1 pin (GPIO1–10). |
| Power control | Momentary illuminated button | Requests orderly shutdown; not a direct unsupervised cut of Pi 5 power. |
| Service controls | ESP32-S3 reset/boot access and Pi 5 recovery/test access | Required during development; may be internal in the enclosure. |

Footswitch and expression connectors need series protection, ESD protection, debouncing, and controlled ground-current paths. They shall not share sensitive analog return paths with the guitar input.

The 5.0-inch 800×480 panel has about 187 pixels per inch (0.135 mm per pixel). Text and touch targets must therefore be larger in pixels than on a 7-inch panel of the same resolution; a 20 px font is only about 2.7 mm tall, and a 9 mm touch target is about 67 px. Final sizes are a usability decision tested under V-BEGINNER and V-TOUCH.

### 6.1 Display electrical summary

The following are **vendor specifications** from the TDO final specification rev 4.0 (2024-10-18), not Guitar-MFX measurements.

| Item | Value |
|---|---|
| Panel | 5.0-inch a-Si TFT, IPS, normally black, 800×480, 16.7M colours, viewing angle 80° in all directions, contrast 1000–1200, 550 cd/m² |
| Driver / touch IC | ST7265 / GT911 |
| Interface | 24-bit parallel RGB TTL; SYNC or SYNC-DE mode (no DE-only mode). RGB565 is used by grounding the unused low bits. |
| Supply | Single VDD 3.1–3.6 V (3.3 V typical), about 70 mA. No external bias rails are required. |
| DCLK | **23 min / 25 typ / 27 max MHz**; Th 808/816/848 DCLK, Tv 496/512/528 lines (≈60 Hz at 25 MHz; ≈51 Hz at 23 MHz with maximum porches). In SYNC mode the datasheet fixes Thbp=43, Thfp=8, Thw=4, Tvbp=12, Tvfp=8, Tvw=4. |
| DCLK polarity | The datasheet shows both polarities (DCLPOL), but the FPC has no select pin; the fixed polarity must be found at bring-up. |
| Backlight | 12 LEDs, 6 series × 2 parallel; IF 40 mA typical, 50 mA absolute maximum; VF 18 V typical, 20.4 V maximum; 0.72–1.02 W. Requires a constant-current boost driver with PWM dimming, set to 40 mA. |
| Power-on | VDD → ≥1 ms (internal reset) → ≥10 ms → DISP high → ~85 ms + ≥33 ms → RGB signals → ≥100 ms → backlight on |
| Power-off | Backlight off → ≥1 ms → DISP low → keep DCLK running ≥100 ms while the IC discharges → stop signals → remove VDD |
| Rule | Do not drive input signals while VDD is off. Supply the display VDD from the same 3.3 V rail as the ESP32-S3. |
| Connectors | Display: 40-pin, 0.5 mm FPC (Omron XF2M-4015-1A or equivalent). Touch: 10-pin, 0.5 mm FPC (XF2M-1015-1A or equivalent); touch VDD 2.8–3.3 V; I2C pull-ups 4.7–10 kΩ. |
| Mechanical | Outline 120.70 × 75.80 × 4.63 mm; cover lens 120.40 × 74.82 mm, 0.7 mm tempered glass; lens viewing area 108.80 × 65.60 mm; active area 108.00 × 64.80 mm; air bonding. |

The display pin order resembles the 40-pin RGB layout used by several ESP32-S3 display boards, but every pin and the backlight circuit must be checked against the datasheet before connecting the panel to any third-party board.

### 6.2 ESP32-S3 GPIO budget

On the WROOM-1-N16R8 module, GPIO35–37 are used internally by the octal PSRAM. GPIO43/44 carry the UART console. Programming through the USB-UART port frees GPIO19/20, leaving **31 usable GPIOs**.

| Function | Pins | Direct GPIO |
|---|---:|---|
| RGB display: 16 data + DCLK + HSYNC + VSYNC + DE | 20 | Required |
| I2C (GT911 touch and I/O expander) | 2 | Required |
| Shared interrupt from I/O expander and touch | 1 | Recommended |
| UART to Pi 5 | 2 | Required |
| Master mute (safety) | 1 | Required |
| Backlight PWM | 1 | Required |
| Expression pedal (ADC1) | 1 | Required |
| Encoder A/B | 2 | Recommended |
| **Total** | **30** | |

Through the I/O expander: display DISP, touch reset, encoder push, footswitches, headphone-insert sense, per-output mute enables, and indicator LEDs.

If more pins are needed: try SYNC mode without DE (−1, behaviour of the unused DE pin must be tested), or move the encoder to the expander (−2). Strapping pins GPIO0, 3, 45, and 46 may carry display data because the display inputs do not pull them at reset; **GPIO45 must never be pulled high at boot**, as that selects 1.8 V flash and prevents booting.

### 6.3 Framebuffer

Two 800×480 RGB565 frames (768 000 bytes each, about 1.5 MB total) are kept in octal PSRAM, with the ESP-IDF RGB driver's bounce-buffer mode enabled. On the ESP32-S3 the PSRAM shares a bus with the flash; Espressif lists Wi-Fi activity and flash writes as causes of display drift. The architecture avoids both on the ESP32-S3: it does not use Wi-Fi, and it does not write flash during normal play (presets live on the Pi).

### 6.4 Display bring-up adapter and gate

Espressif's reference for 800×480 on ESP32-S3 runs at 21 MHz PCLK (41 Hz refresh, about 26 fps LVGL benchmark on one core) with a stated RGB-interface ceiling near 30 MHz. These are **vendor figures**. This panel's minimum DCLK is 23 MHz, so the ESP32-S3 choice is conditional on a bench test.

The first PCB of the project is a 2-layer adapter that plugs onto the 44-pin development board, with a solid ground plane on the bottom layer and:

- 40-pin and 10-pin FPC connectors with short RGB traces;
- 22–33 Ω series resistor on DCLK;
- constant-current boost backlight driver (about 21 V, 40 mA) with PWM;
- touch I2C pull-ups.

Jumper wires are not acceptable for this test: a 20-line bus at 23–25 MHz over loose wires produces ringing and crosstalk that would be mistaken for an MCU limit.

**Display gate:** stable image with no visible flicker or drift at DCLK ≥ 23 MHz, with both DCLK polarities tried, double framebuffer in PSRAM, and bounce buffer enabled; LVGL benchmark frame rate recorded; GT911 touch working; the power-on and power-off sequences in §6.1 implemented. Before testing, confirm the development board's module with `esptool.py --chip esp32s3 chip_id` and `flash_id` (expected: 8 MB PSRAM at 3.3 V and 16 MB flash). Results go in [03-Measurements.md](03-Measurements.md) §3.13. If the gate fails, evaluate the STM32U5G9 fallback. The display runs RGB565.

## 7. USB-C hardware and architecture candidates

### 7.1 Pi 5 constraint

Raspberry Pi's **vendor documentation** states that Pi 5 gadget mode uses the native SoC USB OTG peripheral exposed through the Pi 5 USB-C power connector; the PCIe-attached RP1 controller serving the main USB ports does not support OTG. On Pi 5 Model B, the USB-C connector carries USB 2.0 data only; USB 3.x SuperSpeed pins are not wired on that connector. The gadget data pair is not available on the 40-pin header, so the Pi 5's own USB-C receptacle must participate in the native-gadget path. See the [Raspberry Pi OTG whitepaper](https://pip-assets.raspberrypi.com/categories/685/app-notes-guides-whitepapers/documents/RP-009276-WP-1-Using%20OTG%20mode%20on%20Raspberry%20Pi%20SBCs).

For Raspberry Pi 5 Model B, the proposed gadget configuration places the following under `[all]` in `/boot/firmware/config.txt`, not under `[cm5]`:

```ini
[all]
dtoverlay=dwc2,dr_mode=peripheral
```

This configuration is a documented platform mechanism; Guitar-MFX full-duplex UAC2, clock recovery, Windows/Linux host behavior, and the proposed power/data interconnect are **not validated**.

### 7.2 Proposed native-gadget panel connection — bench proof required

The proposed, unproven mechanical/electrical path is:

- A panel-mounted USB-C receptacle on the Guitar-MFX main board connects to the Pi 5 USB-C receptacle through a short internal cable or interposer.
- The path carries USB 2.0 D+/D− and the required configuration-channel signals.
- Host VBUS is isolated from the Pi 5 5 V rail so neither side can back-power the other, satisfying UR-PWR-002 only if measurements confirm it.
- Main power enters the main board from its external DC input and reaches Pi 5 through the 40-pin header 5 V/GND pins.

This approach is **Proposed, bench proof required**. It shall not be included in PCB layout until a hand-made cable or adapter proves all of the following:

- The host enumerates the Pi 5 consistently in device mode with the intended CC arrangement.
- Isolating VBUS does not prevent role detection or cause the Pi 5 to leave device mode.
- VBUS-presence sensing, disconnect, reconnect, host sleep, cold boot, and warm reboot behavior are defined.
- No measurable back-feed reaches the Pi 5 rail or host VBUS.
- A legacy-OTG-style USB-C cable/adapter is evaluated if ordinary USB-C role detection cannot be made reliable.

The Pi 5 is powered through its header during this experiment. Raspberry Pi's OTG whitepaper describes header power as the way to free the USB-C port for gadget use, while Raspberry Pi's **vendor power documentation** notes that header power bypasses USB-PD negotiation. The Guitar-MFX design must therefore provide and measure inrush limiting, rail sequencing, input protection, bulk capacitance, current capability, and brownout behavior rather than assuming that the Pi 5 USB-C power path still protects the system. See the [Raspberry Pi USB Power Delivery whitepaper](https://pip-assets.raspberrypi.com/categories/685/app-notes-guides-whitepapers/documents/RP-009856-WP-1-USB%20Power%20delivery%20on%20Raspberry%20Pi%205.pdf).

### 7.3 Co-primary USB audio candidates

Do not select a winner before Phase 3. Evaluate these as co-primary architectures:

| Criterion | Native Pi 5 UAC2 gadget | Dedicated class-compliant USB audio bridge |
|---|---|---|
| Bill-of-materials cost | Measure cable/interposer, protection, CC/VBUS circuitry, and any clock/rate-matching cost. | Measure bridge controller, clocks, memory/firmware, protection, and extra digital-audio circuitry. |
| Added latency | Measure complete analog and USB paths; do not infer from buffer sizes. | Measure bridge buffering plus the bridge-to-Pi digital link. |
| Pi 5 digital interfaces | Determine coexistence with the selected codec and control buses. | Prove that an available I2S/TDM, USB-host, or other supported link remains available alongside the codec. |
| Host-driver behavior | Test class-compliant enumeration, duplex streaming, sleep/reconnect, and the intended Windows DAW. | Perform the same host matrix and document any firmware/driver dependency. |
| Development risk | Includes CC/VBUS role behavior, gadget configuration, clock-domain handling, and Linux routing. | Includes controller firmware/tooling, digital-link integration, added clock domains, availability, and board complexity. |

All comparison results belong in [03-Measurements.md](03-Measurements.md). Phase 3 selects the architecture only after the documented tests; until then, both remain proposed.

### 7.4 Evidence gathered since revision 0.2

- **Native gadget:** a community lab write-up documents a Pi 5 enumerating as a UAC2 sound card to a host computer ([RASPIAUDIO CamillaDSP notes](https://github.com/RASPIAUDIO/CamillaDSP/blob/main/docs/usb_audio_gadget_pi5.md)). This is **community evidence** that the software path exists on Pi 5; the panel/power arrangement in §7.2 is still unproven.
- **Digital interfaces for a bridge (corrected 2026-10-14):** according to Raspberry Pi's **vendor I2S whitepaper**, Pi 5 has two I2S instances, `I2S0` (Pi is clock producer) and `I2S1` (Pi is clock consumer), each with up to four bidirectional data lanes, but **they occupy the same GPIO pins (18–27) and Pi 5 cannot act as both at the same time** (whitepaper p. 8). There is therefore **one usable I2S bus**, with one BCLK/LRCK pair. An earlier revision of this document wrongly stated that a bridge could use a second, separate I2S bus. See [Using the I2S peripherals on Raspberry Pi SBCs](https://pip-assets.raspberrypi.com/categories/1259-audio-camera-and-display/documents/RP-009699-WP-1-Using%20the%20I2S%20peripherals%20on%20Raspberry%20Pi%20SBCs.pdf).
- **Consequence:** with the codec as clock master (Pi on `I2S1`), a dedicated USB bridge would have to run as a clock consumer on the same BCLK/LRCK and use an extra data lane (GPIO22/23), i.e. a single audio clock domain with asynchronous rate handling on the USB side; otherwise it needs a different link. Whether a given bridge chip supports this is unverified and must be checked before the audio-board schematic is frozen.

## 8. Power hardware

### 8.1 Development power

- Use the official Raspberry Pi 27 W supply for normal Pi 5 bench development.
- Use the Raspberry Pi 5 Active Cooler during performance and thermal validation.
- For the USB VBUS-isolation experiment, use a current-limited, measured 5.1 V bench supply into the appropriate 5 V/GND header connection and follow the Phase 3 test plan.
- Keep fan wiring and switching regulators away from the guitar input and measurement cables.

### 8.2 Proposed Guitar-MFX main-board power tree

The proposed main input remains a regulated 12 V DC adapter rated around 3 A. Convert it to the Pi rail using an efficient synchronous buck regulator designed for 5.1 V at up to 5 A, with its final rating, transient response, cabling loss, and thermal margin established by measurement.

Do **not** use a linear regulator to drop 12 V to the Pi 5 rail. At multi-ampere load it would dissipate excessive heat. Low-noise LDOs may be used after switching conversion for controlled low-current codec, reference, or analog rails.

Because the board powers Pi 5 through the 40-pin header, it bypasses USB-PD negotiation and does not inherit an assumed USB-C inrush/protection sequence. The board shall provide:

- Input fuse or resettable protection.
- Reverse-polarity and surge protection appropriate to the selected adapter.
- Inrush limiting and a defined startup/shutdown sequence.
- Bulk and local capacitance based on measured transient load.
- Separate filtered rails for digital, display/backlight, codec, reference, and sensitive analog loads.
- A supply sent to the front-panel board, where a local regulator makes the 3.3 V rail shared by the ESP32-S3 and the display VDD (display about 70 mA typical, vendor figure), so the display is never driven while unpowered.
- Capacity on that supply for the display backlight: a constant-current boost driver on the front-panel board, about 21 V compliance at 40 mA, PWM dimming (about 1 W).
- A pre-regulated rail sent to the audio board, where local low-noise LDOs make the final codec and analog rails (two-stage regulation).
- A supervised power latch or load switch.
- Enough hold-up time for the ESP32-S3 to mute audio and request orderly Linux shutdown, or another measured power-loss strategy.
- Brownout detection, logging, and deterministic recovery.
- Verified conductor, connector, header-pin, and copper current capacity.

No power implementation is validated until the board bring-up measurements are recorded in [03-Measurements.md](03-Measurements.md).

## 9. Cooling, storage, and enclosure

### 9.1 Cooling and board placement

Use the Active Cooler during Pi 5 development so thermal throttling does not invalidate benchmarks. The final enclosure places the audio and main boards beside the Pi 5, connected by ribbon from the 40-pin header, and the front-panel board behind the display, without blocking cooler airflow, fan connector access, test points, or service removal. Keep the ribbon short.

The layout shall place the high-impedance guitar input and low-level analog circuitry away from the cooler motor, PWM wiring, Pi 5 switching nodes, and the main-board power converters. On the audio board, use **one continuous ground plane** and separate circuits by placement — input at one end, codec in the middle, outputs at the other end, the ribbon connector at the edge farthest from the input buffer — rather than splitting the plane. If the Active Cooler remains in the final unit, measure both acoustic and electrical fan noise at all fan states. A passive enclosure-coupled heat spreader may be evaluated, but it is not selected until an enclosed thermal test proves no sustained throttling.

### 9.2 Pi 5 connectors and boot storage

The Pi 5 HDMI, USB-A, and Ethernet connectors remain physically present but are not product I/O. Ethernet is useful for development, recovery, and profiling. The enclosure must provide clearance even when those ports are not externally accessible.

Raspberry Pi has no onboard eMMC on Pi 5 Model B. A high-endurance 32–64 GB microSD card is the Phase 0 boot medium and the default final medium if no later storage decision is made. NVMe boot through the Pi 5 PCIe FFC connector is an open option because it may improve robustness or boot behavior, but it must be compared by measured boot time, write reliability, power-loss recovery, power consumption, and mechanical complexity.

Raspberry Pi's **vendor documentation** supports NVMe boot through the Pi 5 PCIe FFC interface. Routing PCIe on a Guitar-MFX board or adding an FFC/M.2 assembly would substantially increase layout, signal-integrity, power, cooling, and mechanical difficulty. It shall not be added without a documented reason and measured benefit. See the [Raspberry Pi NVMe documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#nvme-ssd-boot).

### 9.3 Mechanical requirements

The enclosure shall support the guitar input, main outputs, headphone, USB-C, DC, footswitch, and expression connectors mechanically. Board-mounted jacks shall not carry enclosure loads without adequate support. The USB-C internal cable/interposer, cooler, microSD or NVMe service path, header ribbons, the cable to the front-panel board, display window and FPC route (§6.1), and board removal sequence must be represented in the mechanical model before PCB layout is frozen.

## 10. Phase 0 purchase list

### Required

- Raspberry Pi 5 with at least 4 GB RAM.
- Official Raspberry Pi 27 W USB-C power supply.
- Raspberry Pi 5 Active Cooler.
- High-endurance 32–64 GB microSD card for development.
- HiFiBerry DAC2 ADC Pro — required; it carries the selected rev A codec pair (PCM1863 + PCM5122, §11.1).
- Pisound hardware revision 1.2 — optional since 2026-10-14, as a second reference path.
- Components and prototyping board for the 1 MOhm guitar input buffer.
- Shielded guitar and loopback cables plus required 6.35/3.5 mm adapters.
- Momentary switches and one rotary encoder for control prototyping.
- UART wiring or a level-compatible board-to-board connection between Pi 5 and ESP32-S3.
- Display bring-up adapter PCB and parts (§6.4): 40-pin and 10-pin 0.5 mm FPC connectors, backlight boost driver, passives.

The second HAT is required because the final product owns the codec and mixed-signal PCB. Choosing from a single measured candidate would not reveal interface-specific latency, converter-filter delay, driver, clock, or analog limitations. Per-candidate results shall be recorded in [03-Measurements.md](03-Measurements.md).

Already purchased: the 44-pin ESP32-S3-WROOM-1-N16R8 development board and the TDO TL050WVH12CT-B1714A display.

### Strongly recommended test equipment

- One professional USB audio interface suitable for reference measurements and re-amping.
- Oscilloscope with appropriate probes.
- True-RMS multimeter.
- Logic analyzer for UART, SPI, I2C, USB-state, and boot diagnostics.
- Known resistive loads for line and headphone testing.
- Current-limited bench supply for header-power and board bring-up tests.

### Do not purchase or fabricate yet

- Production quantities of the final codec, headphone amplifier, USB bridge, or power components.
- Audio-board fabrication before Phase 0 audio measurements are complete, and main-board fabrication before the Phase 3 USB/power-path proof is complete. The display bring-up adapter is exempt.
- NVMe hardware or PCIe layout solely on assumption; first define the storage comparison.
- Daisy hardware unless the optional perceptual-latency reference is considered useful.

## 11. Final codec selection gate

The Guitar-MFX audio-board codec shall not be selected until comparative measurements exist. A minimum of two production-relevant codec or interface configurations must be assessed; Pisound and HiFiBerry bench measurements provide required reference points but do not automatically qualify their circuits or codec ICs for direct reuse.

Each candidate must be evaluated for:

- Stable 24-bit, 48 kHz ALSA operation at the required small buffers.
- Measured analog round-trip latency, including converter filter group delay.
- Measured dynamic range, idle noise, THD+N, and frequency response in the actual evaluation assembly.
- Input full-scale level and gain-calibration range.
- Clock topology and compatibility with both USB-audio candidates.
- Pop/mute behavior and driver recovery.
- Linux ASoC driver maturity and expected component availability.
- PCB layout complexity and separation of analog and digital supplies/grounds.
- At least 2 ADC channels with differential inputs and 2 DAC channels (§5).
- A 0 to +30 dB input PGA, either inside the codec or as a separate part (§4.1).
- Ability to act as I2S clock master from its own crystal. Pi 5 has one usable I2S bus at a time, so any USB bridge must share it (§7.4).

All test conditions and results shall be entered in the per-candidate tables in [03-Measurements.md](03-Measurements.md). The component datasheet and official driver documentation shortlist parts; comparative prototype measurements make the board decision.

### 11.1 Selected for audio-board rev A (2026-10-14)

**TI PCM1863 (ADC) + TI PCM5122 (DAC)** — the same pair as the HiFiBerry DAC2 ADC Pro, so the converter path can be measured on that HAT before the audio board is drawn. Vendor specifications, not Guitar-MFX results:

| Item | PCM1863 (ADC) | PCM5122 (DAC) |
|---|---|---|
| Dynamic range | 110 dB differential input, 106 dB single-ended | 112 dB |
| Gain / level | PGA −12 to +32 dB, I2C-controlled (covers the 0 to +30 dB need, §4.1) | 2 V RMS line output |
| Full scale | 2.1 V RMS single-ended, 4.2 V RMS differential | 2 V RMS |
| Filter latency at 48 kHz | 30/fs normal, **10/fs low-latency** | 20/fs normal, **3.5/fs low-latency** |
| Package | TSSOP-30 | TSSOP-28 |
| Supply | single 3.3 V | — |
| Linux | `pcm186x` driver | `pcm512x` driver; HiFiBerry overlay for the pair |

- Use the low-latency filters: about 13.5/fs ≈ 0.28 ms for the converters.
- Clocking follows the HiFiBerry design with the codec as clock master and Pi 5 as clock consumer (`I2S1`). The project runs only at 48 kHz, so one 49.152 MHz oscillator replaces the two-oscillator arrangement. Codec-master operation on Pi 5 is checked in Phase 0.
- Status: selected for rev A; validation remains measurement-gated (§11 criteria, [03-Measurements.md](03-Measurements.md) §4.3).

### 11.2 Rev B upgrade candidate: Cirrus Logic CS4282P

Vendor specifications: ADC 123 dB and DAC 128 dB dynamic range (A-weighted, measured with the external buffer's contribution deducted), THD+N −110 / −115 dB, current-output DAC, no internal PGA, 48-pin QFN with 0.4 mm pitch, mainline Linux driver `cs530x`. Adopting it requires an external digitally controlled PGA, a differential input driver, and a current-to-voltage output stage per channel. See [90-DecisionRecord.md](90-DecisionRecord.md) ADR-018.

**Upgrade trigger (Phase 0):** with the HiFiBerry DAC2 ADC Pro at the normal PGA setting, measure the noise floor (a) with the input shorted and (b) with the guitar connected but not played. If (b) is clearly higher than (a), for example by more than 10 dB, the guitar is the dominant noise source and a better ADC would not be audible. If (b) is close to (a), the converter limits the noise floor and CS4282P is evaluated for rev B.

## 12. Hardware not selected as the primary platform

| Platform | Reason it is not primary |
|---|---|
| ESP32-S3 alone | Excellent UI/control device, but insufficient demonstrated margin and audio ecosystem for the desired full neural models, large effects library, display, and bidirectional USB audio at once. |
| Daisy Seed alone | Deterministic low latency and intended for embedded audio; A2-Lite support is relevant, but it offers less effects and operating-system headroom than the selected Pi 5 architecture. It remains an optional reference or future hybrid path. |
| Teensy 4.1 alone | Strong MCU performance and USB-audio support, but its standard ecosystem has less ready-made support for the desired Linux plugin/model workflow. |
| x86 mini PC | Ample DSP performance, but typical mini-PC ports cannot operate as the required class-compliant USB audio device without extra hardware, and the form factor is less suitable for the custom 40-pin board objective. |
| Raspberry Pi Compute Module 5 on a custom carrier | Same SoC as Pi 5, with USB 2.0 and an OTG role pin on the module connector, eMMC, and a PMIC enable pin. Rejected for now because the carrier needs a 5 V/5 A power tree and fine-pitch connectors, passive cooling is weaker than Pi 5, and OTG on CM5 is not officially documented. |
| Rockchip/Allwinner modules (Radxa, Orange Pi, Banana Pi) | Similar price to CM5 when compared module to module; risk that required peripherals depend on 5.10/6.1 vendor kernels while mainline PREEMPT_RT arrived in Linux 6.12; their NPUs do not help per-sample NAM inference; weaker carrier reference material. |
| NXP i.MX8M Plus module + Yocto | Strong industrial embedded-Linux platform with an on-chip Cortex-M7 and Elk Audio OS support, but the highest software and hardware effort and little hobby-community support for a first audio project. |
| MCU-only (STM32H7 / Daisy class) with NAM A2-Lite | Now credible for tone (vendor A2-Lite claims in [90-DecisionRecord.md](90-DecisionRecord.md) §2), but every effect would have to be written by hand and bare-metal real-time audio is a poor fit for AI-assisted code. A candidate for a later revision. |

These comparisons explain scope; they are not measurements of the Guitar-MFX hardware. Rationale and sources are in [90-DecisionRecord.md](90-DecisionRecord.md) ADR-005 and ADR-015.

## 13. References

- [Raspberry Pi 5 product page](https://www.raspberrypi.com/products/raspberry-pi-5/)
- [Raspberry Pi OTG whitepaper](https://pip-assets.raspberrypi.com/categories/685/app-notes-guides-whitepapers/documents/RP-009276-WP-1-Using%20OTG%20mode%20on%20Raspberry%20Pi%20SBCs)
- [Raspberry Pi USB Power Delivery whitepaper](https://pip-assets.raspberrypi.com/categories/685/app-notes-guides-whitepapers/documents/RP-009856-WP-1-USB%20Power%20delivery%20on%20Raspberry%20Pi%205.pdf)
- [Raspberry Pi NVMe documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#nvme-ssd-boot)
- [PiPedal system requirements](https://rerdavies.github.io/pipedal/SystemRequirements.html)
- [PiPedal audio configuration and published latency examples](https://rerdavies.github.io/pipedal/Configuring.html)
- [Pisound documentation](https://blokas.io/pisound/docs/)
- [HiFiBerry DAC2 ADC Pro documentation](https://www.hifiberry.com/docs/data-sheets/datasheet-dac2-adc-pro/)
- [NAM A2 guide](https://www.tone3000.com/guides/nam-a2-the-complete-guide)
- [Linux USB gadget testing](https://docs.kernel.org/usb/gadget-testing.html)
- [Raspberry Pi: Using the I2S peripherals on Raspberry Pi SBCs](https://pip-assets.raspberrypi.com/categories/1259-audio-camera-and-display/documents/RP-009699-WP-1-Using%20the%20I2S%20peripherals%20on%20Raspberry%20Pi%20SBCs.pdf)
- [TDO TL050WVH12CT-B1714A LCD module specification rev 4.0](references/datasheets/display/TL050WVH12CT-B1714A_LCD-Module-Spec_Rev4.0.pdf)
- [ESP-IDF: RGB interfaced LCD (ESP32-S3)](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/lcd/rgb_lcd.html)
- [ESP-FAQ: LCD](https://docs.espressif.com/projects/esp-faq/en/latest/software-framework/peripherals/lcd.html)
- [ESP32-S3-DevKitC-1 v1.0 user guide](https://docs.espressif.com/projects/esp-dev-kits/en/latest/esp32s3/esp32-s3-devkitc-1/user_guide_v1.0.html)
