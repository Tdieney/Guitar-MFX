# Guitar-MFX Decision Record and Supporting Evidence

| Field | Value |
|---|---|
| Version | 0.8 |
| Date | 2026-10-14 |
| Status | Pre-hardware; project measurements have not yet been recorded |
| Purpose | Authoritative rationale and evidence behind decisions kept in the numbered project documents |

## 0. Context for contributors

Guitar-MFX is a personal embedded electric-guitar multi-effects unit and a learning project for the user's embedded skills. It is played by one person, with one guitar, in a quiet room. The final processor is a Raspberry Pi 5 Model B running Linux, PiPedal, LV2 effects, and native NAM A2. Three custom Guitar-MFX boards — an audio board and a main board connected to the Pi 5 40-pin header by ribbon, and a front-panel board carrying the ESP32-S3 behind the display — integrate the mixed-signal audio front end, outputs, power tree, protected control I/O, mute relays, control surface, and panel connections. An ESP32-S3-WROOM-1-N16R8 renders the complete local UI on a 5.0-inch 800×480 RGB touchscreen and handles the performance controls.

Project priorities remain:

1. Useful effects and convincing captured-amplifier tone.
2. Low, stable playing latency.
3. Beginner-friendly controls and presets.
4. Bidirectional USB audio and model-management workflow.
5. A serviceable DIY hardware design.

Targets, vendor specifications, community measurements, and project measurements are different evidence classes:

- **Target:** desired result; not evidence that hardware achieves it.
- **Vendor/source claim:** attributed to its source and not treated as a Guitar-MFX result.
- **Community measurement:** useful context with method/configuration limitations.
- **Project measurement:** exists only when recorded with conditions in [03-Measurements.md](03-Measurements.md).

## 1. Success criteria and evidence ownership

| Criterion | Current target or decision | Evidence required |
|---|---|---|
| Analog round-trip latency | Aspirational ≤3 ms; first prototype gate ≤5 ms | Physical analog loopback in [03-Measurements.md](03-Measurements.md) |
| Captured-amp tone | Native NAM A2-Full on Pi 5 with calibrated input gain | Owner blind A/B and calibration record in `03-Measurements.md` |
| Guitar input | At least 1 MOhm with headroom for agreed passive/active sources | Electrical measurements in `03-Measurements.md` |
| Audio stability | Zero xruns for the applicable one-hour/four-hour gate | Versioned soak result in `03-Measurements.md` |
| USB audio | Reliable class-compliant full-duplex operation | Native-versus-bridge Phase 3 comparison and soak results |
| Guitar-MFX boards | Safe rails, correct sequencing, low-noise analog path, default mute | Staged bare-board and integrated bring-up records |
| Display | Stable 800×480 RGB image at DCLK ≥ 23 MHz on the ESP32-S3 | Display gate records in `03-Measurements.md` §3.13 |

The exact gate definitions live in [02-Architecture.md](02-Architecture.md). This record explains why they exist; it does not contain project results.

## 2. NAM A2 evidence and implications

The following comes from the **TONE3000 developer/vendor guide**, not from Guitar-MFX measurements:

- One A2 download contains A2-Full and A2-Lite; the host selects which width to run.
- A2-Full is intended for pro-audio systems with processing headroom.
- A2-Lite is the reduced-width form for resource-constrained embedded hardware.
- TONE3000 reports A2-Full as roughly 30–40% more CPU-efficient than A1-Standard.
- TONE3000's listening and quantitative comparisons were produced by an interested ecosystem participant. They are useful evidence but not an objective Guitar-MFX requirement or substitute for owner A/B testing.

Source: [NAM A2 complete guide](https://www.tone3000.com/guides/nam-a2-the-complete-guide).

### Consequences

- Raspberry Pi 5 runs A2-Full as the baseline. A2-Lite is not used as a Pi 5 latency shortcut.
- Input-gain calibration and converter behavior remain critical even when the neural model is accurate.
- The deferred hybrid architecture can run A2-Lite on an MCU and A2-Full on Linux from one model library without a separate conversion pipeline.

### 2.1 Evidence added 2026-10

All items are **vendor/source claims or community demonstrations**, not Guitar-MFX measurements:

- A2 became the standard NAM architecture on 2026-06-02. TONE3000 states that A2-Lite runs at about 50% CPU on a 600 MHz Arm Cortex-M7 costing about US$3 in volume.
- TONE3000 reports a MUSHRA listening test with more than 1,000 participants in which A2-Lite scored alongside A1-Standard and also beat several commercial modellers, and A2-Full beat Neural DSP V1/V2, IK ToneX V2, and Line 6 Proxy by a wide margin. TONE3000 is an interested party.
- A community demo runs A2-Lite (about 1,871 parameters) on a Raspberry Pi Pico 2 (RP2350, dual Cortex-M33 at 300 MHz) at 48 kHz with about 1 ms latency, as a USB-audio loopback without a codec.
- NAM ports exist for the Daisy Seed (STM32H750 Cortex-M7).
- PiPedal 2.0 supports NAM A2 and can browse and install TONE3000 models directly; users report stable operation and about 5 ms round trip with a complex preset (community report).

**Implication:** A2-Lite is no longer only a reduced-quality fallback; an MCU-only design became credible for tone. This triggered the platform re-evaluation recorded under ADR-005. The Pi 5 architecture was retained.

## 3. Architecture Decision Records

### ADR-001 — Linux is the primary DSP platform

**Decision:** Use Raspberry Pi 5 with Linux and PiPedal rather than a bare-metal MCU as the main effects host.

**Rationale:** The project prioritizes a broad effects library, editable routing, presets, networking, native model management, and bidirectional USB audio. Rebuilding the equivalent LV2/plugin ecosystem on an MCU would substantially expand scope.

**Consequence:** Linux scheduling, boot time, storage integrity, thermal behavior, and controlled shutdown become project responsibilities. Low latency must be measured rather than assumed.

**Status:** Existing project decision; prototype validation required.

---

### ADR-002 — Latency target is aspirational, with a staged gate

**Decision:** Aim for analog round-trip latency no greater than 3 ms, but use 5 ms as the first playable prototype gate.

**Rationale:** Third-party hardware spans roughly sub-millisecond to several milliseconds depending on routing. PiPedal's published examples also vary by interface and buffer configuration. A staged gate avoids declaring a hardware capability before testing while still keeping the design competitive.

**Consequence:** Measure analog input to analog output for every relevant codec, buffer/period setting, real-time configuration, and representative effect chain. Owner perception is recorded alongside the number.

**Status:** Engineering target; no Guitar-MFX measurement yet.

---

### ADR-003 — PiPedal 2.x is the baseline audio host

**Decision:** Evaluate PiPedal rather than first writing a custom LV2 host.

**Rationale:** PiPedal documents Raspberry Pi 5 support, pedal-oriented presets and controls, LV2 hosting, and NAM A2 support. This is **project documentation from PiPedal**, not a Guitar-MFX performance result. See [PiPedal](https://rerdavies.github.io/pipedal/) and [system requirements](https://rerdavies.github.io/pipedal/SystemRequirements.html).

**Consequence:** The project depends on an external open-source project and must pin versions, preserve a recoverable image, and verify every required workflow.

**Status:** Existing project decision; prototype validation required.

---

### ADR-004 — ESP32-S3 is the UI and safety coprocessor

**Decision:** Keep touchscreen, touch controller, footswitches, encoder, UI rendering, heartbeat, and output-mute supervision on an ESP32-S3. The specific module and display are defined in ADR-015.

**Rationale:** Separating UI work prevents display/touch load from sharing the Linux audio deadline and provides a fast-starting controller for boot/fault indication. It also gives mute supervision that keeps working when Linux hangs.

**Consequence:** A versioned, framed UART protocol and explicit resynchronization behavior are required. Audio never crosses this link.

**2026-10 note:** the user questioned whether the Pi 5 could drive the display and run the UI itself. Research showed: ST7796 support on Pi 5 relies on the deprecated `fbtft` subsystem or third-party drivers; FT6336 has no in-tree Pi overlay; documented reports show browser/GUI activity causing audio stutter through cache and memory-bandwidth contention, which `isolcpus` does not remove; and the Pi gives no UI during its boot and no mute if Linux hangs. For a parallel RGB panel, Pi 5 DPI modes overlap the I2S pins. The separate ESP32-S3 was therefore kept, with more justification than before.

**Status:** Existing project decision, re-confirmed 2026-10; integration not yet validated.

---

### ADR-005 — Raspberry Pi 5 Model B remains the final processor

**Decision:** Use the same Raspberry Pi 5 SBC for development and the final unit.

**Rationale:** One platform avoids a late processor/module migration and keeps the measured CPU, drivers, USB controller, storage paths, cooling hardware, and software image relevant to the final assembly. The standard SBC is also easier to replace during DIY development.

**Consequences:**

- The final enclosure must accommodate the SBC and its physically present HDMI, USB-A, Ethernet, microSD, USB-C, cooler, and PCIe FFC features.
- Ethernet is useful for development even though it is not required product I/O.
- There is no onboard eMMC; microSD and optional NVMe need a measured storage decision.
- The Pi 5 USB-C receptacle is involved in both normal power and native gadget data, creating the unresolved USB/power problem in ADR-011.

**Re-evaluation (2026-10):** prompted by the A2 evidence in §2.1 and by the user's interest in embedded-Linux skills, the following alternatives were compared:

| Alternative | Strengths | Why not selected |
|---|---|---|
| Compute Module 5 on a custom carrier | Same BCM2712 SoC and RP1 as Pi 5. USB 2.0 D+/D− (pins 103/105) and `USB_OTG_ID` (pin 101) on the module connector would remove the Pi 5 USB-C power/data conflict; `PMIC_Enable` (pin 99) gives hardware power-off; eMMC 16–64 GB; ECC LPDDR4x; production to at least January 2036; from US$45 (vendor figures). | The carrier needs a 5 V/5 A power tree with datasheet-defined sequencing, fine-pitch Amphenol connectors, and a 90 Ω USB pair; the CM5 datasheet states OTG is "not officially documented"; CM5 has less passive heat sinking than Pi 5. The user chose the easier Pi 5 after its feasibility was confirmed. |
| Rockchip / Allwinner modules (Radxa CM3/CM4/CM5, Orange Pi CM4, Banana Pi CM5 Pro) | Cheap SoCs; some CM4-compatible modules from about US$23. | Compared module to module, prices are similar to CM5 (Radxa CM5 about US$75). Rockchip BSP kernels are 5.10/6.1 while PREEMPT_RT entered mainline arm64 in Linux 6.12, so required peripherals may force a non-RT kernel. NPUs do not help per-sample NAM inference. Carrier reference material is thinner. No measured RT-audio data was found — this is a support-risk argument, not proof of inferiority. |
| NXP i.MX8M Plus module + Yocto | Industrial embedded-Linux standard; on-chip Cortex-M7; Elk Audio OS support; 10–15-year longevity. | Highest software and hardware effort, little hobby support, and no prebuilt PiPedal — too many unknowns for a first audio project. |
| MCU-only (STM32H7 / Daisy class) with A2-Lite | Instant start, ~1–2 ms latency, deepest embedded learning. | Every effect must be written by hand (UR-AUD-005 requires a broad reorderable chain), and bare-metal real-time audio is a poor fit for AI-assisted code. Candidate for a later revision. |

Pi 5 feasibility was then checked: PiPedal 2.0 runs NAM A2 on Pi 5 (§2.1); a community write-up shows Pi 5 enumerating as a UAC2 gadget; and a dedicated USB bridge remains a second candidate if the native gadget fails (ADR-011), although its connection to the Pi's single I2S bus is still to be proven. With two candidate USB paths, UR-CON-002 does not rest on one approach. The user retained Pi 5 Model B.

Note: Elk Audio OS officially supports Pi 4 / CM4 for its low-latency drivers but not Pi 5 / CM5 (vendor documentation, 2026-09).

**Status:** Selected by the user as project scope; re-confirmed 2026-10; performance gates remain open.

---

### ADR-006 — Initial sample rate is fixed at 48 kHz

**Decision:** Use 48 kHz for the first complete implementation.

**Rationale:** It is the native working rate assumed by the selected guitar-model workflow and limits CPU, converter, and USB test combinations. A higher number alone would not demonstrate better sound.

**Consequence:** Codec, USB, plugin, and measurement configurations must explicitly use 48 kHz unless a controlled experiment says otherwise.

**Status:** Existing engineering decision; signal integrity and performance remain to be measured.

---

### ADR-007 — Build a custom Guitar-MFX board on the Pi 5 header

**Decision:** Design and fabricate a custom mixed-signal board that mates with the Pi 5 40-pin header and replaces the commercial audio HAT in the final product.

**Rationale:** The final unit needs a calibrated guitar input, selected codec/clocking, line and headphone outputs, default mute, product power tree, protected foot/expression/control I/O, ESP32 interface, and panel USB routing in one serviceable assembly.

**Consequences:**

- Commercial HATs remain bench tools only.
- Codec and analog decisions have real PCB/respin cost.
- Pisound and HiFiBerry DAC2 ADC Pro are both required Phase 0 reference paths; a single measured path is insufficient.
- Production-relevant codec candidates must pass the comparative gate in [01-Hardware.md](01-Hardware.md) §11.
- The first board is expected to require at least one respin, so test points and staged bare-board power-up are mandatory.

**Status:** Selected by the user as project scope; **amended by ADR-013** (three boards — audio, main, front-panel — instead of one header-mounted board). Schematic and layout are not yet validated.

---

### ADR-008 — High-impedance, calibrated guitar input

**Decision:** Use an input impedance of at least 1 MOhm plus repeatable analog/digital gain calibration.

**Rationale:** Passive pickups can be loaded by lower impedances, and NAM response depends on the level delivered to the model. Pisound's vendor-specified 100 kOhm input is therefore not the final direct passive-guitar input.

**Consequence:** Input protection, RF filtering, buffer stability, full-scale level, headroom, clip detection, and calibration must be verified on the actual circuit.

**Status:** Design requirement; not yet measured.

---

### ADR-009 — Default mute replaces traditional true bypass

**Decision:** Use a normally-muted relay or analog mute circuit, supervised by ESP32-S3.

**Rationale:** The current loads are powered full-range speakers and headphones. Passing a passive guitar directly to them would be quiet and would omit amp/cab processing, so pedal-style raw true bypass does not provide the intended safe fallback.

**Consequence:** Boot, shutdown, Linux crash, codec restart, preset change, and control-link loss tests must verify silence or a controlled ramp.

**2026-10 amendment:** guitar-amplifier use was added (ADR-017) through the main outputs, with no dedicated amplifier output, so there is no true-bypass path and all outputs default to silent. The mute circuit is specified in ADR-017.

**Status:** Design requirement; circuit not yet measured.

---

### ADR-010 — Hybrid audio processing remains deferred

**Decision:** Do not add an MCU audio front end unless the measured Pi 5 system misses the latency/feel gate after buffer/period, effect-chain, real-time scheduling, and thermal work.

**Rationale:** Hybrid processing adds a sample-accurate digital link, clock-domain decisions, delay compensation, two firmware targets, and preset synchronization. It should solve a measured problem rather than a predicted one.

**Consequence:** A low-cost deterministic MCU board may be borrowed or purchased solely as an optional perceptual reference before the owner judges the Phase 1 gate. If hybrid is later justified, one A2 file can serve A2-Lite on the MCU and A2-Full on Linux.

**Status:** Deferred; not accepted architecture.

---

### ADR-011 — Native gadget and dedicated USB bridge are co-primary candidates (amended by ADR-020)

**Decision:** Evaluate native Pi 5 UAC2 gadget mode and a dedicated class-compliant USB audio bridge before selecting the board architecture.

**Rationale:** Raspberry Pi's **vendor OTG whitepaper** confirms the native SoC OTG peripheral is exposed through the Pi 5 USB-C power connector while RP1 host ports do not support OTG. It also instructs users to power Pi 5 through the GPIO header to free USB-C for device operation. This establishes platform capability, not Guitar-MFX full-duplex UAC2 reliability.

The proposed native product path uses a panel USB-C connector, short cable/interposer to the Pi 5 USB-C receptacle, isolated host VBUS, and main power through the header. This may fail because USB-C role detection depends on CC and VBUS-presence behavior. Header power also bypasses USB-PD negotiation and requires the custom board to provide measured inrush limiting, sequencing, protection, capacitance, and brownout behavior.

**Consequence:** Before PCB layout, Phase 3 must use a hand-made cable or adapter to prove the VBUS-isolated concept and compare it against a dedicated bridge on BOM, added latency, available Pi interfaces alongside the codec, host behavior, clocks, and development risk. A legacy-OTG-style cable may be required for investigation.

On Pi 5 Model B the `dwc2` overlay belongs under `[all]`, and the USB-C connector carries USB 2.0 data only:

```ini
[all]
dtoverlay=dwc2,dr_mode=peripheral
```

Sources: [Raspberry Pi OTG whitepaper](https://pip-assets.raspberrypi.com/categories/685/app-notes-guides-whitepapers/documents/RP-009276-WP-1-Using%20OTG%20mode%20on%20Raspberry%20Pi%20SBCs), [USB Power Delivery on Raspberry Pi 5](https://pip-assets.raspberrypi.com/categories/685/app-notes-guides-whitepapers/documents/RP-009856-WP-1-USB%20Power%20delivery%20on%20Raspberry%20Pi%205.pdf), and [Linux UAC2 gadget framework](https://docs.kernel.org/usb/gadget-testing.html).

**2026-10 evidence:** a community lab write-up documents Pi 5 enumerating as a UAC2 gadget ([RASPIAUDIO CamillaDSP notes](https://github.com/RASPIAUDIO/CamillaDSP/blob/main/docs/usb_audio_gadget_pi5.md)); this supports the software path, not the VBUS-isolated panel arrangement. Raspberry Pi's I2S whitepaper states that `I2S0` (clock producer) and `I2S1` (clock consumer) on Pi 5 each have up to four data lanes, but they occupy the same GPIO pins and Pi 5 cannot act as both at the same time (p. 8). **Correction 2026-10-14:** an earlier version of this record said a bridge could use a second I2S bus; that was wrong. A bridge must share the codec's BCLK/LRCK on an extra data lane (single clock domain) or use another link; feasibility depends on the bridge chip and is unverified.

**Status:** Amended 2026-10-14 by ADR-020: native gadget first, bridge as fallback. The VBUS-isolation approach still requires bench proof.

---

### ADR-012 — Measurements have one authoritative home

**Decision:** Store every project measurement only in [03-Measurements.md](03-Measurements.md).

**Rationale:** Earlier documents mixed targets, community reports, vendor specifications, and future measurements. A versioned ledger with complete conditions prevents a number from losing its hardware/software context.

**Consequence:** Other documents state targets and decisions, then link to the ledger. Failures remain in the record and are not rewritten as successes.

**Status:** Documentation rule.

---

### ADR-013 — Three Guitar-MFX boards (audio, main, front-panel), not a HAT

**Decision:** Split the product hardware into three boards:

- **Audio board:** input, PGA, codec, output stages, local LDOs, mute relays and their master-mute gate.
- **Main board:** power tree, external footswitch and expression jacks, panel USB-C, and signal routing between the boards.
- **Front-panel board:** ESP32-S3-WROOM-1-N16R8 module, display and touch connectors, backlight driver, local 3.3 V rail, I/O expander, encoder, power button.

The audio and main boards connect to the Pi 5 40-pin header by ribbon and sit beside the Pi. The front-panel board sits behind the display. The HAT form factor is not used.

**Rationale:**

- The main argument is **sequencing**: the audio board can be built, connected to a Pi 5 by ribbon, powered from a bench supply, and measured for noise floor before the main board exists. For a first audio design, an early measured noise floor is the most valuable result.
- **Revision isolation:** the audio board is expected to be revised two or three times; the main board should not have to be re-spun with it.
- Separation from switching regulators helps noise, but a single large board with careful zoning could achieve most of that benefit, so this is a secondary reason.
- The HAT form factor (65 × 56 mm, mounted 8–11 mm above the Pi 5 switching regulators) works against the quiet-room noise priority; UR-MEC-001 already allows a larger enclosure.
- **Front-panel board:** the RGB display bus (20 lines at 23–27 MHz) must stay short, so the ESP32-S3 sits behind the display rather than on a main board beside the Pi. It can also be brought up on the bench before the main board exists.

**Consequences:** a ribbon carries I2S and I2C onto the audio board (source-end series resistors, interleaved grounds, short length); a board-to-board connection carries the pre-regulated rail and mute lines; a cable from the main board carries the supply, mute line, and external-control signals to the front-panel board. The mute-gate inputs are pulled to the muted state on the audio board, so an unplugged cable leaves the outputs silent. More connectors and assembly steps than a single board. The exact interconnect and pinouts are open. A small 2-layer display bring-up adapter is planned as the first PCB of the project.

**Alternatives considered:**

- One large board with spatial separation — technically valid; rejected by the user in favour of independent iteration.
- **Two shields** (user proposal, compared 2026-10-14): an audio shield stacked on the Pi 5 and a UI shield on the 44-pin ESP32-S3 development board, with the Pi 5 powered by the official USB-C adapter. Less design work, and no header-power or USB-C VBUS-isolation risk. Trade-offs: the analog section sits directly above the Pi 5 switching regulators on a tall stacking header (to clear the cooler); the development board stays in the product; native USB gadget is impossible while the adapter occupies the Pi 5 USB-C port, so USB audio needs a bridge chip or moves to v2. The tone-determining circuits are identical in both plans. Not selected.

**History:** 2026-10-08 — two boards (audio, main), with the ESP32-S3 on the main board unless moved to a front-panel board. 2026-10-14 — three boards selected by the user after comparing them with the two-shield plan; the front-panel board is part of the set.

**Status:** Selected by the user 2026-10-14; not yet designed.

---

### ADR-014 — Complete local UI on the ESP32-S3 through a Pi-side bridge daemon

**Decision:** The local screen alone is sufficient for playing (UR-UI-009). The ESP32-S3 renders it with LVGL. A bridge daemon on the Pi is the only component that talks to PiPedal and exposes a compact binary UART protocol to the ESP32-S3. The PiPedal web interface remains for maintenance (Wi-Fi, plugin installation, model/IR file management, audio-device configuration).

**Rationale:** PiPedal has no native local-display mode, and its websocket API is internal and undocumented, reverse-engineered only for version 2.0.110 by a community project that warns it may change. Placing all PiPedal coupling in one Pi-side daemon means an API change requires a daemon update only, not a firmware update. Keeping deep setup in the web UI avoids text entry and file browsing on a 5-inch screen, and bounds the protocol to about 8–10 message types.

**Alternatives considered:** a Chromium kiosk on the Pi showing the PiPedal web UI (no UI code to write, but browser load on the audio computer is a documented stutter risk and the display/touch drivers are weak on Pi 5); a native UI app on the Pi (same driver and contention problems, plus the API dependency).

**Status:** Selected; not yet implemented.

---

### ADR-015 — Display and control MCU

**Decision:** Use the purchased **TDO TL050WVH12CT-B1714A** (5.0-inch, 800×480, IPS, parallel RGB, GT911 touch) driven by an **ESP32-S3-WROOM-1-N16R8**. Development uses the purchased 44-pin board with the same module. Selection of the ESP32-S3 is conditional on the display gate ([01-Hardware.md](01-Hardware.md) §6.4). Fallback: **STM32U5G9**.

**Rationale:**

- The display needs only a single 3.3 V supply and is IPS (wide viewing angles), per the vendor specification.
- 800×480 is the resolution at which ESP32-S3 RGB driving is widely demonstrated; Espressif's reference runs at 21 MHz PCLK with about 26 fps LVGL benchmark. Higher resolutions such as 1024×600 push the ESP32-S3 past its PSRAM bandwidth (vendor and community reports).
- The WROOM-1-N16R8 module contains its flash and octal PSRAM, so the custom PCB does not route memory buses. It is the same module as on the development board.
- The architecture avoids the two documented causes of RGB drift on ESP32-S3: it uses no Wi-Fi and does not write flash during play.

**Risks:** the panel's minimum DCLK is 23 MHz (25 MHz typical), above the ESP32-S3's comfortable 21 MHz but under its ~30 MHz ceiling; DCLK polarity is fixed inside the panel and unknown; 30 of 31 GPIOs are used.

**Alternatives considered:**

| Option | Assessment |
|---|---|
| Keep ST7796 3.5-inch + FT6336 | Superseded by the user's larger-display requirement; also poorly supported if driven from Linux. |
| ESP32-P4 | No PCLK ceiling, faster PSRAM, MIPI-DSI and 2D accelerator; newer ecosystem with open RGB-panel issues; harder custom PCB; not owned. |
| **STM32U5G9** (Cortex-M33 160 MHz, 4 MB flash, **3 MB internal SRAM**, LTDC, NeoChrom GPU, LQFP144) | Holds two 800×480 RGB565 frames internally with no external RAM; plentiful GPIO; ST's STM32U5G9J-DK2 kit includes a 5-inch 800×480 RGB touch panel and an official LVGL port; higher embedded-learning value. Not chosen because the user already owns the ESP32-S3 board and the module-based PCB is simpler. Selected as fallback. |
| Renesas RA8D1 | 1 MB SRAM fits only one frame; designs usually add SDRAM; smaller community. |
| STM32H7 / NXP i.MX RT | Usually need external SDRAM for double buffering; BGA packages on several parts; too hard for a first custom PCB. |
| Pi 5 DPI | DPI modes overlap the I2S pins needed by the codec. |

**Re-evaluation (2026-10-13):** the user asked whether a cheaper chip could do better with the same 5.0-inch panel. Vendor list prices checked that day (comparison only, not quotations): ESP32-S3-WROOM-1-N16R8 module about US$5.2 at 1 piece and US$3.6 at 100 ([LCSC](https://www.lcsc.com/product-detail/C2913202.html)), including flash and PSRAM; STM32U5G9ZJT6Q chip alone about US$13–28 depending on distributor ([Mouser](https://www.mouser.com/en/ProductDetail/STMicroelectronics/STM32U5G9ZJT6Q?qs=pBunHO6%2BwtHTKyjOWi%2Btmg%3D%3D), [LCSC](https://www.lcsc.com/product-detail/C22445371.html)); Allwinner T113-S3 (Cortex-A7 with 128 MB DDR3 in package) about US$18 at [LCSC](https://www.lcsc.com/product-detail/Microcontroller-Units-MCUs-MPUs-SOCs_Allwinner-Tech-T113-S3_C5197687.html), plus external flash and several supply rails. Cheaper MCUs with a display controller lack the RAM for one 800×480 RGB565 frame and need external SDRAM; RP2350 has 520 KB SRAM. Conclusion: no cheaper chip does better for this panel. The user kept the ESP32-S3; the display runs RGB565.

**Status:** Display purchased; ESP32-S3 selected subject to the display gate; re-confirmed 2026-10-13.

---

### ADR-016 — Global analog gain calibration, per-preset digital model level

**Decision:** The input provides 1 MOhm impedance and a software-set PGA of 0 to +30 dB. The PGA setting is a global calibration for the user's guitar and is not stored per preset. The level fed to each NAM model is adjusted digitally in PiPedal and stored with the preset.

**Rationale:** Estimated source levels range from tens of millivolts (light single-coil playing) to about 2 V peak (hard playing on active or hot pickups), roughly 30 dB. NAM models are level-dependent, so the drive level must be repeatable. The user has one guitar, so a per-preset analog gain would add PGA switching on every preset change without benefit.

**Alternatives considered:** per-preset analog gain (possible clicks on preset change); fixed trimpot (not repeatable).

**Status:** Selected; circuit not yet designed or measured.

---

### ADR-017 — Output set, mute, and volume

**Decision:**

- Outputs, following common commercial multi-effects units: **Main out L/MONO and Main out R** on 6.35 mm jacks at line level, and a **3.5 mm headphone out** with its own analog volume potentiometer (sized for up to 300 Ω). No dedicated guitar-amplifier output. One stereo mix, so the codec needs 2 DAC channels.
- A guitar amplifier connects to the main outputs. The user switches the cabinet IR off in the settings and, for the amplifier's front input, bypasses the amp model block. A global Line / Instrument level setting and an optional mono sum on L/MONO apply to the main outputs.
- Inserting headphones silences the main outputs.
- Mute: two relays at the jacks, after the output drivers, muted when unpowered; one direct ESP32-S3 master-mute GPIO through a logic gate drives both; per-output enables come from the I/O expander.
- Volume: two independent analog potentiometers in the signal path — **Master** (both main outputs) and **Phones** (headphones). The single encoder is for navigation and parameter editing only. The digital output level of the audio graph ramps from silence at startup.

**Rationale:** The user plays through headphones, powered speakers, a computer, and a guitar amplifier. Commercial units such as the Line 6 HX Stomp serve amplifiers, mixers, and monitors from the same L/MONO and R jacks with a global Instrument/Line setting ([HX Stomp manual](https://line6.com/data/6/0a020a4010c935bb66a4c0c44f/application/pdf/HX%20Stomp%20Manual%20-%20English%20.pdf)); a separate amplifier output with its own attenuator adds parts without adding capability. Relays before the drivers would not stop driver power-up transients. The master-mute path must not depend on the I/O expander or on Linux. When Linux hangs, ALSA DMA typically replays the last buffer as a loud buzz, so the hardware mute must not depend on Linux. Analog potentiometers keep volume control working when software fails and give an absolute knob position. Two knobs rather than one, because headphone insertion mutes the main outputs: with one shared knob, headphones would start at the speaker level. Many compact multi-effects units use a single volume knob (for example, the HX Stomp's VOLUME knob can be set to control main and headphone outputs together, or headphones only); the two-knob layout here resembles an audio interface, which this unit also is.

**History:** On 2026-10-08 the user first chose an encoder-driven digital master volume, then reverted to an analog master potentiometer the same day, and chose separate Master and Phones knobs over a single knob. On 2026-10-13 the headphone jack changed from 6.35 mm to 3.5 mm. On 2026-10-14 the separate guitar-amplifier output (6.35 mm mono with attenuator and ground lift) and the three global output modes (Full-range / Amp FX return / Amp input) were withdrawn in favour of the main-output layout above.

**Alternatives considered:** one relay before the output split (does not stop driver transients); electronic mute with analog switches (fail-silent behaviour harder to guarantee); encoder-driven digital master volume (no hardware brake on line outputs; needs analog-domain codec volume with zero-cross steps); a single master knob for all outputs (level jump into headphones on insertion); single knob plus a headphone trim in the UI (loses the hardware headphone limit).

**Status:** Selected; circuits not yet designed or measured.

---

### ADR-018 — Codec: PCM1863 + PCM5122 for rev A, CS4282P as rev B candidate

**Decision:** The audio board rev A uses **TI PCM1863** (stereo ADC with PGA −12 to +32 dB) and **TI PCM5122** (stereo DAC), the converter pair of the HiFiBerry DAC2 ADC Pro, with the codec as I2S clock master from a single 49.152 MHz oscillator and low-latency converter filters. **Cirrus Logic CS4282P** is the rev B upgrade candidate, evaluated only if Phase 0 shows the converter limits the noise floor ([01-Hardware.md](01-Hardware.md) §11.2).

**Rationale (vendor specifications, not project results):**

- The PCM pair can be measured on the HiFiBerry HAT with the user's guitar before the board is drawn; no other shortlisted codec has an equivalent reference.
- Drivers (`pcm186x`, `pcm512x`) and an overlay for the pair already run on Raspberry Pi.
- PGA inside PCM1863 covers the required 0 to +30 dB; no separate PGA.
- TSSOP packages are hand-solderable; PCM1863 runs from one 3.3 V supply.
- Converter latency with low-latency filters is about 13.5/fs ≈ 0.28 ms at 48 kHz, against about 0.18 ms for CS4282P — a 0.1 ms difference, small next to one 64-sample Linux buffer (1.33 ms).

**Review of the CS4282P proposal (2026-10-14).** The user brought an alternative recommendation from another AI. Checked against the CS4282P datasheet (DS1318F1, August 2025), the Linux source, and the Raspberry Pi I2S whitepaper:

- Confirmed: ADC 123 dB and DAC 128 dB dynamic range, THD+N −110 / −115 dB; dynamic range quoted after deducting the external buffer's noise; current-output DAC needing an I/V stage; no internal PGA (Hybrid Gain Control drives an external preamp); mainline `cs530x` driver lists `cirrus,cs4282`.
- Confirmed and adopted: Pi 5 `I2S0` and `I2S1` share pins and cannot be used together (correction to ADR-005 and ADR-011).
- Not chosen for rev A because: 48-pin QFN with 0.4 mm pitch and exposed pad is impractical to hand-solder; it needs an external digitally controlled PGA, a differential input driver, and per-channel I/V stages, each adding noise that the 123 dB figure excludes; there is no HAT to measure it first; and for an electric guitar the noise floor is often set by the instrument (single-coil hum, room interference) amplified by high-gain amp models, so the 13 dB converter advantage may not be audible. The Phase 0 noise comparison decides this with data.

**Alternatives considered:** CS4282P (rev B candidate, above); TLV320ADC6140 + PCM5122 (113 dB ADC, PGA to +42 dB, but 4×4 mm QFN, no reference HAT, custom overlay); ES9290 (PGA to +30 dB, 116 dB, 5×5 mm QFN, no Linux driver found); ES9292PRO (announced August 2026, no Linux driver evidence); WM8960 (single chip with headphone driver, about 94 / 98 dB SNR — noisier); CS4272 (114 dB, TSSOP, but no PGA and Raspberry Pi driver not verified).

**Status:** Selected for rev A; measurement-gated.

---

### ADR-019 — 12 V input, soft power button, and power-loss strategy

**Decision:**

- **Input:** 12 V DC adapter, about 3 A, on a barrel jack.
- **Power control:** soft power button, as on the Quad Cortex. A press starts the unit; another press runs an orderly shutdown supervised by the ESP32-S3: mute, Linux poweroff, wait for the Pi 5 header 3.3 V to fall, cut Pi 5 power, display power-off sequence, then the controller's kill input. A long press forces power off. A push-button on/off controller on the main board stays powered.
- **Power loss:** no hold-up for the Pi 5. A power-fail comparator mutes the relays and cuts the Pi 5 in hardware; a few millifarads of hold-up let the ESP32-S3 finish the display power-off sequence. Linux runs from a read-only root; presets are written atomically on a separate data partition.

**Market comparison (2026-10-14; vendor and community sources):**

| Unit | How it turns off | Guidance |
|---|---|---|
| [MOD Dwarf](https://forum.mod.audio/t/dwarf-on-off-switch/6030) (Linux) | No power switch; unplug | MOD staff: unplugging does not damage the unit; leaving out the switch was a design decision. A user reports read-only or RAM-based system partitions ([community information](https://forum.mod.audio/t/power-disconnection-countermeasure-for-mod-dwarf/11523)). |
| [Neural DSP Quad Cortex](https://neuraldsp.com/manual/quad-cortex) | Capacitive power button, then SHUT DOWN | "Disconnecting the power cable while the device is running is not recommended." |
| [HeadRush Pedalboard](https://cdn.inmusicbrands.com/HeadRush/pedalboard/Pedalboard-Quickstart-Guide-v2_1_3.pdf) | Press and hold the power button | — |
| [Kemper Profiler](https://www.kemper-amps.com/forum/thread/11201-probably-a-silly-question-power-down-kemper/) | On/Off control | Forum guidance: switch the unit off before the power strip so the outputs are driven to a safe level. |

**Rationale:**

- **12 V:** common, inexpensive adapters and a simple buck to the Pi 5 rail. USB-C PD needs a PD sink controller and gives the unit two USB-C ports; 9 V pedal supplies rarely provide 3 A.
- **Soft button:** normal shutdowns are clean. Raspberry Pi's resilient-filesystem whitepaper names the page cache, in-flight writes, and SD card internal operations as power-loss corruption causes; a read-only root removes the first two, and an orderly shutdown also avoids the third in normal use. The relays already prevent pops, so pop control is not the reason.
- **No Pi hold-up:** finishing a Linux shutdown after power loss needs roughly 10 s at about 10 W (about 100 J, estimate), i.e. on the order of 10 F of supercapacitors plus charge and boost circuits. A read-only root makes power loss tolerable instead.
- **Hardware power-fail mute:** muting does not depend on firmware, and cutting the Pi 5 first saves hold-up energy for the display power-off sequence.

**Alternatives considered:**

- **Hard power switch, MOD style:** fewer parts and automatic start from a pedalboard power strip, but every power-off is a power loss and data safety rests entirely on the read-only root. Recommended in review for simplicity; not selected by the user.
- **Hard switch plus soft button:** most parts; not selected.
- **Supercapacitor hold-up for a full Linux shutdown:** cost and board area; not selected.
- **USB-C PD or 9 V input:** see rationale; not selected.

**Consequences:** the main board gains a push-button controller, Pi 5 load switch, ideal diode, hold-up capacitance, and power-fail comparator; the power-fail line is a further input to the audio-board mute gate; the ESP32-S3 handles Pi 5 power enable, Pi 3.3 V sense, button interrupt, power-off request, and power-fail flag through the I/O expander, with no new direct GPIO; the Pi image needs `POWER_OFF_ON_HALT=1`, `PSU_MAX_CURRENT`, a read-only root, and a data partition for PiPedal state; V-POWER and V-STORAGE gain power-loss rows.

**Status:** Selected by the user 2026-10-14; not yet designed or measured.

---

### ADR-020 — USB audio: native gadget first, bridge fallback; dry channel to the computer

**Decision:**

- Develop the native Pi 5 UAC2 gadget first. Keep a dedicated USB audio bridge as the fallback, built only if the native path fails P3-GATE. Reserve a spare connector carrying I2S lane 1 (GPIO22/23), BCLK, LRCK, and the codec MCLK so a bridge daughterboard needs no audio-board respin.
- USB channels for v1: three to the computer (processed L/R and the dry guitar signal after the calibrated PGA) and two from the computer, 24-bit, 48 kHz.
- Re-amping through the unit is deferred beyond v1.

**Rationale:**

- **Audio board unchanged:** the tone-critical board stays simple and hand-solderable, with no USB controller near the analog section.
- **Testable without a PCB:** the native path needs only the Pi 5 and a hand-made cable, so Phase 3 decides with measurements before the main board is drawn.
- **Little hardware:** header power (ADR-019) already frees the Pi 5 USB-C port, one reason for the three-board plan (ADR-013).
- **Learning goal:** USB gadget, ALSA, and clock-domain work are core embedded-Linux topics.
- **Why not the bridge first:** an XMOS-class controller is a fine-pitch QFN, needs its own firmware toolchain and a development kit, and its risks appear only after hardware is built. Raspberry Pi's I2S whitepaper shows four-lane use only with the Pi as clock producer (HiFiBerry DAC8x); multi-lane operation with the Pi as clock consumer is not yet shown by a product.
- **Dry channel:** common on mid-range and budget units — [HX Stomp](https://line6.com/support/topic/65525-hx-stomp-reamp-question) (dry on USB 7), [Boss GX-100](https://support.roland.com/hc/en-us/articles/4508370106651-GX-100-Applying-GX-100-effects-to-DAW-playback-audio-and-recording-as-a-new-track), [Boss GX-10](https://support.roland.com/hc/en-us/articles/30332301042843-GX-10-Applying-Effects-to-Dry-Sound-after-Recording-Re-amping), [Valeton GP-200](https://www.fullcompass.com/prod/644365-valeton-gp-200-gp-200-multi-effects-processor-black) (6-in/4-out USB). With the native path it costs only software.
- **Re-amping deferred:** the user observed that a dry track can be turned into a finished track on the computer; the free [NAM plugin](https://github.com/sdatkinson/NeuralAmpModelerPlugin) runs the same model files there. Re-amping through the unit needs input switching in the effect chain and is not needed for v1.

**Fallback trigger:** the native path fails P3-GATE — device role cannot be held reliably with isolated VBUS, or the four-hour duplex run causes xruns on the local playing path with NAM A2-Full.

**Alternatives considered:** bridge first; deferring USB audio to v2 (would lower UR-CON-002 from Must); processed stereo only; dry channel plus re-amping in the first milestone.

**Consequences:** the routing service is the main software risk and must stay below the audio threads in priority; a spare connector and a few series resistors are added; open question 7 closes and question 10 narrows to "does the native path pass".

**Status:** Selected by the user 2026-10-14; not yet measured.

## 4. External latency context

These figures are not Guitar-MFX measurements. They illustrate why both numerical and perceptual comparisons matter.

| Device | Reported analog round-trip latency | Evidence class | Qualification |
|---|---:|---|---|
| Dimehead NAM Player | 0.5 ms including AD/DA | [Vendor claim](https://www.dimehead.de/faq/) | Dedicated neural player; method/configuration are not independently reproduced here |
| Line 6 Helix Floor | 1.8 ms for one DSP path | [Community measurement on Line 6 forum](https://line6.com/support/topic/37727-helix-as-audio-interface-round-trip-latency/) | Mainstream floor modeller; not an official specification |
| Neural DSP Quad Cortex | About 1.75–1.8 ms simple path; about 4.4–4.5 ms when crossing to another processing row/core in cited tests | [Community measurements on Neural DSP forum](https://unity.neuraldsp.com/t/qc-numbers-report-latency-send-levels-etc/10291) | Routing dependent; not an official specification |
| PiPedal with tested USB interfaces | Published examples vary by interface, buffer, and period settings | [PiPedal project measurements](https://rerdavies.github.io/pipedal/Configuring.html) | Useful platform context; not this project's HAT, kernel, or signal chain |

The Guitar-MFX 5 ms first-prototype gate is judged using project analog-loopback measurements, stability, and owner playing tests. The table is context, not pass evidence.

## 5. Phase 0 hardware rationale

| Item | Role | Decision basis |
|---|---|---|
| Raspberry Pi 5 + official 27 W supply + Active Cooler | Development and final processor baseline | Same SBC is used through the project; vendor-recommended power/cooling are a stable starting point |
| Pisound v1.2 | First HAT reference path | Convenient Pi 5 development interface with published ALSA/audio support |
| HiFiBerry DAC2 ADC Pro | Required second HAT reference path | Exposes interface/codec/filter/driver differences before the custom codec is chosen |
| Reference USB audio interface | Independent measurement and re-amp path | Prevents the device under test from measuring itself |
| Prototype 1 MOhm input stage | Guitar-source interface | Commercial HAT input alone does not satisfy the final passive-pickup requirement |
| 44-pin ESP32-S3-WROOM-1-N16R8 board + TDO 5.0-inch RGB display + display bring-up adapter | UI/control prototype and display gate | Purchased; adapter is the first project PCB (ADR-015) |
| Deterministic MCU audio board | Optional perceptual reference | Helps the owner compare playing feel before deciding whether hybrid complexity is justified |

No table entry is a measurement or production-part approval.

## 6. Linux real-time investigation order

The baseline investigation order is:

1. Fixed 48 kHz operation.
2. Measured buffer-size and period-count sweep.
3. PREEMPT_RT-capable kernel and verified real-time permissions/scheduling.
4. Performance governor and thermal monitoring.
5. Effect-chain and plugin profiling with A2-Full.
6. IRQ/thread affinity or core isolation only where profiling shows benefit.
7. Wi-Fi/background-service experiments with every change recorded.

If this misses the gate, an alternative such as a Xenomai/EVL dual-kernel or a purpose-built low-latency Yocto/Elk-style system may be investigated. That path is listed as a risk response, not accepted architecture; Pi 5 support, PiPedal compatibility, maintenance, and measured benefit are unknown.

## 7. Power, storage, and board implications

- The 12 V input (ADR-019) uses a synchronous buck for the Pi 5 rail; a multi-ampere 12 V-to-5 V linear regulator is not acceptable.
- Power control and power loss follow ADR-019: soft power button, orderly shutdown, no Pi hold-up, read-only root, hardware power-fail mute.
- Low-noise LDOs remain candidates only for controlled low-current analog/reference rails.
- Phase 0 boots from high-endurance microSD.
- Raspberry Pi **vendor documentation** supports Pi 5 NVMe boot through the PCIe FFC connection. NVMe remains optional because adding PCIe/M.2 hardware materially increases layout, signal-integrity, power, cooling, and mechanical scope. See [Raspberry Pi NVMe documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#nvme-ssd-boot).
- Atomic preset writes, a known-good preset, recoverable images, and forced-power-loss testing apply to either storage choice.
- Each Guitar-MFX first article is expected to require at least one respin. Board bring-up starts with unpopulated/bare-board checks and current-limited rail verification before Pi 5 is installed.
- Audio-board rails are made by local low-noise LDOs (about ±12 V for op-amps, 3.3 V for the codec; proposed) from a pre-regulated dual rail of about ±13 V supplied by the main board (two-stage regulation).
- The display adds a 3.3 V load shared with the ESP32-S3 and a backlight boost driver of about 1 W.

## 8. Open decisions

- Headphone model and impedance (stage designed for up to 300 Ω meanwhile).
- Confirmation of the rev A codec (ADR-018) by Phase 0 measurements, and whether CS4282P is evaluated for rev B.
- If the bridge fallback is activated: bridge chip (must be an I2S clock consumer on the codec MCLK), daughterboard design, and the Pi 5 multi-lane consumer overlay.
- Whether the ESP32-S3 passes the display gate at DCLK ≥ 23 MHz, or the STM32U5G9 fallback is needed.
- Physical interconnect between the three boards and the Pi 5 header: which cable carries which signals, connector types, and pinouts.
- Whether the display gate runs on the bring-up adapter before front-panel rev A is drawn, or on front-panel rev A directly.
- Whether the unit turns on by itself when 12 V is applied (useful with a pedalboard power strip).
- Push-button controller part, hold-up capacitance, and shutdown timeout (sized at schematic design, measured under V-POWER).
- Final analog rail voltages (about ±12 V proposed).
- Whether the native gadget passes P3-GATE or the bridge fallback is needed.
- Which board carries the bridge-fallback connector (decided with the board interconnect).
- Host operating systems and recording applications to support.
- Exact panel USB-C CC/VBUS isolation implementation, if native gadget wins.
- Final microSD or NVMe boot/storage medium.
- Onboard footswitch count, expression input, and MIDI requirements.
- Whether the optional deterministic latency reference is needed before the Phase 1 owner judgement.
- Whether an alternative real-time OS approach or hybrid architecture is justified by failed measurements.

## 9. Supporting sources

### NAM and A2

- [NAM A2 complete guide](https://www.tone3000.com/guides/nam-a2-the-complete-guide)
- [Introducing NAM Architecture 2 (TONE3000)](https://www.tone3000.com/blog/introducing-neural-amp-modeler-nam-architecture-2-a2)
- [Architecture A2 (Neural Amp Modeler author)](https://www.neuralampmodeler.com/post/architecture-a2)
- [Running NAM on embedded hardware (TONE3000)](https://www.tone3000.com/blog/running-nam-on-embedded-hardware)
- [NAM A2-Lite on Raspberry Pi Pico 2 demo](https://github.com/oyama/pico-neural-amp-modeler-demo)
- [GuitarML NeuralSeed](https://github.com/GuitarML/NeuralSeed) and [Mercury](https://github.com/GuitarML/Mercury)
- [NeuralAmpModelerCore](https://github.com/sdatkinson/NeuralAmpModelerCore)
- [TONE3000 embedded NAM pedal reference](https://github.com/tone-3000/nam-pedal)
- [Slimmable NAM paper](https://arxiv.org/abs/2511.07470)
- [State of NAM on Raspberry Pi (Nostatic blog)](https://blog.nostatic.org/2025/02/state-of-nam-on-raspberry-pi.html)

### PiPedal and LV2

- [PiPedal](https://rerdavies.github.io/pipedal/)
- [PiPedal system requirements](https://rerdavies.github.io/pipedal/SystemRequirements.html)
- [PiPedal audio configuration and published measurements](https://rerdavies.github.io/pipedal/Configuring.html)
- [PiPedal latency guide](https://rerdavies.github.io/pipedal/AudioLatency.html)
- [PiPedal now supports TONE3000](https://www.tone3000.com/blog/pipedal-now-supports-tone3000)
- [Crade-Cortex: community documentation of PiPedal's websocket protocol](https://github.com/EmilienAubert/Crade-Cortex)
- [ToobAmp](https://github.com/rerdavies/ToobAmp)

### Raspberry Pi 5, CM5, and USB

- [Raspberry Pi 5](https://www.raspberrypi.com/products/raspberry-pi-5/)
- [Raspberry Pi OTG whitepaper](https://pip-assets.raspberrypi.com/categories/685/app-notes-guides-whitepapers/documents/RP-009276-WP-1-Using%20OTG%20mode%20on%20Raspberry%20Pi%20SBCs)
- [Raspberry Pi USB Power Delivery whitepaper](https://pip-assets.raspberrypi.com/categories/685/app-notes-guides-whitepapers/documents/RP-009856-WP-1-USB%20Power%20delivery%20on%20Raspberry%20Pi%205.pdf)
- [Raspberry Pi: Using the I2S peripherals on Raspberry Pi SBCs](https://pip-assets.raspberrypi.com/categories/1259-audio-camera-and-display/documents/RP-009699-WP-1-Using%20the%20I2S%20peripherals%20on%20Raspberry%20Pi%20SBCs.pdf)
- [Raspberry Pi NVMe boot documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#nvme-ssd-boot)
- [Linux UAC2 gadget documentation](https://docs.kernel.org/usb/gadget-testing.html)
- [RASPIAUDIO: USB audio gadget on Pi 5](https://github.com/RASPIAUDIO/CamillaDSP/blob/main/docs/usb_audio_gadget_pi5.md)
- [Compute Module 5 datasheet](https://datasheets.raspberrypi.com/cm5/cm5-datasheet.pdf)
- [Compute Module 5 launch and pricing](https://www.raspberrypi.com/news/compute-module-5-on-sale-now/)
- [Designing a CM5 carrier: CM4 vs CM5 differences](https://raspipcb.com/tech-blog/designing-a-custom-carrier-board-for-raspberry-pi-compute-module-5-differences-between-cm4-and-cm5-in-pcb-designers-view/)
- [Elk Audio OS supported hardware](https://elk-audio.github.io/elk-docs/html/documents/supported_hw.html)

### Alternative embedded-Linux platforms

- [Rockchip RK3588 mainline status (CNX Software)](https://www.cnx-software.com/2024/12/21/rockchip-rk3588-mainline-linux-support-current-status-and-future-work-for-2025/)
- [Radxa CM4 RK3576 pricing (Liliputing)](https://liliputing.com/radxa-cm4-with-an-rk3576-processor-could-be-a-cheaper-alternative-to-the-radxa-cm5-with-rk3588/)
- [Orange Pi Compute Module 4 (CNX Software)](https://www.cnx-software.com/2023/08/29/orange-pi-compute-module-4-low-cost-rockchip-rk3566-alternative-to-raspberry-pi-cm4/)
- [NXP i.MX 8M Plus](https://www.nxp.com/products/i.MX8MPLUS)
- [linux-sunxi mainlining effort](https://linux-sunxi.org/Linux_mainlining_effort)

### Display and control MCU

- [TDO TL050WVH12CT-B1714A LCD module specification rev 4.0](references/datasheets/display/TL050WVH12CT-B1714A_LCD-Module-Spec_Rev4.0.pdf)
- [ESP-IDF: RGB interfaced LCD (ESP32-S3)](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/lcd/rgb_lcd.html)
- [ESP-FAQ: LCD](https://docs.espressif.com/projects/esp-faq/en/latest/software-framework/peripherals/lcd.html)
- [ESP32-S3-DevKitC-1 v1.0 user guide](https://docs.espressif.com/projects/esp-dev-kits/en/latest/esp32s3/esp32-s3-devkitc-1/user_guide_v1.0.html)
- [Waveshare ESP32-S3-Touch-LCD-7B 1024×600 flicker report](https://github.com/waveshareteam/ESP32-S3-Touch-LCD-7B/issues/4)
- [ESP-IDF: RGB interfaced LCD (ESP32-P4)](https://docs.espressif.com/projects/esp-idf/en/stable/esp32p4/api-reference/peripherals/lcd/rgb_lcd.html)
- [ESP-IDF issue #18333: RGB panel DMA stride on ESP32-P4](https://github.com/espressif/esp-idf/issues/18333)
- [STM32U5G9J-DK2](https://www.st.com/en/evaluation-tools/stm32u5g9j-dk2.html) and [LVGL port](https://github.com/lvgl/lv_port_stm32u5g9j-dk2)
- [Renesas RA8D1](https://www.renesas.com/en/products/ra8d1)
- [ST7796 on Raspberry Pi 5 forum thread](https://forums.raspberrypi.com/viewtopic.php?t=379622)
- [FocalTech FT6336 overlay forum thread](https://forums.raspberrypi.com/viewtopic.php?t=277581)

### Power control and power loss

- [MOD forum: Dwarf on/off switch](https://forum.mod.audio/t/dwarf-on-off-switch/6030) and [power disconnection countermeasure](https://forum.mod.audio/t/power-disconnection-countermeasure-for-mod-dwarf/11523)
- [Neural DSP Quad Cortex manual](https://neuraldsp.com/manual/quad-cortex) and [Quad Cortex power switch update](https://neuraldsp.com/news/a-power-switch-that-never-breaks-may-2020-quad-cortex-development-update)
- [HeadRush Pedalboard quickstart guide](https://cdn.inmusicbrands.com/HeadRush/pedalboard/Pedalboard-Quickstart-Guide-v2_1_3.pdf)
- [Kemper forum: powering down the Profiler](https://www.kemper-amps.com/forum/thread/11201-probably-a-silly-question-power-down-kemper/)
- [Raspberry Pi whitepaper: Making a more resilient file system](https://pip-assets.raspberrypi.com/categories/685-app-notes-guides-whitepapers/documents/RP-003610-WP-3-Making%20a%20more%20resilient%20file%20system.pdf)
- [embeddedTS: Filesystem corruption with SD cards](https://docs.embeddedts.com/Filesystem_Corruption_with_SD_cards)
- [Raspberry Pi documentation: bootloader configuration (`POWER_OFF_ON_HALT`, `PSU_MAX_CURRENT`)](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html)
- [Analog Devices LTC2955 push-button on/off controller](https://www.analog.com/en/products/ltc2955.html)

### USB recording comparisons

- [Line 6 forum: HX Stomp re-amp channels](https://line6.com/support/topic/65525-hx-stomp-reamp-question)
- [Roland support: GX-100 effects on DAW playback (DRY channel)](https://support.roland.com/hc/en-us/articles/4508370106651-GX-100-Applying-GX-100-effects-to-DAW-playback-audio-and-recording-as-a-new-track)
- [Roland support: GX-10 re-amping](https://support.roland.com/hc/en-us/articles/30332301042843-GX-10-Applying-Effects-to-Dry-Sound-after-Recording-Re-amping)
- [Valeton GP-200 product listing (6-in/4-out USB)](https://www.fullcompass.com/prod/644365-valeton-gp-200-gp-200-multi-effects-processor-black)
- [Neural Amp Modeler plugin](https://github.com/sdatkinson/NeuralAmpModelerPlugin)

### Audio hardware and external comparisons

- [Pisound documentation](https://blokas.io/pisound/docs/general-specifications/)
- [HiFiBerry DAC2 ADC Pro documentation](https://www.hifiberry.com/docs/data-sheets/datasheet-dac2-adc-pro/)
- [Dimehead NAM Player FAQ](https://www.dimehead.de/faq/)
- [Line 6 community Helix latency measurement](https://line6.com/support/topic/37727-helix-as-audio-interface-round-trip-latency/)
- [Neural DSP community Quad Cortex measurements](https://unity.neuraldsp.com/t/qc-numbers-report-latency-send-levels-etc/10291)

## 10. Changelog

| Date | Change |
|---|---|
| 2026-08-07 | v0.1 — Initial research draft and ADR set. |
| 2026-08-07 | v0.2 — Promoted to authoritative decision record; selected Pi 5 SBC for the final processor; selected a custom 40-pin Guitar-MFX board; made native USB gadget and dedicated bridge co-primary candidates; corrected power, mute, A2 variant, storage, measurement-ownership, and board bring-up rationale. |
| 2026-10-08 | v0.3 — Added learning goal and usage context; added NAM A2 2026 evidence; recorded the Pi 5 re-evaluation against CM5, Rockchip/Allwinner, i.MX8M Plus, and MCU-only (ADR-005); re-confirmed the ESP32-S3 UI split (ADR-004); amended ADR-007, ADR-009, and ADR-011; added ADR-013 (two ribbon-connected boards), ADR-014 (local UI via bridge daemon), ADR-015 (TDO 5-inch RGB display and ESP32-S3-WROOM-1-N16R8, STM32U5G9 fallback), ADR-016 (global gain calibration), and ADR-017 (outputs, modes, mute, volume). |
| 2026-10-13 | v0.4 — ADR-017 changed to separate analog Master and Phones potentiometers (encoder for navigation only). ADR-015 re-evaluated on cost against STM32U5G9, ESP32-P4, STM32H7B3, RA8D1, i.MX RT1170, and Allwinner T113-S3; ESP32-S3 retained. |
| 2026-10-14 | v0.5 — ADR-017 outputs changed to Main out L/MONO + R (6.35 mm) and 3.5 mm headphone out, following common commercial units; dedicated guitar-amplifier output and the three output modes withdrawn; amplifier use via main outputs with the cabinet IR switched off manually; ADR-009 amendment updated. Corrected ADR-005 and ADR-011: Pi 5 `I2S0` and `I2S1` share the same pins and cannot be used at the same time, so there is one usable I2S bus. Added ADR-018: PCM1863 + PCM5122 for rev A, CS4282P as rev B candidate. |
| 2026-10-14 | v0.6 — ADR-013 changed to three boards (audio, main, front-panel) after comparison with the two-shield plan, which is recorded as not selected; master-mute gate placed on the audio board with inputs pulled to muted; open decisions updated. |
| 2026-10-14 | v0.7 — Added ADR-019: 12 V DC input, soft power button with orderly shutdown, no Pi 5 hold-up, read-only root, and hardware power-fail mute, with a comparison of MOD Dwarf, Quad Cortex, HeadRush, and Kemper power control. Proposed analog rails of about ±12 V. Changelog rows put back in date order. |
| 2026-10-14 | v0.8 — Added ADR-020: native USB gadget first with a bridge fallback and a reserved lane-1 connector; three USB channels to the computer (processed L/R, dry) and two from it; re-amping through the unit deferred. Amended ADR-011. Recorded the Pi 5 I2S lane mapping. |
