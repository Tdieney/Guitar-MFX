# Guitar-MFX System Architecture

**Revision:** 0.8  
**Date:** 2026-10-14  
**Status:** Proposed architecture; measured validation is recorded in [03-Measurements.md](03-Measurements.md)

Related documents:

- [00-UserRequirement.md](00-UserRequirement.md)
- [01-Hardware.md](01-Hardware.md)
- [03-Measurements.md](03-Measurements.md)
- [90-DecisionRecord.md](90-DecisionRecord.md) — authoritative rationale and supporting sources

## 1. Architecture Summary

The proposed MFX separates real-time audio processing from the user interface:

- A Raspberry Pi 5 Model B is both the development and final audio processor. It runs the audio engine, NAM A2 neural amp models, conventional effects, preset storage, and the software side of USB audio.
- Three custom Guitar-MFX boards replace the commercial audio HAT in the final product. The audio and main boards connect to the Pi 5 40-pin header by ribbon cable and sit beside the Pi; the front-panel board sits behind the display. The **audio board** owns the guitar input, codec, analog outputs, local low-noise regulation, output mute relays, and their mute gate. The **main board** owns the product power tree, external control jacks, panel USB-C, and signal routing between the boards. The **front-panel board** owns the ESP32-S3, display and touch connectors, backlight driver, I/O expander, encoder, and power button. See [01-Hardware.md](01-Hardware.md) §2.1.
- An ESP32-S3-WROOM-1-N16R8 renders the complete local UI on a 5.0-inch 800×480 RGB touchscreen and reads the encoder, footswitches, and expression pedal. It also supervises power and mute.
- A bridge daemon on the Pi translates between PiPedal and the ESP32-S3 (§5.2).
- Commercial audio HATs are used only for development measurements and codec/interface comparison before the audio board is committed.

This division keeps display rendering and touch activity away from the real-time audio path. Audio must never be transported through the ESP32-S3 or over the control UART.

```text
 Guitar
   |
   v
Audio board: high-Z input -> PGA -> codec/ADC
                                      |  I2S + I2C (ribbon)
                                      v
                              Raspberry Pi 5
                         PiPedal + NAM A2-Full + LV2
                           |          |          |
                           |          |          +<== USB audio candidate == Computer / DAW
                           |          |
                           |   bridge daemon <-- UART --> ESP32-S3 -> 5" RGB display / GT911 touch
                           |                                 |        encoder / footswitches
                           |                                 +--> master mute (relay coils)
Audio board: codec/DAC -> master pot -> line driver --[relay]--> Main out L/MONO, R (6.35 mm)
                  |                                              speakers, monitors, mixer, guitar amp
                  +-> phones pot -> HP amp --[relay]--> headphones (3.5 mm)
```

The USB path in this diagram is the native Pi 5 UAC2 gadget, developed first, with a dedicated USB audio bridge as the fallback (§4.4). Neither is validated for Guitar-MFX.

## 2. Responsibility Boundaries

| Domain | Owner | Responsibilities |
|---|---|---|
| Real-time audio | Raspberry Pi 5 | Codec I/O, effect graph, NAM inference, tuner input, metering, and USB audio routing |
| Presets and models | Raspberry Pi 5 | Save/load presets, model library, parameter validation, atomic updates, and version migration |
| Analog audio hardware | Guitar-MFX audio board | High-Z input, PGA, codec and clocks, output stages, headphone volume, default-muted relays and master-mute gate |
| Power and external I/O | Guitar-MFX main board | Power tree, protected external footswitch and expression jacks, panel USB-C, signal routing between the boards |
| Control-surface hardware | Guitar-MFX front-panel board | ESP32-S3 module, display and touch connectors, backlight driver, local 3.3 V rail, I/O expander, encoder, power button |
| PiPedal ↔ UI translation | Bridge daemon on Pi 5 | Talks to PiPedal over its websocket API and to the ESP32-S3 over UART; flattens PiPedal state into what the UI draws; applies the cabinet, output-level, and mono settings |
| User interface | ESP32-S3 | Draw the complete local UI, read touch/encoder/footswitches/expression, send user actions, and show cached state |
| Safety control | ESP32-S3 plus audio-board relays | Keep outputs muted during boot/fault/shutdown and supervise Linux heartbeat and power state |
| Computer connection | Pi 5 native OTG path first; dedicated bridge as fallback | Provide full-duplex, class-compliant USB audio once P3-GATE validates the path |
| Measurement record | Documentation process | Store every project-measured result in [03-Measurements.md](03-Measurements.md), including failures |

Linux is the source of truth for the active preset and DSP parameter values. The ESP32-S3 keeps a display cache, but it must resynchronize after reconnection rather than assuming that cached state is current.

## 3. Audio Signal Architecture

### 3.1 Local playing path

The default mono-guitar signal graph is:

```text
Input protection and RF filter
  -> high-Z buffer / PGA (global instrument calibration)
  -> ADC
  -> input trim (per preset) and noise gate
  -> pre-amplifier effects
  -> NAM A2 amp capture            (user may bypass when feeding an amp's front input)
  -> cabinet IR, if not contained in the selected capture
                                   (switched off in settings when feeding a guitar amp)
  -> post-amplifier effects
  -> limiter
  -> DAC
  -> analog master pot -> Main out L/MONO and R
  -> analog phones pot -> headphone output
```

There are no output modes. When the main outputs feed a guitar amplifier, the user switches the cabinet IR off with a global setting and, for the amplifier's front input, bypasses the amp model block like any other effect. A global Line / Instrument level setting and an optional mono sum on L/MONO apply to the main outputs ([01-Hardware.md](01-Hardware.md) §5.3–5.4).

The effect order must be editable. A beginner mode should initially present a small number of useful controls while an advanced view exposes the complete graph and detailed parameters.

Recommended initial effect set:

- Chromatic tuner.
- Noise gate.
- Compressor.
- Boost/overdrive/distortion.
- NAM A2 amp capture.
- Cabinet impulse-response loader.
- Parametric or graphic equalizer.
- Chorus/flanger/phaser.
- Delay.
- Reverb.
- Output limiter.

### 3.2 Format and levels

- Fixed system sample rate: 48 kHz for the first implementation.
- Physical converter format: 24-bit where supported.
- Internal processing: 32-bit floating point.
- Guitar input: at least 1 MOhm with repeatable input-gain calibration. The analog PGA is a global calibration for the instrument and is not stored per preset.
- Presets: store digital input trim, effect parameters, and capture/IR references. The cabinet-off, output-level, and mono settings are global, not per preset. Master and phones volume are analog potentiometers outside software control.
- Preset switching: use mute, ramping, or crossfade logic to prevent clicks and dangerous level jumps.

These are design targets or rules, not measured results.

### 3.3 Tuner path

The tuner reads a clean branch immediately after input calibration, before time-varying effects. When tuner-mute is enabled, the output graph is muted but input analysis continues. Pitch, note, and confidence data are sent to the ESP32-S3 at a modest update rate; raw audio is not sent to the UI controller.

### 3.4 NAM A2-Full and A2-Lite

According to the **TONE3000 vendor/developer guide**, one A2 model file contains both A2-Full and A2-Lite, and the host selects which width to execute. A2-Full is intended for systems with sufficient processing headroom; A2-Lite is the reduced-width form intended for processing-constrained embedded or microcontroller-class hardware. The same guide reports that A2-Full is approximately 30–40% more CPU-efficient than A1-Standard. These are source claims, not Guitar-MFX measurements. See the [NAM A2 guide](https://www.tone3000.com/guides/nam-a2-the-complete-guide).

The Pi 5 architecture therefore uses A2-Full as its baseline. A2-Lite is not a latency-tuning lever for Pi 5. If the deferred hybrid architecture is ever evaluated, an MCU may run A2-Lite while Linux runs A2-Full from the same model library; no separate conversion or asset pipeline is required.

## 4. Bidirectional USB Audio

### 4.1 Intended USB interface

The target computer-facing interface is a class-compliant USB Audio Class 2 device:

- Two playback channels from the computer to the MFX.
- Three capture channels from the MFX to the computer: processed L/R and the dry guitar signal (after the calibrated PGA, before any processing). Confirmed by the user on 2026-10-14 (UR-CON-007).
- 24-bit, 48 kHz initial format.
- Class-compliant operation without a project-specific host driver where the host OS supports UAC2; behavior must be tested on the intended Windows release and DAW.

Initial routing is:

```text
Computer playback L/R -> MFX USB receive -> monitor mixer -> local outputs
Processed MFX L/R     -> MFX USB transmit -> computer recording input
Dry guitar            -> MFX USB transmit -> computer recording input
```

The dry track can be re-processed on the computer with the NAM plugin and the same model and IR files. Re-amping through the unit, where computer playback replaces the guitar input of the effect chain, is deferred beyond v1 (UR-CON-008); it needs no hardware change.

### 4.2 Proposed native Pi 5 physical path — not solved

Raspberry Pi's **vendor OTG whitepaper** states that Pi 5 gadget mode uses the native SoC OTG peripheral exposed through the Pi 5 USB-C power connector; the RP1-connected host ports do not support OTG. The Pi 5 Model B USB-C connector carries USB 2.0 data only, its USB 3.x SuperSpeed pins are not wired, and the USB 2.0 data pair is not exposed on the 40-pin header.

For Pi 5 Model B, the proposed configuration places the overlay under `[all]` in `/boot/firmware/config.txt`, not under `[cm5]`:

```ini
[all]
dtoverlay=dwc2,dr_mode=peripheral
```

The proposed physical arrangement is **Proposed, bench proof required**:

- A panel-mounted USB-C receptacle on the Guitar-MFX main board routes to the Pi 5's own USB-C receptacle through a short internal cable or interposer.
- USB 2.0 D+/D− and the required configuration-channel signals pass through that path.
- Host VBUS is isolated so host power cannot reach the Pi 5 rail and the product cannot back-power the host.
- The main board powers the Pi 5 through the 40-pin header 5 V/GND pins.

This does not yet prove UR-PWR-002. USB-C role detection depends on configuration-channel and VBUS-presence state, so VBUS isolation may prevent or destabilize device role. A hand-made cable or adapter must demonstrate enumeration, disconnect/reconnect, host sleep, cold boot, warm reboot, VBUS isolation, and no back-feed before PCB layout begins. A legacy-OTG-style USB-C cable/adapter may be required during bench investigation if the normal CC arrangement blocks role detection.

Header power bypasses Pi 5 USB-PD negotiation and must not be assumed to retain the USB-C input's inrush or protection behavior. The main board must provide measured inrush limiting, sequencing, protection, bulk capacitance, current delivery, and brownout behavior. These statements define risks and test obligations; they do not validate the proposed solution.

### 4.3 USB clocking and routing

The codec audio clock and host USB transfer timing are separate timing domains. The native design should first evaluate a stable audio master clock with asynchronous USB feedback or controlled rate matching. Sample-rate conversion is a fallback if long-duration measurements show unavoidable drift, underruns, or overruns; it is not assumed to be mandatory.

Linux can expose UAC2 through ConfigFS and the `usb_f_uac2` gadget function, including an asynchronous mode with feedback (`c_sync=async`, `fb_max`). A Guitar-MFX routing service would connect USB streams to the local audio graph and expose routing state to PiPedal and the ESP32-S3. PiPedal opens one ALSA device, so the gadget card has to be bridged into the graph without disturbing the real-time path: the service runs below the audio threads' priority, preferably on a separate core, and uses feedback so that the host follows the codec clock before any resampling is considered. Linux UAC2 framework availability does not prove this application's full-duplex behavior.

### 4.4 Native first, bridge fallback

Selected by the user on 2026-10-14 ([90-DecisionRecord.md](90-DecisionRecord.md) ADR-020). The native path is developed first; the bridge is activated only if the native path fails P3-GATE, and a spare connector keeps that possible without an audio-board respin ([01-Hardware.md](01-Hardware.md) §7.5). The two paths are:

1. **Native Pi 5 UAC2 gadget:** uses the Pi 5 USB-C OTG data path and Linux gadget framework.
2. **Dedicated class-compliant USB audio bridge:** adds a controller on a Guitar-MFX board and a digital-audio link to Pi 5. Pi 5 has only one usable I2S bus at a time (`I2S0` and `I2S1` share the same pins), so the bridge must share the codec's clocks on an extra data lane or use another link ([01-Hardware.md](01-Hardware.md) §7.4).

If the fallback is activated, a comparison documents bill-of-materials cost, measured added latency, Pi 5 lane availability alongside the codec, host-driver behavior, clocking/rate-matching needs, firmware/tool dependencies, component availability, and development risk, recorded in [03-Measurements.md](03-Measurements.md) before the bridge daughterboard is designed.

USB validation for either candidate covers:

- Simultaneous playback and recording for at least four hours.
- No audible glitches, channel swaps, or growing clock drift.
- Reconnection after computer sleep, cable removal, and MFX reboot.
- Operation with the intended Windows DAW and at least one Linux host.
- Measured end-to-end monitoring and recording latency.
- Demonstrated power isolation and absence of back-feed.

## 5. Control and User-Interface Architecture

### 5.1 ESP32-S3 peripherals and UI scope

The ESP32-S3-WROOM-1-N16R8 owns:

- TDO TL050WVH12CT-B1714A 5.0-inch 800×480 RGB display, driven from two framebuffers in PSRAM with bounce buffering ([01-Hardware.md](01-Hardware.md) §6.3), including the display power-on/off sequence (§6).
- GT911 capacitive touch and an I2C I/O expander.
- One navigation encoder, footswitch inputs, expression input, and debounce logic.
- Master-mute GPIO, per-output mute enables, headphone-insert sensing, and Linux heartbeat supervision.
- Local boot, fault, and shutdown screens.

The ESP32-S3 does not use Wi-Fi and does not write its flash during normal play, because both disturb RGB display refresh on this chip.

**UI scope (UR-UI-009):** the local UI alone is sufficient for playing — presets, effect enable/order, parameters, tuner, levels, cabinet-off and output-level settings, USB and system status. Wi-Fi configuration, plugin installation, NAM model and IR file management, and audio-device configuration stay in the PiPedal web interface as a maintenance path.

### 5.2 Control link and bridge daemon

PiPedal's websocket API is **internal and undocumented**; the only description is a community reverse-engineering of version 2.0.110, which warns that it can change between releases ([Crade-Cortex](https://github.com/EmilienAubert/Crade-Cortex)). The bridge daemon on the Pi is therefore the only component that talks to PiPedal. When PiPedal changes, only the daemon is updated; the ESP32-S3 firmware and UART protocol stay the same. Pin the PiPedal version in the system image and re-verify the daemon after every PiPedal update.

The daemon flattens PiPedal state into exactly what the UI draws: preset list, current chain (order, names, enabled state), parameter descriptors for the selected block (name, unit, range, value, enum labels), tuner, meters, and status.

Use a dedicated 3.3 V UART between the ESP32-S3 and Pi 5, initially at 921600 baud. The protocol should use framed binary messages with:

```text
protocol version | message type | payload length | sequence number | payload | CRC
```

Required message groups include:

- Full-state synchronization and heartbeat.
- Preset list, select, save, rename, and delete requests.
- Effect enable/bypass and parameter updates.
- Meter, tuner, CPU-load, temperature, and fault status.
- Footswitch and expression-pedal events.
- Effect reorder requests and preset save.
- Cabinet-off, output-level (Line / Instrument), and mono settings; headphone-insert state.
- Display/UI software compatibility information.

The expected size is about 8–10 message types.

Commands must be acknowledged or rejected with a reason. Parameters must be identified by stable IDs rather than screen positions. Repeated absolute-value commands should be safe to retry, and both sides must detect a protocol-version mismatch.

### 5.3 Physical controls

Touch is suitable for setup and editing, but live performance needs controls that can be used without precise visual attention. The initial proposal includes:

- At least three external or panel-mounted footswitch functions: previous preset, next preset, and assignable/tuner.
- One rotary encoder with push action for navigation and parameter editing.
- Two analog potentiometers on the audio board: **Master** (both main outputs) and **Phones** (headphones). They are in the signal path and work without software ([01-Hardware.md](01-Hardware.md) §5.6).
- The Pi 5 is the only I2C master of the codec; the ESP32-S3 never writes to the codec.
- A power control and accessible recovery/reset control.

External footswitch connectors require ESD protection, filtering, and a clearly documented wiring standard. An expression-pedal input can be added after its connector type and resistance convention are selected.

## 6. Boot, Mute, Fault, and Shutdown Behavior

The output must default to muted in hardware. The safe-state sequence is:

```text
OFF: only the push-button controller is powered
  -> UI_BOOT: button press; ESP32-S3 starts; output remains muted; Pi 5 still unpowered
  -> LINUX_BOOT: ESP32-S3 closes the Pi 5 load switch; UI shows progress; heartbeat awaited
  -> AUDIO_STARTING: codec and graph initialize; levels ramp from silence
  -> READY: output mute releases only after stable audio is confirmed
  -> FAULT: any lost heartbeat or audio failure immediately remutes output
  -> SHUTDOWN: button press; mute, UART shutdown command, Linux poweroff, wait for the
     Pi 5 header 3.3 V to fall, open the Pi 5 load switch, display power-off sequence,
     controller kill input turns the unit off
  -> POWER_FAIL: 12 V input lost; hardware mutes and opens the Pi 5 load switch;
     ESP32-S3 completes the display power-off sequence on hold-up energy
```

Mute is implemented by two relays at the output jacks, after the output drivers, muted when unpowered and driven through one direct master-mute GPIO ([01-Hardware.md](01-Hardware.md) §5.5). This matters for crashes: when Linux hangs, the ALSA DMA engine typically keeps replaying the last audio buffer, producing a loud repeating buzz. The ESP32-S3 detects the lost heartbeat and closes the relays.

Level safety: at `AUDIO_STARTING` the digital output level of the audio graph starts at silence and ramps up before the relays release. The analog master and phones potentiometers stay wherever the user left them.

Display sequencing: the ESP32-S3 follows the panel's power-on order (VDD, DISP, signals, then backlight) and, at `SHUTDOWN`, turns the backlight off, drives DISP low, and keeps DCLK running for at least 100 ms before stopping signals and removing display power ([01-Hardware.md](01-Hardware.md) §6.1).

A normally-muted output is preferable to traditional guitar true bypass for headphones and full-range speakers: an unprocessed guitar signal would be very quiet and would omit the required amplifier/cabinet processing. There is no dedicated amplifier output and therefore no true-bypass path; all outputs default to silent.

Normal shutdown is orderly ([01-Hardware.md](01-Hardware.md) §8.3). Abrupt power loss is tolerated rather than prevented: there is no hold-up for the Pi 5, Linux runs from a read-only root, and presets are written atomically (§9). A long press of the power button forces power off if the ESP32-S3 or the Pi 5 hangs. If the Pi 5 does not halt within the shutdown timeout, the ESP32-S3 removes its power anyway, which is treated like a power loss.

## 7. Real-Time Performance Architecture

### 7.1 Latency gates

The owner wants an immediate playing feel, but final latency has not been measured on this hardware. Therefore:

- Aspirational end-to-end local monitoring target: no more than 3 ms.
- First prototype acceptance gate: no more than 5 ms while running one NAM A2-Full model, cabinet processing, and a representative effect chain.
- The final choice must also pass blind playing comparisons; a numerically lower latency is not useful if the system is unstable or sounds worse.

Latency must be measured from analog input to analog output. Driver buffer settings alone are not an end-to-end measurement. Results belong only in [03-Measurements.md](03-Measurements.md).

### 7.2 Linux real-time policy

The baseline evaluation uses a headless, audio-dedicated Linux image and PREEMPT_RT where supported by the selected PiPedal/Raspberry Pi OS combination. This approach has not yet passed the project latency gate.

- Use a 64-bit Raspberry Pi OS image compatible with PiPedal and its real-time guidance.
- Disable unused background services, radios, and periodic jobs only after their purpose is understood.
- Use the performance CPU governor during audio operation.
- Start with conservative audio buffers, then reduce buffer size and period count while measuring latency and xruns.
- Apply CPU affinity, IRQ affinity, and core isolation only when profiling shows a benefit.
- Monitor throttling and keep the Pi 5 actively cooled during bench tests.

Buffer sizes should be benchmarked rather than fixed prematurely. If PREEMPT_RT cannot meet the gate, the risk table in §12 defines alternative real-time approaches for investigation; none is accepted architecture at this revision.

### 7.3 Stability and capacity gates

- Zero xruns during a one-hour development stress test.
- Zero xruns during a four-hour release-candidate playing/USB test.
- Maintain at least 20% real-time deadline margin in the approved maximum effect chain.
- No sustained thermal throttling at the worst tested ambient temperature.
- Preset changes and UI activity must not disturb the audio deadline.

## 8. Software Components

### 8.1 Raspberry Pi 5

| Component | Role |
|---|---|
| Raspberry Pi OS 64-bit | Baseline operating system for evaluation |
| PiPedal 2.x | Audio graph, LV2 hosting, presets, and device management |
| NAM A2 plugin/runtime | A2-Full captured amplifier and cabinet processing on Pi 5 |
| LV2 plugins | Conventional effects, EQ, delay, reverb, dynamics, and utility blocks |
| Guitar-MFX bridge daemon | Sole client of the undocumented PiPedal websocket API; ESP32 protocol, flattened UI state, cabinet and output-level settings, mute/heartbeat, and USB routing coordination |
| Native UAC2 gadget configuration or bridge service | Candidate class-compliant computer audio path |
| Watchdog and startup services | Ordered startup, recovery, logging, and safe shutdown |

Software versions must be pinned for reproducible images. Model and plugin licenses must be checked before they are redistributed in a finished device image.

### 8.2 ESP32-S3

The controller firmware uses ESP-IDF 5.2.0. Its modules should be separated into:

- Display and touch drivers: ESP-IDF RGB panel driver with two PSRAM framebuffers and bounce buffering; GT911 over I2C; display power sequencing.
- UI application/state cache.
- Physical input service.
- Framed transport and protocol client.
- Power, mute, heartbeat, and fault supervisor.
- Firmware update and recovery support.

LVGL is a suitable candidate if it meets display memory and refresh-rate tests. Existing unrelated UI code should be treated as reference material only; hardware assumptions and application logic must be reviewed before reuse. The reference code in `ui/` targets 800×480 but a 7-inch-class layout; on the 5.0-inch panel (about 187 PPI) its minimum text size and touch targets must be enlarged.

## 9. Storage, Presets, Models, and Updates

Pi 5 Model B has no eMMC. Phase 0 uses a high-endurance microSD card as the specified boot medium. The final boot medium remains an open decision between:

- **High-endurance microSD:** simplest mechanical and electrical path, but endurance and power-loss behavior require testing.
- **NVMe through the Pi 5 PCIe FFC connector:** a vendor-supported boot option that may improve storage behavior, but routing PCIe on a Guitar-MFX board or adding an M.2/FFC assembly significantly increases signal-integrity, power, cooling, layout, and enclosure difficulty.

The final choice follows measured boot time, repeated-write behavior, forced-power-loss recovery, current consumption, thermal impact, and serviceability. Results belong in [03-Measurements.md](03-Measurements.md).

For either medium:

- Run the root filesystem read-only, with runtime writes kept in RAM (an overlay, as Raspberry Pi OS provides), so an unplanned power loss cannot corrupt the system. Raspberry Pi's resilient-filesystem whitepaper lists the page cache, in-flight writes, and SD card internal operations as the causes of power-loss corruption.
- Keep user presets, PiPedal settings, downloaded models, and IRs on a separate writable data partition that is backed up. How PiPedal stores its data, and how to place it on that partition, is checked in Phase 1.
- Write presets atomically so power loss cannot leave a partially written active preset.
- Validate imported NAM and IR files before making them selectable.
- Provide Wi-Fi transfer through PiPedal or the bridge service during setup, with an option to disable Wi-Fi during performance.
- Keep a known-good factory preset that can load even when a user model is missing or corrupt.
- Separate system updates from user content and provide rollback or a recoverable image procedure.

Raspberry Pi documents NVMe boot for Pi 5 through the PCIe FFC interface; this is **vendor capability**, not a measured Guitar-MFX reliability result. See the [Raspberry Pi NVMe documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#nvme-ssd-boot).

## 10. Development Phases and Verification Gates

Every gate result, including failures and retests, is recorded in [03-Measurements.md](03-Measurements.md).

### Phase 0 — Audio feasibility and comparison bench

Use Raspberry Pi 5, Pisound, HiFiBerry DAC2 ADC Pro, active cooling, a protected high-Z input buffer, powered speakers/headphones, and measurement equipment. Measure both HAT paths before choosing a production codec shortlist.

**P0-GATE:** Both HAT candidates have recorded latency, noise, THD+N, frequency-response, gain/headroom, driver stability, clock/recovery, and xrun results under documented conditions. The owner has completed a level-matched tone evaluation. The HiFiBerry DAC2 ADC Pro measurements also qualify the rev A codec pair (PCM1863 + PCM5122) and include: codec-as-clock-master operation on Pi 5; low-latency filter settings; and the noise-floor comparison (input shorted versus guitar connected, not played) that decides whether CS4282P is evaluated for rev B ([01-Hardware.md](01-Hardware.md) §11.2).

### Phase 1 — Complete audio engine

Build representative effect chains, tuner, preset switching, cabinet IR handling, output limiting, and real-time tuning around NAM A2-Full.

Before judging playing feel, optionally acquire or borrow a low-cost deterministic MCU audio board solely as a perceptual low-latency reference. This is optional de-risking, not a Phase 1 requirement or production choice; without it, the owner judgement has no controlled low-latency hardware reference.

**P1-GATE:** Meet the first 5 ms analog round-trip target and one-hour zero-xrun stability target with the representative chain, or document the failure and the next experiment. Record the owner latency/tone comparison and whether the optional reference was used.

### Phase 2 — ESP32-S3 control surface

Build the display bring-up adapter ([01-Hardware.md](01-Hardware.md) §6.4) and pass the display gate first. This step does not depend on the Pi 5 and may run in parallel with Phase 0–1. Then implement the bridge daemon and framed protocol, add footswitches and the navigation encoder, and test UI recovery when Linux restarts.

**Display gate (inside P2):** stable image at DCLK ≥ 23 MHz with double PSRAM framebuffers and bounce buffering; LVGL frame rate recorded; GT911 touch working; display power sequences implemented. If it fails, evaluate the STM32U5G9 fallback before continuing Phase 2.

**P2-GATE:** display gate passed; UI and preset activity cause no measured audio xruns; mute supervision works during forced Linux failures; output-level ramp at startup is verified; control recovery and protocol mismatch behavior are recorded.

### Phase 3 — USB and header-power proof on Raspberry Pi 5

Use the same Pi 5 from Phase 0. First build a hand-made USB-C cable/interposer and current-limited header-power setup. Test the VBUS-isolated native gadget proposal before any main-board layout. Evaluate a dedicated USB audio bridge only if the native path fails this gate (§4.4).

**P3-GATE:** Before layout, demonstrate and record:

- Reliable Pi 5 device-role entry with `dwc2` configured under `[all]`.
- USB 2.0 enumeration through the panel/internal-cable concept with isolated VBUS.
- Valid CC/VBUS-presence behavior, including whether a legacy-OTG-style cable is required.
- No back-feed to Pi 5 or host across off, boot, disconnect, and fault states.
- Measured header-power inrush, sequencing, rail droop/ripple, current, protection response, and brownout behavior.
- Four-hour full-duplex native UAC2 run (three channels to the host, two from it) with clock-drift and reconnect results, and no xruns on the local playing path while the approved NAM A2-Full chain runs.
- CPU load of the USB routing service.
- If the native path fails: equivalent host, latency, digital-interface, and stability evidence for the bridge fallback, and the comparison in §4.4.

If neither candidate passes, board layout remains blocked and the failure is recorded rather than reclassified as validated.

### Phase 4 — Guitar-MFX boards design and bring-up

Design the audio board after P0-GATE; it can be connected to a Pi 5 by ribbon, powered from a bench supply, and measured before the main board exists. Design the main board after P3-GATE. The front-panel board can be powered from a bench supply and brought up against a Pi 5 or a PC over UART before the main board exists. Each schematic and layout shall include a documented test-point list with expected voltage, waveform, tolerance, safe probe method, and test state.

**P4-GATE:** Bring up each first article in this order:

1. Verify continuity, shorts, protection orientation, and resistance-to-ground before power.
2. Power each bare board without Pi 5; verify every rail, sequencing, current limit, ripple, and shutdown path.
3. Verify the default-muted state and force every mute/fault transition.
4. Fit dummy loads, then the Pi 5 only after rail behavior is safe.
5. Verify codec clocks and digital audio before connecting a guitar or headphones.
6. Measure the input, codec, line output, and headphone stages against Phase 0 bench references.
7. Repeat latency, noise, headroom, thermal, power, xrun, and USB tests on the integrated assembly.

The first article is expected to need at least one PCB respin. The project schedule and component purchasing plan shall assume it; a first-pass board is not treated as production-ready.

### Phase 5 — Final assembly and enclosure

Integrate the Pi 5, revised audio, main, and front-panel boards, display, controls, panel connectors, selected storage, cooling, and enclosure.

**P5-GATE:** The enclosed device repeats the approved audio, USB, control, storage, power, thermal, connector, and safe-boot/shutdown tests without regression. Any EMC work at this stage is a pre-compliance check unless performed by a qualified test facility.

### Phase 6 — Optional hybrid accelerator

Consider an MCU audio front end only if the measured and owner-evaluated Pi 5 system still fails the local-playing-latency gate after buffer/period, effect-chain, real-time scheduling, and thermal work. The owner judgement must state whether it used the optional deterministic MCU perceptual reference from Phase 1.

**P6-GATE:** Before adopting a hybrid architecture, record a controlled playing comparison, the measured latency benefit, clock/link stability, mixing delay compensation, shared A2 model behavior, BOM impact, and the added firmware/maintenance cost. Hybrid remains deferred unless this gate justifies it.

## 11. Architecture Decisions

| Decision | Status | Rationale |
|---|---|---|
| Linux SBC for primary DSP | Existing project decision; prototype validation required | Best fit for PiPedal, LV2, NAM A2, model storage, and networking |
| Raspberry Pi 5 Model B for development and final processor | Selected by user; re-confirmed 2026-10 against CM5, Rockchip/Allwinner, i.MX8M Plus, and MCU-only; performance gates remain open | Keeps one processor/platform through development and final assembly; core MFX software already runs on it |
| Three Guitar-MFX boards (audio, main, front-panel), not HAT form factor; audio and main ribbon-connected beside the Pi | Selected by user 2026-10-14; first articles not yet designed | Separates analog from switching power; keeps the ESP32-S3 next to the display; audio board can be measured before the main board exists. Two-shield plan not selected (ADR-013) |
| ESP32-S3-WROOM-1-N16R8 for UI/control only | Selected; conditional on the display gate | Display and touch hardware are well supported on ESP32-S3; isolates UI from audio deadlines; fallback STM32U5G9 |
| TDO 5.0-inch 800×480 IPS RGB display with GT911 | Purchased | Replaces the ST7796/FT6336 reuse plan |
| Complete local UI via bridge daemon; web UI kept for maintenance | Selected | Contains the undocumented PiPedal API in one Pi-side component |
| Global analog PGA calibration; digital per-preset model level | Selected; not yet measured | Repeatable NAM drive level without PGA steps on preset change |
| Main out L/MONO + R (6.35 mm) and headphone out (3.5 mm); guitar amplifier via main outputs with cabinet switched off manually | Selected 2026-10-14 | Follows common commercial multi-effects units; one stereo mix keeps the codec at 2 DAC channels |
| Relays at jacks with direct master-mute GPIO; separate analog Master and Phones potentiometers | Selected; circuits not yet measured | Stops driver transients; safety path and volume both independent of the I/O expander and of Linux |
| 12 V DC barrel-jack input, about 3 A | Selected by user 2026-10-14 | Common, inexpensive adapters; simple buck to the Pi 5 rail |
| Soft power button with orderly shutdown; long press forces off | Selected by user 2026-10-14; circuit not yet designed | As on the Quad Cortex; Linux shuts down cleanly in normal use |
| No hold-up for the Pi 5; read-only root, atomic preset writes, hardware power-fail mute | Selected by user 2026-10-14; not yet measured | Pulling the plug stays safe without supercapacitors |
| PiPedal 2.x as host | Existing project decision; prototype validation required | Pedal-oriented host with LV2 and NAM A2 support |
| 48 kHz fixed initial sample rate | Existing engineering decision; measurement gate remains | Quality/CPU/latency compromise and simpler system clocking |
| Pisound plus HiFiBerry comparison | HiFiBerry DAC2 ADC Pro required (it carries the rev A codec pair); Pisound now an optional second reference | A measured reference for the exact chips used on the audio board |
| Codec for audio-board rev A: PCM1863 + PCM5122; CS4282P as rev B candidate | Selected 2026-10-14; measurement-gated | Hand-solderable TSSOP, PGA on chip, drivers and overlay exist, measurable on a HAT first |
| Calibrated 1 MOhm or higher input | Existing design requirement; not yet measured | Necessary for passive pickups and repeatable model drive level |
| Normally-muted output safety | Existing design requirement; not yet measured | More useful than raw true bypass with powered speakers/headphones |
| VBUS-isolated Pi 5 native-gadget cable/interposer | **Proposed, bench proof required** | May free Pi 5 USB-C for data while header-powered, but CC/VBUS role behavior and protection are unresolved |
| Dedicated class-compliant USB audio bridge | Fallback, selected by user 2026-10-14; built only if the native path fails P3-GATE; spare connector reserved | Single clock domain on the Pi, but adds a hard-to-solder chip, firmware, and a multi-lane overlay |
| USB channels: 3 to the computer (processed L/R, dry), 2 from it; re-amping through the unit deferred | Selected by user 2026-10-14 | Dry track can be re-processed on the computer with the NAM plugin; re-amping needs only software later |
| microSD versus NVMe final storage | Open, measurement required | Pi 5 has no eMMC; reliability and complexity trade-offs are unresolved |
| Hybrid MCU audio processing | Deferred, not accepted | Adds clocking, routing, firmware, and preset synchronization complexity |

## 12. Main Risks and Mitigations

| Risk | Consequence | Mitigation |
|---|---|---|
| NAM A2-Full plus effects misses latency target | Poor playing feel | Measure buffer size and period count, reduce effect-chain complexity, tune real-time kernel/scheduling, and preserve thermal headroom; do not substitute A2-Lite as the Pi 5 tuning shortcut |
| PREEMPT_RT cannot meet the gate | Linux architecture may need deeper change | Investigate an alternative such as a Xenomai/EVL dual-kernel or purpose-built low-latency Yocto/Elk-style stack; Pi 5 support, PiPedal compatibility, maintenance state, and benefit are unproven and must be evaluated before adoption |
| VBUS isolation prevents USB-C device-role detection | Native gadget path cannot enumerate reliably | Hand-made cable/interposer test before layout; record CC and VBUS-presence behavior; evaluate legacy-OTG-style cable arrangement |
| Header power has unsafe inrush, droop, or brownout behavior | Boot failure, corruption, or hardware stress | Current-limited bench proof; board-level inrush limiting, sequencing, protection, bulk capacitance, monitoring, and repeated fault tests |
| Native USB gadget clock/routing instability | Recording glitches or drift | Four-hour duplex tests; asynchronous feedback/rate matching; activate the bridge fallback if the gate fails |
| USB routing service disturbs the real-time path | Xruns or lost CPU headroom for NAM | Lower priority than the audio threads, separate core, feedback instead of resampling, four-hour duplex test with NAM A2-Full; bridge fallback |
| Dedicated bridge conflicts with codec interfaces or adds latency | Board cannot support both local and USB audio goals | Confirm digital interface allocation and measure end-to-end latency before schematic freeze |
| Guitar input impedance or level is wrong | Dull tone or incorrect capture response | High-Z buffer, switchable/calibrated gain, scope and analyzer tests |
| Touch/UI load interferes with audio | Clicks or xruns | Separate ESP32-S3 UI, bounded status rates, and no audio over UART |
| A board-to-board cable is unplugged or fails | Outputs unmute unexpectedly, or a board is back-powered through its signal lines | Mute-gate inputs pulled to the muted state on the audio board; no signal driven into an unpowered board; unplug tests during board bring-up |
| SD card internal operations interrupted by sudden power loss | Corruption even with a read-only root | Orderly shutdown in normal use; high-endurance card; repeated plug-pull tests under V-STORAGE; NVMe option |
| Hold-up too short for the display power-off sequence | Display stress or visible artefacts when the plug is pulled | Hardware mute and Pi 5 cut on power fail; backlight off first; hold-up measured under V-POWER |
| Board placement or fan compromises analog noise/cooling | Noise, throttling, or xruns | Boards beside the Pi, analog zoning on one continuous ground plane, short ribbon, fan-state electrical/acoustic tests, and enclosed thermal stress test |
| ESP32-S3 cannot drive the panel cleanly at ≥ 23 MHz DCLK | Flicker or drift; UI unusable | Display bring-up adapter and display gate before any other UI work; bounce buffer; no Wi-Fi or flash writes on the ESP32-S3; fallback STM32U5G9 |
| ESP32-S3 GPIO budget is exhausted (30 of 31 used) | No pin for a late addition | I2C I/O expander for slow signals; SYNC mode without DE or encoder via expander as reserves |
| PiPedal websocket API changes between releases | Local UI breaks after an update | Bridge daemon is the only PiPedal client; pin PiPedal version; re-verify after updates |
| Ground loops between guitar amplifier, powered speakers, and computer | Mains hum | Single-point chassis ground; layout space for a ground-lift switch on the main outputs and for USB or line-output isolation, fitted only if needed; hum test adding one device at a time |
| Pi 5 has one I2S bus at a time (`I2S0`/`I2S1` share GPIO18–27); a USB bridge must share the codec's clocks | Bridge and codec clocking may be incompatible | Confirm the bridge can run as clock consumer on the codec's BCLK/LRCK with its own data lane, or use another link; decide before audio-board schematic freeze |
| microSD power-loss behavior is inadequate | Filesystem or preset corruption | Atomic writes, recovery tests, and measured comparison with optional NVMe |
| PCIe/NVMe addition expands layout scope | Signal-integrity, cost, power, and schedule risk | Keep optional until measured benefit justifies a separately reviewed layout |
| First custom board contains design errors | Schedule and cost overrun | Complete staged bring-up, accessible test points, current-limited power, and plan at least one respin |
| Boot or shutdown pop | Speaker/headphone hazard | Hardware-default mute, controlled ramps, and ESP32-supervised state machine |
| Preset/model corruption | Device cannot start a valid graph | Atomic writes, validation, known-good fallback preset, and recoverable image |

## 13. Verification Matrix

Each gate has a matching empty result table in [03-Measurements.md](03-Measurements.md).

| Gate ID | Requirement area | Verification |
|---|---|---|
| V-TONE | Captured-real-amp tone | Owner blind A/B tests using level-matched captures and conventional models |
| V-EFFECTS | General effects | Preset tests covering every required effect category and reorder/bypass behavior |
| V-BEGINNER | Beginner usability | First-use test: tune, select a preset, change gain, and save without documentation |
| V-TOUCH | Touchscreen | Touch accuracy, refresh rate, viewing-angle readability, text size and touch-target usability on the 5.0-inch panel, and recovery tests |
| V-FOOT | Foot control | Debounce, long-press, simultaneous switch, cable ESD, and live-use tests |
| V-OUTPUT | Main outputs and headphones | Level at Line and Instrument settings, noise, clipping, mute, plug/unplug, mono sum, headphone-insert muting, and load tests; hum test adding powered speakers, computer, and guitar amplifier one at a time |
| V-DISPLAY | Display bring-up | ESP32-S3 module identity, DCLK and polarity, refresh rate, flicker/drift, LVGL frame rate, GT911 touch, and display power-on/off sequencing |
| V-USB | USB recording/playback | Windows/Linux enumeration, DAW recording, four-hour duplex soak, reconnect, clock-drift, and power-isolation tests |
| V-LATENCY | Latency | Analog-in to analog-out loopback at representative CPU load; compare with targets and external context without treating third-party figures as project results |
| V-STABILITY | Stability | One-hour development and four-hour release tests with zero xruns |
| V-POWER | Power and thermal | Worst-chain current, inrush, sequencing, rail ripple, brownout, enclosure temperature, fan-noise, and throttling measurements |
| V-STORAGE | Storage | Boot time, atomic-write behavior, repeated power-loss recovery, endurance plan, and microSD/NVMe comparison if NVMe is evaluated |
| V-BOARD | Board bring-up | Test-point conformance, bare-board rails, codec clocks, analog comparison, mute safety, and integrated regression results |

## 14. References

- [PiPedal project and documentation](https://rerdavies.github.io/pipedal/)
- [PiPedal system requirements](https://rerdavies.github.io/pipedal/SystemRequirements.html)
- [PiPedal audio configuration and published latency examples](https://rerdavies.github.io/pipedal/Configuring.html)
- [NAM A2 guide](https://www.tone3000.com/guides/nam-a2-the-complete-guide)
- [Raspberry Pi 5 product page](https://www.raspberrypi.com/products/raspberry-pi-5/)
- [Raspberry Pi OTG whitepaper](https://pip-assets.raspberrypi.com/categories/685/app-notes-guides-whitepapers/documents/RP-009276-WP-1-Using%20OTG%20mode%20on%20Raspberry%20Pi%20SBCs)
- [Raspberry Pi USB Power Delivery whitepaper](https://pip-assets.raspberrypi.com/categories/685/app-notes-guides-whitepapers/documents/RP-009856-WP-1-USB%20Power%20delivery%20on%20Raspberry%20Pi%205.pdf)
- [Raspberry Pi NVMe documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#nvme-ssd-boot)
- [Linux USB Audio Class 2 gadget documentation](https://docs.kernel.org/usb/gadget-testing.html)
- [Pisound audio specifications](https://blokas.io/pisound/docs/audio/)
- [Crade-Cortex: community documentation of PiPedal's websocket protocol](https://github.com/EmilienAubert/Crade-Cortex)
- [ESP-IDF: RGB interfaced LCD (ESP32-S3)](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/lcd/rgb_lcd.html)
- [TDO TL050WVH12CT-B1714A LCD module specification rev 4.0](references/datasheets/display/TL050WVH12CT-B1714A_LCD-Module-Spec_Rev4.0.pdf)
