# Solar Boiler Controller (PWM) — Requirements

> **Platform:** Seeed Studio XIAO ESP32-C6
> **Document status:** Draft v0.1
> **Last change:** 2026-10-18
> **Language:** This English document is the **primary (authoritative)** version.
> A Bulgarian translation is kept in [requirementsBG.md](requirementsBG.md) with **identical requirement IDs**. Every change is made here first and then mirrored in the translation.

---

## 0. How to use this document

### 0.1 Numbering

Every requirement has a unique ID in the form `<GROUP>-<number>`, e.g. `PWM-03`.
IDs are **never reused** — if a requirement is dropped it is marked `[DROPPED]` instead of being deleted. This way code, tests and commits can reference a specific ID.

### 0.2 Priority

| Marker | Meaning |
|---|---|
| **[M]** | Must — the product is not done without it |
| **[S]** | Should — expected, but may be postponed |
| **[C]** | Could — idea / extra, if time allows |

### 0.3 Adding new requirements

- **New requirement in an existing feature:** add a row to the feature's table using the next free number.
- **New feature:** copy the template from [Appendix A](#appendix-a--new-feature-template), give it a new section number and a new group prefix, and add it to the [Feature list](#feature-list).
- **Open points** go to [Open questions](#16-open-questions) as `Q-xx`; they are not left inside requirement text.
- **Translation:** after every change, update [requirementsBG.md](requirementsBG.md) with the same IDs and add a line to the [Change log](#17-change-log) in both files.

### Feature list

| # | Feature | Prefix | Criticality |
|---|---|---|---|
| 1 | [System concept](#1-system-concept) | `SYS` | — |
| 2 | [Hardware](#2-hardware) | `HW` | — |
| 3 | [Software architecture](#3-software-architecture) | `ARCH` | — |
| 4 | [Heater PWM control](#4-heater-pwm-control) | `PWM` | **CRITICAL** |
| 5 | ["AC simulation" — periodic gaps](#5-ac-simulation--periodic-gaps-for-the-mechanical-thermostat) | `GAP` | **CRITICAL** |
| 6 | [Panel voltage measurement](#6-panel-voltage-measurement) | `VMEAS` | **CRITICAL** |
| 7 | [Current measurement](#7-current-measurement) | `IMEAS` | **CRITICAL** |
| 8 | [Temperature measurement](#8-temperature-measurement) | `TEMP` | **CRITICAL** |
| 9 | [MPPT algorithm](#9-mppt-algorithm) | `MPPT` | **CRITICAL** |
| 10 | [Protections](#10-protections) | `PROT` | **CRITICAL** |
| 11 | [Local UI (OLED, buttons, buzzer)](#11-local-ui-oled-buttons-buzzer) | `UI` | non-critical |
| 12 | [WiFi connectivity](#12-wifi-connectivity) | `WIFI` | non-critical |
| 13 | [Web server and web UI](#13-web-server-and-web-ui) | `WEB` | non-critical |
| 14 | [Data history](#14-data-history) | `HIST` | non-critical |
| 15 | [Settings and storage](#15-settings-and-storage) | `CFG` | mixed |

---

## 1. System concept

### 1.1 Description

The controller connects a **string of solar panels directly to the resistive heating element** of a water boiler (no inverter, no DC/DC converter). PWM adjusts the effective load seen by the panels so that they operate close to their maximum power point (MPP) under any irradiance.

**Physics in short:** with an input capacitor smoothing the panel current, the panels see an effective resistance
`R_eff ≈ R_heater / D`, where `D` is the duty cycle (0…1).
PWM can only **increase** the effective resistance relative to `R_heater`. Therefore:

- in strong sun (`D` close to 1) the heater must load the panels at a voltage **≤ Vmp**;
- in weak light the optimal resistance rises and the controller lowers `D`, keeping the panel voltage around Vmp.

### 1.2 Requirements

| ID | Pr. | Requirement |
|---|---|---|
| SYS-01 | [M] | The system works with a heater designed for 220–230 V AC, fed directly with DC from a solar string. |
| SYS-02 | [M] | The string is sized so that its **Vmp** is close to the heater's rated voltage (≈ 220–230 V). |
| SYS-03 | [M] | The heater resistance must be **≤ Vmp² / Pmp** of the array at full sun (i.e. the heater's rated power at 230 V ≥ the array power at the same voltage). Otherwise PWM cannot reach the MPP in strong sun. |
| SYS-04 | [M] | Regulation is done only by PWM of a power MOSFET — no DC/DC conversion. |
| SYS-05 | [M] | A mechanical thermostat (the boiler's thermostat) stays in series in the power circuit as an **independent hardware protection**. |
| SYS-06 | [M] | The goal is maximum energy harvest, including in cloudy weather and low light. |

---

## 2. Hardware

### 2.1 Components

| ID | Pr. | Requirement |
|---|---|---|
| HW-01 | [M] | Microcontroller: **Seeed Studio XIAO ESP32-C6**. |
| HW-02 | [M] | **PWM output** to a MOSFET (gate) driver that drives the power MOSFET. |
| HW-03 | [M] | **Analog input** for panel voltage through a resistive divider. |
| HW-04 | [M] | **Analog input** for current from a Hall-effect sensor. |
| HW-05 | [M] | Temperature: **DS18B20** (1-Wire) **or** a thermocouple on an analog input (with a suitable amplifier — see Q-03). Selected by a setting. |
| HW-06 | [M] | **Buzzer** (active or passive — see Q-04). |
| HW-07 | [M] | **3 push buttons**. |
| HW-08 | [M] | **I2C OLED display** (assumed SSD1306 / SH1106, 128×64 — see Q-05). |

### 2.2 Power stage and safety requirements

| ID | Pr. | Requirement |
|---|---|---|
| HW-10 | [M] | The driver's PWM input must have an **external pull-down**, so the MOSFET is **off** during reset, boot, MCU power loss or a floating pin. |
| HW-11 | [M] | The GPIO chosen for PWM must not be a strapping pin and must not toggle during boot. |
| HW-12 | [M] | The MOSFET and input capacitors are rated for the string's **Voc at the lowest expected temperature** (Voc rises ~10–20 % in cold), with margin. Guideline: MOSFET ≥ 600 V. |
| HW-13 | [M] | An input capacitor bank after the panels holds the voltage during the OFF part of the PWM period, so the panels stay at a stable operating point. |
| HW-14 | [S] | MOSFET overvoltage protection (TVS/snubber) against cable inductance. |
| HW-15 | [M] | The voltage divider is sized so that the ADC input never exceeds its limit at the maximum possible voltage, and the input is protected (series resistor + clamping). |
| HW-16 | [M] | **Safety:** the circuit carries a dangerous DC voltage (~300 V). It must be clearly defined whether the MCU ground is tied to the panel negative. If it is, connecting USB to a computer while the panels are connected is dangerous, and this must be prevented by design or by isolation (see Q-06). |

### 2.3 Preliminary pin assignment (to be verified)

> The XIAO ESP32-C6 exposes only 3 analog pins (A0–A2) and 11 GPIOs in total. This assignment is tentative and **must be checked against Seeed's schematic** before PCB design.

| Pin | GPIO | Function |
|---|---|---|
| A0 / D0 | GPIO0 | Panel voltage (ADC) |
| A1 / D1 | GPIO1 | Hall current sensor (ADC) |
| A2 / D2 | GPIO2 | Thermocouple via amplifier (ADC) — optional |
| D3 | GPIO21 | PWM output to driver |
| D4 | GPIO22 | I2C SDA (OLED) |
| D5 | GPIO23 | I2C SCL (OLED) |
| D6 | GPIO16 | DS18B20 (1-Wire) |
| D7 | GPIO17 | Button 1 |
| D8 | GPIO19 | Button 2 |
| D9 | GPIO20 | Button 3 |
| D10 | GPIO18 | Buzzer |

---

## 3. Software architecture

### 3.1 Principle

The architecture must be **very good and at the same time very simple**. The software is split into two clearly separated zones:

```
┌──────────────────────────────── CRITICAL CORE ─────────────────────────────────┐
│  ControlTask (highest priority, fixed period)                                  │
│    read V, I, T  →  protections  →  MPPT  →  set PWM duty                       │
│  Exclusively owns: ADC, temperature sensor, PWM peripheral                     │
└────────────────────────────────────────────────────────────────────────────────┘
          │ publishes a snapshot (read-only)               ▲ takes validated
          ▼                                                │ settings (copy)
┌──────────────────────────────── NON-CRITICAL MODULES ──────────────────────────┐
│  UI (OLED, buttons, buzzer) │ WiFi │ Web server │ History │ Settings storage   │
└────────────────────────────────────────────────────────────────────────────────┘
```

**Core rule:** whatever happens in the non-critical modules, the core (PWM + voltage + temperature) **never stops**. If the core itself cannot operate correctly, the PWM output **must be off**. There is no third state.

### 3.2 Critical core requirements

| ID | Pr. | Requirement |
|---|---|---|
| ARCH-01 | [M] | All critical logic (V/I/T measurement, protections, MPPT, setting PWM) runs in **one task** (`ControlTask`) with the **highest priority** among application tasks. |
| ARCH-02 | [M] | `ControlTask` runs with a **fixed period** (proposal: 20 ms, in sync with the 50 Hz window from section 5). |
| ARCH-03 | [M] | `ControlTask` **never blocks**: it does not wait on a mutex shared with non-critical tasks, does not write to flash, does no network I/O, does not draw on the display, and does not log synchronously. |
| ARCH-04 | [M] | `ControlTask` is the only owner of the V/I ADC channels, the temperature sensor and the PWM peripheral. No other module accesses them directly. |
| ARCH-05 | [M] | Data for non-critical modules is provided as a **snapshot** (struct of values + version counter) that is copied without blocking the core (double buffer / seqlock). |
| ARCH-06 | [M] | Settings reach the core as a **validated copy**. The core accepts a new copy only at the start of a cycle and only if it passed range checks. Invalid settings are rejected and the core continues with the old ones. |
| ARCH-07 | [M] | The core is guarded by a **hardware/task watchdog**. If `ControlTask` misses its deadlines → reset → PWM is off at startup (see HW-10). |
| ARCH-08 | [M] | The core starts **first** at boot and begins controlling (in safe mode) before WiFi, web and other modules are initialized. A failed init of a non-critical module does not stop the core. |
| ARCH-09 | [M] | The execution time of one core cycle is measured and published (max/average) to monitor headroom. |
| ARCH-10 | [M] | Since the ESP32-C6 has a single main core shared with the WiFi stack, the priority and duration of `ControlTask` must guarantee a deterministic period even during heavy WiFi traffic. |

### 3.3 Non-critical module requirements

| ID | Pr. | Requirement |
|---|---|---|
| ARCH-20 | [M] | Each non-critical module runs in its own task and talks to the core only via the snapshot (ARCH-05) and the settings queue (ARCH-06). |
| ARCH-21 | [M] | A non-critical module that hangs or crashes **does not affect** the core. The module **either works or stays off**. |
| ARCH-22 | [S] | A lightweight supervisor monitors a heartbeat from each non-critical module. On a detected hang it tries to restart the module; after N failed attempts the module is disabled until the next reboot and a warning is raised. |
| ARCH-23 | [S] | Rebooting the whole device because of a non-critical module is a **last resort** and is configurable (can be disabled). |
| ARCH-24 | [M] | I2C transactions to the display use timeouts and cannot block any task indefinitely (the I2C bus can get stuck). |
| ARCH-25 | [S] | Flash writes (history, settings) are done only by non-critical tasks and are rate-limited to protect flash endurance. |

---

## 4. Heater PWM control

| ID | Pr. | Requirement |
|---|---|---|
| PWM-01 | [M] | PWM frequency: **low, default 1 kHz**. Configurable within a safe range (proposal: 500 Hz – 2 kHz). |
| PWM-02 | [M] | Duty resolution: at least 10 bits (proposal: ≤ 0.1 % steps). |
| PWM-03 | [M] | PWM is generated by a **hardware peripheral** (LEDC or MCPWM), not in software. |
| PWM-04 | [M] | The duty cycle is set **only** by `ControlTask`. |
| PWM-05 | [M] | Duty changes are applied in sync with the end of a PWM period (no truncated pulses). |
| PWM-06 | [M] | Configurable `duty_min` and `duty_max` limits. |
| PWM-07 | [M] | At startup PWM begins at 0 and ramps up smoothly (soft start). |
| PWM-08 | [M] | Any fault state (protection trip, invalid sensor, failed init) results in **duty = 0**. |
| PWM-09 | [S] | Manual mode (fixed duty) for testing and service, available from buttons and web, with all protections active. |
| PWM-10 | [M] | Operating modes: `OFF`, `AUTO` (MPPT), `MANUAL` (see PWM-09). The default mode after boot is configurable, but protections always take precedence. |

---

## 5. "AC simulation" — periodic gaps for the mechanical thermostat

### 5.1 Idea

Boiler thermostats are mechanical and designed for AC. With AC the current crosses zero 100 times per second, so the arc that forms when the contacts open extinguishes by itself. With DC the arc **does not extinguish** and can destroy the contacts or start a fire.

Therefore the PWM signal makes a **longer current-free gap 50 times per second** (every 20 ms). During the gap the current through the thermostat is zero, just like at an AC zero crossing, so if the contacts are opening the arc goes out.

```
       ← ─ ─ ─ ─ ─ ─ ─ ─ 20 ms (50 Hz) ─ ─ ─ ─ ─ ─ ─ ─ →
 ON  ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐        ┌┐ ┌┐
     ││ ││ ││ ││ ││ ││ ││ ││ ││ ││ ││ ││ ││ ││ ││ ││ ││        ││ ││
 OFF ┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘└────────┘└─┘└─
      ←1ms→ (PWM 1 kHz)                                   ← gap →
```

### 5.2 Requirements

| ID | Pr. | Requirement |
|---|---|---|
| GAP-01 | [M] | A current-free gap is inserted at **50 Hz** (every 20 ms). The rate is configurable (proposal: 50 or 100 Hz). |
| GAP-02 | [M] | The gap duration is **configurable** (proposal: 0.5–3 ms, default ~1–2 ms — to be confirmed by testing, see Q-07). |
| GAP-03 | [M] | The gap is **guaranteed** — the output is off for the entire gap regardless of the current duty, including duty = 100 %. |
| GAP-04 | [M] | The gap is synchronized with PWM: it starts at the end of a PWM period and never cuts a pulse in the middle. |
| GAP-05 | [M] | Gap generation must not depend on non-critical tasks. Hardware generation (MCPWM / timer) or an ISR is preferred over a regular task. |
| GAP-06 | [M] | The reduction of the effective maximum duty caused by the gaps (e.g. 1 ms per 20 ms = −5 %) is taken into account in power calculations and in MPPT. |
| GAP-07 | [M] | The feature **cannot be disabled** from the user interface in `AUTO` mode (at most in a service mode with a clear warning). |
| GAP-08 | [S] | Validation: test with the real thermostat at maximum current to confirm the arc extinguishes at the chosen gap duration. |

---

## 6. Panel voltage measurement

| ID | Pr. | Requirement |
|---|---|---|
| VMEAS-01 | [M] | Panel voltage is measured through a resistive divider on an analog input. |
| VMEAS-02 | [M] | The ESP32-C6 calibrated ADC reading (eFuse calibration) is used, plus a user gain and offset. |
| VMEAS-03 | [M] | The measurement is averaged/filtered (many samples per cycle) to suppress PWM ripple. |
| VMEAS-04 | [S] | Sampling is synchronized with the PWM/gap or dense enough to avoid aliasing. |
| VMEAS-05 | [M] | Range: from 0 to at least cold-weather Voc plus margin (proposal: 0–400 V). |
| VMEAS-06 | [M] | Invalid readings are detected (saturated ADC, open divider, implausible jumps) → PWM off + fault. |
| VMEAS-07 | [S] | Calibration from the web UI: the user enters a voltage measured with a multimeter and the controller computes the gain. |

---

## 7. Current measurement

| ID | Pr. | Requirement |
|---|---|---|
| IMEAS-01 | [M] | Current is measured with a Hall-effect sensor on an analog input. |
| IMEAS-02 | [M] | The sensor zero (offset) is auto-calibrated at startup while PWM is 0, and can be calibrated manually. |
| IMEAS-03 | [M] | Sensitivity (mV/A) is configurable for the sensor model used. |
| IMEAS-04 | [M] | Filtering/averaging as for voltage. |
| IMEAS-05 | [M] | Instantaneous power `P = V × I` and accumulated energy (Wh/kWh) are calculated. |
| IMEAS-06 | [M] | Invalid sensor detection (out of range, missing) → fault. Behaviour on current-sensor failure is decided in Q-09 (whether the core may continue in a limited voltage-only mode). |
| IMEAS-07 | [S] | Sensor placement (panel side or heater side) is decided in Q-08. |

---

## 8. Temperature measurement

| ID | Pr. | Requirement |
|---|---|---|
| TEMP-01 | [M] | **DS18B20** (1-Wire) is supported. |
| TEMP-02 | [S] | A **thermocouple** via analog input (with an external amplifier) is supported. |
| TEMP-03 | [M] | The sensor type is selected by a setting. |
| TEMP-04 | [M] | Reading the DS18B20 **does not block** the core: the conversion (up to 750 ms) is started asynchronously and the result is fetched in a later cycle. |
| TEMP-05 | [M] | Fault detection: missing sensor, CRC error, 85 °C power-on value, −127 °C, implausible jump. |
| TEMP-06 | [M] | If temperature is invalid for longer than a configurable time (e.g. 5 s) → **PWM off** + alarm. |
| TEMP-07 | [M] | Configurable **target temperature** and **hysteresis**: PWM stops when the target is reached and resumes when the temperature drops below `target − hysteresis`. |
| TEMP-08 | [M] | Separate **maximum (emergency) temperature** — above it PWM is turned off regardless of any other setting. |
| TEMP-09 | [S] | Temperature offset correction. |

---

## 9. MPPT algorithm

> **Custom algorithm — to be described separately by the user.**
> Until then this section only defines the general frame the algorithm must fit into.

| ID | Pr. | Requirement |
|---|---|---|
| MPPT-01 | [M] | The algorithm runs inside `ControlTask` and complies with all constraints of section 3.2 (non-blocking, deterministic time). |
| MPPT-02 | [M] | The algorithm's output is a requested duty, which then passes through the limits (`duty_min/max`, protections, temperature). The algorithm **cannot** bypass protections. |
| MPPT-03 | [M] | The algorithm is isolated in its own module with a clear interface (input: V, I, P, time; output: duty) so it can be swapped and tested. |
| MPPT-04 | [S] | The algorithm's internal state is published in the snapshot for diagnostics (web/charts). |
| MPPT-05 | — | *(Algorithm details — to be described.)* |

---

## 10. Protections

All protections run in the critical core. On a trip PWM is turned off (duty = 0).

| ID | Pr. | Requirement |
|---|---|---|
| PROT-01 | [M] | **Overtemperature:** T > T_max → shutdown + alarm (see TEMP-08). |
| PROT-02 | [M] | **Temperature sensor fault** → shutdown (see TEMP-06). |
| PROT-03 | [M] | **Overvoltage:** V > V_max → shutdown. |
| PROT-04 | [M] | **Overcurrent:** I > I_max → shutdown. |
| PROT-05 | [M] | **Undervoltage:** V < V_min → PWM stops (night / too little light), no alarm. |
| PROT-06 | [M] | **Voltage sensor fault** → shutdown. |
| PROT-07 | [M] | Each protection has a threshold, hysteresis and/or re-enable delay (to avoid chattering). |
| PROT-08 | [S] | Plausibility protection "current without voltage / voltage without current" — detect a burnt heater, open thermostat or shorted MOSFET (e.g. current flowing at duty = 0). |
| PROT-09 | [M] | Every protection trip is recorded in the event log (see HIST). |
| PROT-10 | [M] | Protection thresholds are validated on input; invalid or dangerous values are rejected. |

---

## 11. Local UI (OLED, buttons, buzzer)

**Non-critical module** — if it hangs, the core keeps running.

### 11.1 OLED display

| ID | Pr. | Requirement |
|---|---|---|
| UI-01 | [M] | Main screen: voltage, current, power, temperature, duty, mode, status/fault. |
| UI-02 | [M] | Additional screens: energy today/total, WiFi status and IP address, active alarms. |
| UI-03 | [S] | Menu for basic settings (target temperature, mode, manual duty, WiFi reset). |
| UI-04 | [S] | Automatic dimming/off after inactivity (against OLED burn-in). |

### 11.2 Buttons (3 pcs)

| ID | Pr. | Requirement |
|---|---|---|
| UI-10 | [M] | Software debounce. |
| UI-11 | [M] | Short and long press detection. |
| UI-12 | [M] | Proposed meaning: **Up / Down / OK** (long OK = enter/leave menu). |
| UI-13 | [S] | Special hold combination for: forced AP mode, factory reset. |

### 11.3 Buzzer

| ID | Pr. | Requirement |
|---|---|---|
| UI-20 | [M] | Audible signal on alarm (overtemperature, sensor fault, protection trip). |
| UI-21 | [S] | Short beep on button press (can be disabled). |
| UI-22 | [S] | Ability to silence the current alarm from a button or the web. |

---

## 12. WiFi connectivity

**Non-critical module.**

| ID | Pr. | Requirement |
|---|---|---|
| WIFI-01 | [M] | At startup the controller tries to connect as **STA** to the stored network. |
| WIFI-02 | [M] | If there is no stored network, or the connection fails within a configurable time (proposal: 30 s), it switches to **AP mode**. |
| WIFI-03 | [M] | In AP mode the web UI allows **scanning for networks**, choosing a router and entering its password. |
| WIFI-04 | [S] | Captive portal in AP mode (the phone opens the page automatically). |
| WIFI-05 | [S] | In AP mode the controller periodically retries STA (if a network is stored), e.g. after a power outage when the router boots slower. |
| WIFI-06 | [M] | On connection loss in STA mode — automatic reconnect. |
| WIFI-07 | [S] | AP name: `SolarBoiler-XXXX` (XXXX from the MAC address), with a configurable password. |
| WIFI-08 | [S] | mDNS hostname (e.g. `solarboiler.local`). |
| WIFI-09 | [S] | Time sync via NTP in STA mode (for history timestamps). |
| WIFI-10 | [M] | No WiFi activity affects the period of `ControlTask` (see ARCH-10). |

---

## 13. Web server and web UI

**Non-critical module.**

### 13.1 General

| ID | Pr. | Requirement |
|---|---|---|
| WEB-01 | [M] | Built-in web server, reachable in both STA and AP mode. |
| WEB-02 | [M] | All UI files (HTML/JS/CSS, chart library) are stored **on the controller** — no external CDNs, because there is no internet in AP mode. |
| WEB-03 | [S] | Files are stored compressed (gzip) for less space and faster loading. |
| WEB-04 | [M] | REST/JSON API for data and settings — the UI uses only this API. |
| WEB-05 | [S] | Password protection for settings. |
| WEB-06 | [S] | The UI works well on a phone. |

### 13.2 Live data

| ID | Pr. | Requirement |
|---|---|---|
| WEB-10 | [M] | Live data: V, I, P, T, duty, mode, status, alarms, energy today. |
| WEB-11 | [M] | Real-time updates via WebSocket or SSE (proposal: 1–2 times per second). |
| WEB-12 | [S] | Diagnostics: core cycle time, free memory, uptime, WiFi signal strength, module states. |

### 13.3 Settings

| ID | Pr. | Requirement |
|---|---|---|
| WEB-20 | [M] | A page with all settings from section 15, grouped logically. |
| WEB-21 | [M] | Values are validated both in the browser and on the controller (the controller has the final say). |
| WEB-22 | [M] | WiFi settings (change network, forget network). |
| WEB-23 | [S] | Sensor calibration (voltage, current, temperature). |
| WEB-24 | [S] | Export/import of settings as a JSON file. |
| WEB-25 | [S] | Reboot and factory reset. |

### 13.4 Charts and history

| ID | Pr. | Requirement |
|---|---|---|
| WEB-30 | [M] | History charts: voltage, current, power, temperature, duty over time. |
| WEB-31 | [M] | Period selection (e.g. last hour, 24 hours, by day). |
| WEB-32 | [S] | Daily energy chart (bars, kWh/day). |
| WEB-33 | [S] | Event log (alarms, protection trips, reboots). |
| WEB-34 | [C] | Download history as CSV. |

---

## 14. Data history

**Non-critical module.** The core only publishes the snapshot; history is collected by a separate task.

| ID | Pr. | Requirement |
|---|---|---|
| HIST-01 | [M] | History is kept in the controller's memory. |
| HIST-02 | [M] | **Short-term history in RAM** — ring buffer with dense samples (proposal: 1-minute samples for the last 24 hours; V, I, P, T, duty — avg/min/max per interval). |
| HIST-03 | [M] | **Long-term history in flash** (LittleFS) — daily summaries (proposal: energy kWh, max T, max P, operating hours, for at least 1 year). |
| HIST-04 | [S] | Short-term history is periodically saved to flash so it survives a reboot (write rate limited for flash wear). |
| HIST-05 | [M] | **Event log** (ring, in flash): alarms, protection trips, reboots and their reason, settings changes. |
| HIST-06 | [M] | Records are timestamped. Without synchronized time (no NTP), relative time since boot is stored and corrected once time becomes available. |
| HIST-07 | [M] | Lifetime energy counter (kWh) — not reset on reboot. |
| HIST-08 | [M] | Full history storage never stops anything — the oldest records are overwritten. |

---

## 15. Settings and storage

| ID | Pr. | Requirement |
|---|---|---|
| CFG-01 | [M] | Settings are stored in **NVS** (non-volatile storage) and survive reboots. |
| CFG-02 | [M] | Every setting has a **default**, **minimum** and **maximum**, defined in one place in the code. |
| CFG-03 | [M] | On corrupted/missing settings, defaults are loaded and an event is logged. |
| CFG-04 | [M] | Settings are **versioned** — on a new firmware version old settings are migrated or filled with defaults. |
| CFG-05 | [M] | Settings changes reach the core only as described in ARCH-06. |
| CFG-06 | [M] | Flash writes are done by a non-critical task and only on an actual change. |

**Initial list of settings** (to be extended):

| Group | Setting |
|---|---|
| PWM | frequency, duty_min, duty_max, default mode, manual duty |
| AC simulation | gap rate, gap duration |
| Temperature | sensor type, target T, hysteresis, T_max, offset |
| Voltage | gain/calibration, V_min, V_max |
| Current | sensitivity, offset, I_max |
| MPPT | *(to be added with the algorithm description)* |
| WiFi | SSID, password, connect timeout, AP password, hostname |
| Web | access password |
| UI | button beep, display dimming timeout |

---

## 16. Open questions

| ID | Question |
|---|---|
| Q-01 | What exactly is the MPPT algorithm? *(to be described)* |
| Q-02 | What are the heater power and the solar array parameters (number of panels, Vmp, Voc, Imp)? Needed to check SYS-03 and for sizing. |
| Q-03 | Thermocouple — which type (K?) and which amplifier? Analog (e.g. AD8495) or digital (MAX31855/MAX6675 over SPI — that would not be an analog input)? |
| Q-04 | Is the buzzer active (on/off only) or passive (different tones via PWM)? |
| Q-05 | Exact OLED model (SSD1306 / SH1106, 0.96" / 1.3")? |
| Q-06 | How is the controller powered, and is there galvanic isolation between the MCU and the DC power circuit? |
| Q-07 | What gap duration is sufficient to extinguish the arc in the actual thermostat at maximum current? |
| Q-08 | Hall sensor — on the panel side (average current, better for MPPT) or the heater side (pulsed current)? Model and range? |
| Q-09 | If the current sensor fails — shut everything down, or continue in a limited voltage-only mode? |
| Q-10 | Are OTA firmware updates via web needed? |
| Q-11 | Arduino framework or pure ESP-IDF? |

---

## 17. Change log

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-10-18 | Initial draft: concept, hardware, architecture, PWM, AC simulation, measurements, protections, UI, WiFi, web, history, settings. |

---

## Appendix A — New feature template

```markdown
## N. <Feature name>

**Criticality:** CRITICAL / non-critical module

### N.1 Description

<What it does and why.>

### N.2 Requirements

| ID | Pr. | Requirement |
|---|---|---|
| XXX-01 | [M] | ... |
| XXX-02 | [S] | ... |
```

After adding: list the feature in the [Feature list](#feature-list), record the change in the [Change log](#17-change-log), and mirror everything in [requirementsBG.md](requirementsBG.md) with the same IDs.
