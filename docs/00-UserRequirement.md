# Guitar-MFX User Requirements

| Field | Value |
|---|---|
| Document status | Baseline requirements |
| Revision | 0.8 |
| Date | 2026-10-15 |
| Project type | Personal DIY electric-guitar multi-effects processor; learning and research project |
| Primary user | Beginner to multi-effects processors and to audio electronics; experienced with PCB design, soldering, and 3D design |

## 1. Product goal

Guitar-MFX shall be a standalone electric-guitar multi-effects unit that is easy for a beginner to operate but capable of high-quality tones. It shall combine a broad, modifiable effects chain with capture-based neural amplifier modelling, cabinet simulation, local touchscreen control, headphone and powered-speaker outputs, and computer recording.

The final processor is a Raspberry Pi 5 Model B SBC. The project will design and fabricate three custom boards (the **Guitar-MFX boards**): an audio board and a main board connected to the Pi 5 40-pin header by ribbon cable, and a front-panel board carrying the ESP32-S3 behind the display. Together they integrate the product-specific mixed-signal audio, power, protected control I/O, the control surface, and panel connections. Commercial audio HATs remain development and measurement tools; they are not the final product audio board.

The project is a serious personal build, not currently a commercial product. Component selection may prioritise technical suitability and sound quality over the lowest cost. The user can source suggested components and can design, assemble, and solder PCBs and create a custom enclosure.

### 1.1 Learning goal and usage context

- The project is also a vehicle for improving the user's embedded-systems skills. The user designs the PCBs and mechanical parts; research and system architecture are prepared with assistance, and software may be written with AI assistance.
- When tone quality and learning value conflict, **tone takes priority**.
- The unit is played by one person with one electric guitar, alone in a room. Stage reliability features are not a priority; a low audible noise floor in a quiet room is.

## 2. Priorities

The project priorities, in order, are:

1. Convincing captured-amplifier tone and a useful library of effects.
2. Low, comfortable playing latency and glitch-free audio.
3. Beginner-friendly operation, presets, and clear feedback.
4. Computer integration, including bidirectional USB audio.
5. Hardware that can be extended and modified later.

Size is no longer a primary constraint. A larger enclosure is acceptable when it improves controls, cooling, connectors, or serviceability.

## 3. Functional requirements

### 3.1 Audio input and processing

| ID | Requirement | Priority |
|---|---|---|
| UR-AUD-001 | Accept a standard electric guitar through a 6.35 mm mono instrument input. | Must |
| UR-AUD-002 | Preserve the tone of passive pickups with a suitable high-impedance input stage; active pickups and hot humbuckers shall not clip under the supported gain settings. | Must |
| UR-AUD-003 | Provide capture-based neural amp modelling with the character and playing response of captured real amplifiers. Conventional algorithmic amp modelling alone is not sufficient. | Must |
| UR-AUD-004 | Provide cabinet/speaker simulation suitable for full-range PC speakers and headphones. | Must |
| UR-AUD-005 | Provide a broad, reorderable and extensible effects chain. Expected effect families include noise gate, compressor, boost/overdrive/distortion/fuzz, EQ, modulation, pitch effects, delay, reverb, looper, amp capture, and cabinet IR. | Must |
| UR-AUD-006 | Include an instrument tuner. | Must |
| UR-AUD-007 | Allow effects, models, parameters, and routing to be modified without redesigning the hardware. | Must |
| UR-AUD-008 | Support presets so a beginner can obtain a useful sound without constructing a chain from scratch. | Must |

"Broad effects support" means the platform shall support these effect families and allow new ones to be installed or developed. It does not require every effect to run simultaneously.

### 3.2 Outputs and monitoring

| ID | Requirement | Priority |
|---|---|---|
| UR-OUT-001 | Provide two main outputs, **Main out L/MONO** and **Main out R**, on 6.35 mm jacks, for powered speakers, studio monitors, mixers, and guitar amplifiers. L/MONO alone carries a mono mix. | Must |
| UR-OUT-002 | Drive headphones through a dedicated 3.5 mm headphone output with its own physical volume control. | Must |
| UR-OUT-003 | Provide a physical master-volume control that remains understandable and accessible during normal use. The confirmed control is an analog potentiometer in the main-output path, so it keeps working if software fails. The headphone output has its own potentiometer (UR-OUT-002). | Must |
| UR-OUT-004 | Prevent loud pops, uncontrolled noise, or full-scale output during boot, shutdown, crashes, cable connection, and preset changes. | Must |
| UR-OUT-005 | Driving a passive loudspeaker directly is not required. | Not required |
| UR-OUT-006 | A guitar amplifier is connected through the main outputs; there is no dedicated amplifier output. The user switches off the cabinet IR in the settings when feeding an amplifier, and also bypasses the amp model when feeding the amplifier's front input. Speakers, monitors, and mixers take the main outputs with the cabinet IR on. | Must |
| UR-OUT-007 | *Withdrawn 2026-10-14:* the three global output modes are replaced by the manual cabinet setting in UR-OUT-006, as on common commercial multi-effects units. | — |
| UR-OUT-008 | Inserting headphones silences the main outputs. | Should |

The outputs the user will actually use are headphones, powered PC speakers, a computer over USB, and a guitar amplifier. The layout follows common commercial multi-effects units: Main out L and R plus a headphone output. Powered PC speakers with a 3.5 mm or RCA input connect through an adapter cable. The headphone model is still to be confirmed (§6).

### 3.3 User interface and controls

| ID | Requirement | Priority |
|---|---|---|
| UR-UI-001 | Provide a local colour touchscreen interface. | Must |
| UR-UI-002 | Use the purchased TDO TL050WVH12CT-B1714A display: 5.0-inch, 800×480, IPS, parallel RGB interface, GT911 capacitive touch. This replaces the earlier plan to reuse the ST7796 3.5-inch display and FT6336 touch controller. | Must |
| UR-UI-003 | Present a beginner-oriented workflow: choose a preset, play, then expose deeper editing progressively. | Must |
| UR-UI-004 | Display the active preset, effect chain, amp/cab selection, tuner, input/output levels, bypass state, USB state, and system status. | Must |
| UR-UI-005 | Provide physical controls where touch alone would be unsafe or inconvenient, including power/shutdown and master volume. | Must |
| UR-UI-006 | Provide a connector for external foot control. | Must |
| UR-UI-007 | One rotary encoder with push action for navigation and parameter editing; three onboard footswitches; one external footswitch jack (dual momentary); one expression-pedal input. Footswitch functions are assigned in software. Counts confirmed by the user on 2026-10-15. | Must |
| UR-UI-010 | MIDI only over USB in v1, as a MIDI function of the USB gadget; no physical MIDI port. Confirmed by the user on 2026-10-15. | Must |
| UR-UI-008 | A reset or service control may be included when required for recovery or firmware development. | Should |
| UR-UI-009 | The local screen alone shall be sufficient for playing: choosing presets, enabling, disabling and reordering effects, editing parameters, tuning, viewing input/output levels, switching the cabinet IR off for a guitar amplifier, and viewing USB and system status. Deep setup — Wi-Fi configuration, plugin installation, NAM model and IR file management, and audio-device configuration — may remain in the PiPedal web interface as a maintenance path. | Must |

### 3.4 Computer and connectivity

| ID | Requirement | Priority |
|---|---|---|
| UR-CON-001 | Connect to a computer using a panel-mounted USB Type-C connector. | Must |
| UR-CON-002 | Appear to the computer as a bidirectional USB audio device. | Must |
| UR-CON-003 | Send the processed guitar signal to recording software without a separate external recorder. | Must |
| UR-CON-004 | Accept stereo computer playback and route it to the MFX headphone and powered-speaker outputs. | Must |
| UR-CON-007 | Send the dry guitar signal (after the calibrated PGA, before any processing) to the computer as a separate USB channel, so a recording can be re-processed later, for example with the NAM plugin on the computer. Confirmed by the user on 2026-10-14. | Must |
| UR-CON-008 | Re-amping through the unit (computer playback of a dry track into the effect-chain input) is deferred beyond v1; it needs no hardware change and may be added in software later. | Deferred |
| UR-CON-005 | Support loading and managing neural models, cabinet IRs, effects, and presets. | Must |
| UR-CON-006 | Wireless connectivity may be used for model downloads, updates, remote editing, and maintenance. | Should |

USB channel map for v1 (confirmed 2026-10-14): three channels to the computer (processed L/R and dry guitar) and two from the computer, 24-bit, 48 kHz. Host operating systems and recording applications remain open (§6, question 3). The native Pi 5 USB gadget is developed first; a dedicated USB audio bridge is the fallback if the native path fails its Phase 3 gate ([01-Hardware.md](01-Hardware.md) §7.3).

### 3.5 Power and mechanical requirements

| ID | Requirement | Priority |
|---|---|---|
| UR-PWR-001 | Operate from an external 12 V DC adapter (barrel jack, about 3 A) feeding the main-board power tree; the computer USB-C connection is for data, not main power. Input type confirmed by the user on 2026-10-14. | Must |
| UR-PWR-002 | Permit a USB data connection to a computer without back-powering either the Pi 5 or the host. | Must |
| UR-PWR-003 | Turn the unit on and off with a soft power button, as on the Quad Cortex: one press starts the unit; another press runs an orderly shutdown (mute, Linux shutdown, then power off). Confirmed by the user on 2026-10-14. | Must |
| UR-PWR-004 | Battery operation is not currently required. | Not required |
| UR-PWR-006 | The unit turns on by itself when 12 V is applied, so a pedalboard power strip can start it; the soft power button still turns it off and on. A jumper to disable automatic turn-on is considered for a later revision. Confirmed by the user on 2026-10-15. | Must |
| UR-PWR-005 | Pulling the DC plug while the unit runs must not damage the unit, corrupt the system or saved presets, or cause a loud pop; changes not yet saved may be lost. No hold-up supply is provided to finish a Linux shutdown after power loss. Confirmed by the user on 2026-10-14. | Must |
| UR-MEC-001 | The enclosure may be large enough for the Pi 5 SBC, the Guitar-MFX boards placed beside it, cooling, foot controls, robust connectors, and service access. | Must |
| UR-MEC-002 | Design and fabricate three custom Guitar-MFX boards — an audio board and a main board connected to the Raspberry Pi 5 40-pin header by ribbon cable, and a front-panel board behind the display — replacing the commercial audio HAT in the final product. The boards are not constrained to the HAT form factor. Board count confirmed by the user on 2026-10-14. | Must |
| UR-MEC-003 | The Pi 5 HDMI, USB-A, and Ethernet connectors may remain physically present but are not required as product-facing connections; Ethernet may be used during development. | Should |

## 4. Existing hardware and skills

The following items are already available and should be reused when technically appropriate:

- ESP32-S3 development board, 44-pin, ESP32-S3-WROOM-1-N16R8 module (16 MB flash, 8 MB octal PSRAM, 3.3 V); module identity to be confirmed with `esptool` on arrival.
- TDO TL050WVH12CT-B1714A 5.0-inch RGB display with GT911 touch (purchased).
- Electric guitar.
- Powered PC speakers.

The previously listed 3.5-inch ST7796 display and FT6336 touch controller are no longer planned for the product (UR-UI-002).

The user can solder and design PCBs and mechanical parts. The design does not need to be restricted to plug-together beginner modules, but development shall still proceed in measurable stages before committing to the custom mixed-signal Guitar-MFX boards.

## 5. Proposed engineering targets

These targets translate the user's quality goals into measurable criteria. They are proposed engineering goals, not measured project results. All project measurements shall be recorded only in [03-Measurements.md](03-Measurements.md).

| Metric | Proposed goal | Initial acceptance gate | Verification |
|---|---:|---:|---|
| Sample rate | 48 kHz fixed | 48 kHz | ALSA/device configuration |
| Converter format | 24-bit I/O where supported; 32-bit float DSP | No unintended 16-bit truncation | Device capability and captured test files |
| Analog round-trip latency | Aim for no more than 3 ms | No more than 5 ms for the first playable prototype | Physical loopback measurement |
| Audio dropouts | 0 xruns during a one-hour stress test | 0 xruns during a one-hour normal-chain test | ALSA/PiPedal counters |
| Guitar input impedance | At least 1 MOhm | At least 1 MOhm | Direct measurement |
| Input headroom | No clipping with the agreed passive/active pickup test set at a valid trim setting | Same | Calibrated test signal and guitar tests |
| Boot behaviour | Safe and quiet; UI shows progress | Audio output remains muted until valid | Repeated cold-boot test |
| Tone validation | Native captured-amp model plus calibrated gain and cabinet IR | Blind A/B judged usable by the owner | Level-matched recordings and playing test |

The 3 ms latency value is intentionally a goal rather than a locked hardware requirement. Published PiPedal measurements vary with the interface, buffer count, kernel, and processing chain, so the final threshold must be based on this project's measured hardware and playing tests.

### 5.1 External latency reference

These figures are vendor claims or community measurements, **not Guitar-MFX measurements**, and do not prove that the corresponding playing experience or test methods are equivalent. They provide context for the 5 ms prototype gate; the project result belongs in [03-Measurements.md](03-Measurements.md).

| Device | Category | Reported analog round-trip latency | Source type and qualification |
|---|---|---:|---|
| Dimehead NAM Player | Dedicated neural-model hardware player | 0.5 ms including AD/DA | [Vendor claim](https://www.dimehead.de/faq/); test method and configuration are not independently documented here |
| Line 6 Helix Floor | Mainstream floor modeller | 1.8 ms for one DSP path | [Line 6 community measurement](https://line6.com/support/topic/37727-helix-as-audio-interface-round-trip-latency/); not an official Line 6 specification |
| Neural DSP Quad Cortex | Neural floor modeller | About 1.75–1.8 ms for a simple path; about 4.4–4.5 ms when processing crosses to another row/core in the cited tests | [Neural DSP community measurements](https://unity.neuraldsp.com/t/qc-numbers-report-latency-send-levels-etc/10291); path-dependent and not an official specification |

## 6. Open requirements

The following questions must be resolved before the Guitar-MFX boards and enclosure are frozen:

1. *Closed 2026-10-08:* the ST7796 module is no longer used; the purchased TDO display uses a parallel RGB interface (UR-UI-002).
2. *Closed 2026-10-14:* the main outputs are 6.35 mm jacks at line level; the powered PC speakers connect through an adapter cable (UR-OUT-001).
3. Which computer operating systems and recording applications must be supported?
4. *Closed 2026-10-15:* three onboard footswitches plus the external footswitch jack (UR-UI-007).
5. *Closed 2026-10-15:* one expression-pedal input in the first hardware revision (UR-UI-007).
6. *Closed 2026-10-15:* USB MIDI only; no physical MIDI port in v1 (UR-UI-010).
7. *Closed 2026-10-14:* USB sends processed stereo plus a dry guitar channel; re-amping through the unit is deferred (UR-CON-007, UR-CON-008).
8. What maximum acceptable boot-to-audio time should be used after the Linux prototype is measured?
9. Should the final boot medium remain a high-endurance microSD card or use NVMe through the Pi 5 PCIe FFC connection after boot-time and reliability testing?
10. Does the native Pi 5 UAC2 gadget, developed first, pass its Phase 3 gate, or is the dedicated USB audio bridge fallback needed?
11. Which headphones will be used, and what is their impedance? The headphone stage is designed for up to 300 Ω until this is answered.
12. *Closed 2026-10-14:* there is no dedicated amplifier output, so no true-bypass path; all outputs default to silent (UR-OUT-006).

## 7. Requirement traceability

Hardware selections are defined in [01-Hardware.md](01-Hardware.md). System responsibilities, data flow, boot behaviour, and validation stages are defined in [02-Architecture.md](02-Architecture.md). Project measurements are owned by [03-Measurements.md](03-Measurements.md). Decision rationale and supporting evidence are owned by [90-DecisionRecord.md](90-DecisionRecord.md).
