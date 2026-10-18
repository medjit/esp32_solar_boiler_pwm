# Solar Boiler Controller (PWM) — Requirements

> **Platform:** Seeed Studio XIAO ESP32-C6
> **Document status:** Draft v0.7
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
- **Open points** go to [Open questions](#18-open-questions) as `Q-xx`; they are not left inside requirement text.
- **Translation:** after every change, update [requirementsBG.md](requirementsBG.md) with the same IDs and add a line to the [Change log](#19-change-log) in both files.

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
| 11 | [Buzzer](#11-buzzer) | `UI` | non-critical |
| 12 | [WiFi connectivity](#12-wifi-connectivity) | `WIFI` | non-critical |
| 13 | [Web server and web UI](#13-web-server-and-web-ui) | `WEB` | non-critical |
| 14 | [Data history](#14-data-history) | `HIST` | non-critical |
| 15 | [Settings and storage](#15-settings-and-storage) | `CFG` | mixed |
| 16 | [MOSFET heatsink thermal management](#16-mosfet-heatsink-thermal-management) | `HSINK` | **CRITICAL** |
| 17 | [Firmware size and flash layout](#17-firmware-size-and-flash-layout) | `FW` | — |

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
| HW-02 | [M] | **Two outputs** to the MOSFET (gate) driver that drives the power MOSFET: the **PWM** signal and the **gap window / enable** signal (see section 2.3 and GAP-05). |
| HW-03 | [M] | **Analog input** for panel voltage through a resistive divider. |
| HW-04 | [M] | **Analog input** for current from a Hall-effect sensor. |
| HW-05 | [M] | Temperature: **DS18B20** (1-Wire) **or** a thermocouple on an analog input (with a suitable amplifier — see Q-03). Selected by a setting. |
| HW-06 | [M] | **Buzzer** (active or passive — see Q-04). |
| HW-07 | [M] | ~~**3 push buttons**.~~ **[DROPPED v0.7]** — no local UI, the web UI is the only interface. |
| HW-08 | [M] | ~~**I2C OLED display**.~~ **[DROPPED v0.7]** — no local UI, the web UI is the only interface. |
| HW-09 | [M] | **Heatsink temperature sensor** for the power MOSFET(s), on its own free pin (see section 16 and Q-13). |
| HW-17 | [M] | **Fan on/off output** for the heatsink fan, on its own free pin, through a transistor/MOSFET driver (see HW-18). |

### 2.2 Power stage and safety requirements

| ID | Pr. | Requirement |
|---|---|---|
| HW-10 | [M] | **Both** driver inputs (PWM and enable) must have an **external pull** to their inactive level, so the MOSFET is **off** during reset, boot, MCU power loss or a floating pin. |
| HW-11 | [M] | The GPIOs chosen for PWM and for the gap window must not be strapping pins and must not toggle during boot. |
| HW-12 | [M] | The MOSFET and input capacitors are rated for the string's **Voc at the lowest expected temperature** (Voc rises ~10–20 % in cold), with margin. Guideline: MOSFET ≥ 600 V. |
| HW-13 | [M] | An input capacitor bank after the panels holds the voltage during the OFF part of the PWM period, so the panels stay at a stable operating point. |
| HW-14 | [S] | MOSFET overvoltage protection (TVS/snubber) against cable inductance. |
| HW-15 | [M] | The voltage divider is sized so that the ADC input never exceeds its limit at the maximum possible voltage, and the input is protected (series resistor + clamping). Circuit and values: section 6.2. |
| HW-16 | [M] | **Safety:** the circuit carries a dangerous DC voltage (~300 V). It must be clearly defined whether the MCU ground is tied to the panel negative. If it is, connecting USB to a computer while the panels are connected is dangerous, and this must be prevented by design or by isolation (see Q-06). |
| HW-18 | [M] | The fan is switched by a transistor/MOSFET (the GPIO never drives the fan directly), with a flyback diode across the fan and an external pull-down on the driver input, so the fan is off during reset/boot. Fan supply voltage — see Q-14. |
| HW-25 | [M] | The power switch is a **low-side N-channel MOSFET** (between the heater and PV−). Consequence: the MCU ground is the PV− potential (see HW-16), and a low-side driver is sufficient (no bootstrap / high-side stage needed). |
| HW-26 | [M] | **Safety:** with low-side switching the heater and the thermostat stay connected to PV+ even when the MOSFET is off. A **DC-rated** isolator/breaker between the panels and the controller is required for servicing. |

### 2.3 MOSFET driver

| ID | Pr. | Requirement |
|---|---|---|
| HW-19 | [M] | The driver has a **logic PWM input** that accepts 3.3 V logic from the ESP32 directly. |
| HW-20 | [M] | The driver has a **second, independent enable/shutdown input** that turns the output off regardless of the PWM input. The gap window (GAP-05) and the emergency off (GAP-10) use this input, so no external AND gate is needed. |
| HW-21 | [M] | **Fail-safe inputs:** with floating inputs (reset, boot, broken wire) the driver output is off — by internal or external pull resistors (see HW-10). |
| HW-22 | [M] | If the driver output used for the MOSFET is **inverted** relative to the PWM input (e.g. LO of IR2104: LO on when IN is low), then the pull-down on the shutdown input is mandatory and the PWM signal is inverted in the ESP32 GPIO matrix. |
| HW-23 | [M] | **Self-oscillating drivers** (IR2151, IR2153 family) are **not suitable**: they have no logic PWM input (pins 2/3 are the RT/CT oscillator) and generate a fixed ~50 % duty. |
| HW-24 | [S] | Candidates (final choice — Q-16): **IR2104 / IR2184** — pin-compatible with the existing 8-pin footprint, IN + SD̅ on the two already wired pins (note HW-22); **UCC27517** — single low-side driver with IN+ / IN− (built-in AND), needs a new footprint; **UCC27524A** — dual low-side with separate EN pins. If galvanic isolation is required (Q-06), an isolated driver with an enable input is used instead. |

### 2.4 Preliminary pin assignment (to be verified)

> The XIAO ESP32-C6 exposes only 3 analog pins (A0–A2) and 11 GPIOs in total. This assignment is tentative and **must be checked against Seeed's schematic** before PCB design.

| Pin | GPIO | Function |
|---|---|---|
| A0 / D0 | GPIO0 | Panel voltage (ADC) |
| A1 / D1 | GPIO1 | Hall current sensor (ADC) |
| A2 / D2 | GPIO2 | Thermocouple via amplifier (ADC) — optional |
| D3 | GPIO21 | PWM output → driver PWM input |
| TBD (already wired on the board) | TBD | Gap window / emergency off → driver enable/shutdown input |
| D4 | GPIO22 | — free (was I2C SDA; OLED removed) |
| D5 | GPIO23 | — free (was I2C SCL; OLED removed) |
| D6 | GPIO16 | DS18B20 (1-Wire) |
| D7 | GPIO17 | — free (was button 1) |
| D8 | GPIO19 | — free (was button 2) |
| D9 | GPIO20 | — free (was button 3) |
| D10 | GPIO18 | Buzzer |
| free pin #1 | TBD | Heatsink temperature sensor (see Q-12, Q-13) |
| free pin #2 | TBD | Fan on/off output (see Q-12) |

---

## 3. Software architecture

### 3.1 Principle

The architecture must be **very good and at the same time very simple**. The software is split into two clearly separated zones:

```
┌──────────────────────────────── CRITICAL CORE ─────────────────────────────────┐
│  GPTimer (hardware, every 20 ms) ──ISR──▶ wakes ControlTask                    │
│  ControlTask (highest priority, every 20 ms)                                   │
│    V, I (ADC DMA avg)  →  protections  →  MPPT  →  set PWM duty, fan           │
│  TempTask (medium priority, every 2–5 s)                                       │
│    DS18B20: request → wait ~750 ms → read + CRC → publish value + timestamp    │
│  AC gap: LEDC window channel → driver enable input (pure hardware)             │
└────────────────────────────────────────────────────────────────────────────────┘
          │ publishes a snapshot (read-only)               ▲ takes validated
          ▼                                                │ settings (copy)
┌──────────────────────────────── NON-CRITICAL MODULES ──────────────────────────┐
│  Buzzer │ WiFi │ Web server │ History │ Settings storage                       │
└────────────────────────────────────────────────────────────────────────────────┘
```

**Core rule:** whatever happens in the non-critical modules, the core (PWM + voltage + temperature) **never stops**. If the core itself cannot operate correctly, the PWM output **must be off**. There is no third state.

### 3.2 Critical core requirements

| ID | Pr. | Requirement |
|---|---|---|
| ARCH-01 | [M] | All critical logic runs in **two tasks**: `ControlTask` (V/I measurement, protections, MPPT, PWM, fan) with the **highest priority** among application tasks, and `TempTask` (temperature sensors, see ARCH-15). |
| ARCH-02 | [M] | `ControlTask` runs with a **fixed period of 20 ms**, equal to the gap period from section 5, so every control cycle covers exactly 20 PWM periods at 1 kHz. |
| ARCH-03 | [M] | `ControlTask` **never blocks**: it does not wait on a mutex shared with non-critical tasks, does not write to flash, does no network I/O, and does not log synchronously. |
| ARCH-04 | [M] | `ControlTask` is the only owner of the ADC, the PWM peripheral (heater PWM and gap window channels) and the fan output. `TempTask` is the only owner of the 1-Wire bus(es). (An NTC sensor on ADC is read by `ControlTask` as part of the ADC sampling.) No other module accesses them directly. |
| ARCH-05 | [M] | Data for non-critical modules is provided as a **snapshot** (struct of values + version counter) that is copied without blocking the core (double buffer / seqlock). |
| ARCH-06 | [M] | Settings reach the core as a **validated copy**. The core accepts a new copy only at the start of a cycle and only if it passed range checks. Invalid settings are rejected and the core continues with the old ones. |
| ARCH-07 | [M] | The core is guarded by a **hardware/task watchdog**. `ControlTask` feeds it **after each completed cycle** — the timer firing alone is not proof of health. If `ControlTask` stops completing cycles → reset → PWM is off at startup (see HW-10). |
| ARCH-08 | [M] | The core starts **first** at boot and begins controlling (in safe mode) before WiFi, web and other modules are initialized. A failed init of a non-critical module does not stop the core. |
| ARCH-09 | [M] | The execution time of one core cycle is measured and published (max/average) to monitor headroom. |
| ARCH-10 | [M] | Since the ESP32-C6 has a single main core shared with the WiFi stack, the priority and duration of `ControlTask` must guarantee a deterministic period even during heavy WiFi traffic. Therefore `ControlTask` priority is **above the WiFi and network system tasks**, and its execution time budget is **≤ 2 ms per cycle** (≤ 10 % of the period). |
| ARCH-11 | [M] | The 20 ms period is clocked by a **hardware general-purpose timer (GPTimer)**: counter resolution e.g. 1 MHz (1 µs per tick), alarm at 20 000 ticks, auto-reload. (The heater PWM uses its own LEDC/MCPWM timer, so it does not consume a GPTimer.) |
| ARCH-12 | [M] | The timer ISR only wakes `ControlTask` (task notification). No ADC reads, driver calls or control logic run inside the ISR. The ISR is placed in IRAM. |
| ARCH-13 | [M] | **Late/missed cycle tolerance:** if a cycle is delayed (e.g. tasks stalled during a flash write/erase), PWM keeps the last duty; the next cycle uses the actual elapsed time (`dt`) for energy integration and MPPT. Late and missed cycles are counted and published. |
| ARCH-14 | [M] | No float-heavy or `double` math beyond the cycle budget: the ESP32-C6 has no FPU, so `float` is software-emulated. Execution time is verified against ARCH-10 (see ARCH-09). |
| ARCH-15 | [M] | **`TempTask`** has medium priority (below `ControlTask`, above non-critical modules). Per sensor: request conversion → wait with `vTaskDelay` (no busy waiting) → read → check CRC → publish value **with a timestamp** to `ControlTask` using the non-blocking mechanism of ARCH-05. While waiting it holds no resource that `ControlTask` needs. |
| ARCH-16 | [M] | **Freshness check:** `ControlTask` treats a temperature as invalid if it is older than a configurable limit (default 3 × read interval). An invalid temperature leads to PWM off (TEMP-06, HSINK-10). This keeps the "works or stays off" rule even though temperatures are read in another task. |

### 3.3 Non-critical module requirements

| ID | Pr. | Requirement |
|---|---|---|
| ARCH-20 | [M] | Non-critical modules run at **low priority** (Arduino `loop()` or their own low-priority tasks) and talk to the core only via the snapshot (ARCH-05) and the settings queue (ARCH-06). |
| ARCH-21 | [M] | A non-critical module that hangs or crashes **does not affect** the core. The module **either works or stays off**. |
| ARCH-22 | [S] | A lightweight supervisor monitors a heartbeat from each non-critical module. On a detected hang it tries to restart the module; after N failed attempts the module is disabled until the next reboot and a warning is raised. |
| ARCH-23 | [S] | Rebooting the whole device because of a non-critical module is a **last resort** and is configurable (can be disabled). |
| ARCH-24 | [M] | ~~I2C transactions to the display use timeouts.~~ **[DROPPED v0.7]** — no display, no I2C. |
| ARCH-25 | [S] | Flash writes (history, settings) are done only by non-critical tasks and are rate-limited to protect flash endurance. |

---

## 4. Heater PWM control

| ID | Pr. | Requirement |
|---|---|---|
| PWM-01 | [M] | PWM frequency: **low, default 1 kHz**. Configurable within a safe range (proposal: 500 Hz – 2 kHz). |
| PWM-02 | [M] | Duty resolution: at least 10 bits (proposal: ≤ 0.1 % steps). |
| PWM-03 | [M] | PWM is generated by the **LEDC hardware peripheral**, not in software. The heater PWM and the gap window use two LEDC timers clocked from the same source (see GAP-09). |
| PWM-04 | [M] | The duty cycle is set **only** by `ControlTask`. |
| PWM-05 | [M] | Duty changes are applied in sync with the end of a PWM period (no truncated pulses). |
| PWM-06 | [M] | Configurable `duty_min` and `duty_max` limits. |
| PWM-07 | [M] | At startup PWM begins at 0 and ramps up smoothly (soft start). |
| PWM-08 | [M] | Any fault state (protection trip, invalid sensor, failed init) results in **duty = 0** and the **enable output LOW** (GAP-10). |
| PWM-09 | [S] | Manual mode (fixed duty) for testing and service, available from the web UI (admin, WEB-40), with all protections active. |
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
| GAP-02 | [M] | The gap duration is **configurable** in whole PWM periods (GAP-09): proposal 1–3 ms at 1 kHz, default 2 ms — to be confirmed by testing, see Q-07. |
| GAP-03 | [M] | The gap is **guaranteed** — the output is off for the entire gap regardless of the current duty, including duty = 100 %. |
| GAP-04 | [M] | The gap is synchronized with PWM: it starts at the end of a PWM period and never cuts a pulse in the middle. |
| GAP-05 | [M] | Gap generation **must not depend on any task or ISR** (including `ControlTask`) and must keep working during flash write/erase, CPU stalls and hangs. **Implementation (decided, Q-15):** pure hardware — a second LEDC channel ("window") at the gap rate, HIGH outside the gap and LOW during it (e.g. 18 ms / 2 ms), drives the driver's enable/shutdown input (HW-20). The driver passes PWM pulses only while the window is HIGH. A software/ISR gap was rejected: a failure there is invisible in normal operation (the heater keeps heating, only the gaps disappear). |
| GAP-06 | [M] | The reduction of the effective maximum duty caused by the gaps (e.g. 1 ms per 20 ms = −5 %) is taken into account in power calculations and in MPPT. |
| GAP-07 | [M] | The feature **cannot be disabled** from the user interface in `AUTO` mode (at most in a service mode with a clear warning). |
| GAP-08 | [S] | Validation: test with the real thermostat at maximum current to confirm the arc extinguishes at the chosen gap duration. |
| GAP-09 | [M] | **Phase lock:** both LEDC timers run from the same clock with an exact integer ratio (1 kHz / 50 Hz = 20) and are started together, so they stay in phase permanently. The gap starts at a PWM period boundary and lasts a whole number of PWM periods, so it always removes complete pulses (GAP-04). |
| GAP-10 | [M] | **Emergency off:** the window/enable output is also a second, independent off path. On any protection trip or fault `ControlTask` drives it LOW in addition to setting duty = 0. The MOSFET can conduct only when **both** PWM and enable are active. |
| GAP-11 | [S] | **Self-check:** `ControlTask` periodically verifies the LEDC configuration of both channels (frequency, window duty, phase). On a mismatch (e.g. window stuck HIGH = no gaps) → PWM off + fault. |

---

## 6. Panel voltage measurement

### 6.1 Requirements

| ID | Pr. | Requirement |
|---|---|---|
| VMEAS-01 | [M] | Panel voltage is measured through a resistive divider on an analog input. |
| VMEAS-02 | [M] | The ESP32-C6 calibrated ADC reading (eFuse calibration) is used, plus a user gain and offset. |
| VMEAS-03 | [M] | The measurement is averaged/filtered (many samples per cycle) to suppress PWM ripple. |
| VMEAS-04 | [S] | The ADC runs in **continuous (DMA) mode** (proposal: ≥ 20 kHz total sample rate). Every cycle `ControlTask` averages all samples of the 20 ms window; since the window spans an integer number of PWM periods, PWM ripple averages out exactly and no CPU time is spent on sampling. |
| VMEAS-05 | [M] | Range: from 0 to at least cold-weather Voc plus margin (proposal: 0–400 V). |
| VMEAS-06 | [M] | Invalid readings are detected (saturated ADC, open divider, implausible jumps) → PWM off + fault. |
| VMEAS-07 | [S] | Calibration from the web UI: the user enters a voltage measured with a multimeter and the controller computes the gain. |
| VMEAS-08 | [M] | The input circuit follows section 6.2: high-voltage resistor chain R1, bottom resistor R2, series resistor R3, dual Schottky clamp D1 (to +3V3 and to GND) and capacitor C1 at the ADC pin. |
| VMEAS-09 | [M] | R1 is a chain of **at least 4 resistors in series**, so that at maximum input voltage each one stays well below its rated working voltage (≈ 100 V each at 400 V; 1206 parts are typically rated 200 V), and a single shorted resistor cannot overload the clamp or the ADC pin. |
| VMEAS-10 | [M] | Full scale: **400 V at the input → ≈ 2.5 V at the ADC pin**, leaving margin below the top of the ADC range, where the ESP32-C6 ADC is least accurate. |
| VMEAS-11 | [M] | **Reverse polarity** is handled by the clamp diode to GND (D1b) together with the current limit of R1 (≈ 0.25 mA at 400 V). **No series diode** in the measurement path: it would add a non-linear, temperature-dependent drop of ≈ 0.5 V. |
| VMEAS-12 | [M] | R1 and R2: tolerance 1 % or better, temperature coefficient ≤ 100 ppm/K. The remaining gain error is removed by calibration (VMEAS-07). |
| VMEAS-13 | [M] | PCB: the high-voltage part (PV+ up to the R1/R2 node) keeps a creepage/clearance distance to all low-voltage circuitry (proposal ≥ 3 mm, to be confirmed against the applicable standard). R1 resistors are placed in a straight line. |
| VMEAS-14 | [S] | R3 + C1 with the divider form an anti-aliasing low-pass filter: τ ≈ 1.1 ms (f_c ≈ 145 Hz). It settles to 0.1 % in ≈ 7.5 ms, well within one 20 ms control cycle, so overvoltage is still detected within one cycle. |

### 6.2 Divider circuit

```
               R1 = 4 × 402 kΩ in series (1206, 1 %)              R3
 PV+ ───[402k]───[402k]───[402k]───[402k]───┬───[ 1 kΩ ]───┬────────────▶ ADC pin (A0)
                                            │              │
                                           ┌┴┐             ├──▶|── +3V3    D1a ┐
                                       R2  │ │ 10 kΩ       │                   ├ BAT54S
                                           └┴┘             ├──|◀── GND     D1b ┘
                                            │              │
                                            │             ═╪═ C1 100 nF
                                            │              │
 PV− / GND ─────────────────────────────────┴──────────────┴─────────────
```

**How it works (in plain words):**

- **R1 and R2 — the divider.** They split the panel voltage in a fixed ratio of about 1 : 162, so 400 V on the panels becomes about 2.5 V for the controller. R1 is split into 4 pieces because one small resistor cannot safely hold 400 V; each piece only carries about 100 V.
- **R3 — current limit.** It sits between the divider and the controller pin, so that only a tiny current can ever flow into the pin, even in a fault.
- **D1 (BAT54S, two Schottky diodes in one package) — the clamp.** If the voltage at the pin tries to go above 3.3 V (overvoltage) or below 0 V (panels connected with reversed polarity), the diodes conduct and hold the pin within about −0.3 V … +3.6 V. Schottky diodes are used because they conduct earlier than the protection diodes inside the ESP32, so they take the current instead of the chip.
- **C1 — filter and charge reservoir.** It smooths fast noise and ripple, and gives the ADC a stable charge to sample from.

**Component values:**

| Part | Value | Notes |
|---|---|---|
| R1a…R1d | 4 × 402 kΩ, 1206, 1 %, ≤ 100 ppm/K | total 1.608 MΩ; ≥ 200 V working voltage each |
| R2 | 10.0 kΩ, 0805/1206, 1 %, ≤ 100 ppm/K | |
| R3 | 1 kΩ | |
| D1 | BAT54S (dual series Schottky) | outer pins to GND and +3V3, common pin to the ADC — check the pinout in the datasheet |
| C1 | 100 nF, X7R, ≥ 16 V | as close to the ADC pin as possible |

**Calculations:**

| Quantity | Formula | Result |
|---|---|---|
| Divider ratio | R2 / (R1 + R2) = 10 k / 1618 k | 1 / 161.8 |
| ADC voltage at 230 V / 300 V / 400 V | V_in / 161.8 | 1.42 V / 1.85 V / 2.47 V |
| Divider current at 400 V | 400 V / 1.618 MΩ | 0.25 mA |
| Total power at 400 V | 400 V × 0.25 mA | 0.10 W (≈ 25 mW per R1 resistor) |
| Voltage across each R1 resistor at 400 V | 400 V / 4 | ≈ 100 V |
| Source impedance seen by the ADC side | R1 ∥ R2 | ≈ 9.94 kΩ |
| Filter time constant | (R1 ∥ R2 + R3) × C1 = 10.94 kΩ × 100 nF | ≈ 1.1 ms → f_c ≈ 145 Hz |
| Attenuation of 1 kHz PWM ripple | √(1 + (1000/145)²) | ≈ 7× (−17 dB) |
| Resolution (theoretical, 12-bit over ~3.3 V) | 0.8 mV × 161.8 | ≈ 0.13 V per step at the panels |
| Max clamp current (reverse polarity, 400 V) | 400 V / 1.608 MΩ | 0.25 mA — harmless |
| One R1 resistor shorted, 400 V | 400 V × 10 k / 1216 k | 3.29 V — clamped, current < 0.3 mA |
| Error from Schottky leakage (worst case ~1 µA) | 1 µA × 9.94 kΩ × 161.8 | ≈ 1.6 V at 400 V (0.4 %); typically much less |

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
| TEMP-04 | [M] | The DS18B20 is read by `TempTask`, **never by `ControlTask`** (see ARCH-15). The wait after a conversion request is at least the conversion time for the chosen resolution (750 ms at 12-bit, 375 ms at 11-bit). |
| TEMP-05 | [M] | Fault detection: missing sensor, CRC error, 85 °C power-on value, −127 °C, implausible jump. |
| TEMP-06 | [M] | If the boiler temperature is invalid or stale (ARCH-16) → **PWM off** + alarm. |
| TEMP-07 | [M] | Configurable **target temperature** and **hysteresis**: PWM stops when the target is reached and resumes when the temperature drops below `target − hysteresis`. |
| TEMP-08 | [M] | Separate **maximum (emergency) temperature** — above it PWM is turned off regardless of any other setting. |
| TEMP-09 | [S] | Temperature offset correction. |
| TEMP-10 | [M] | Boiler temperature read interval is configurable (range 1–60 s, **default 5 s**). Water heats slowly (≈ 0.4 °C/min at 2 kW in 80 L), so this is enough for regulation, and a short interval detects a disconnected sensor quickly. |
| TEMP-11 | [M] | On a CRC error the read is retried (proposal: up to 3 times) before the reading is counted as invalid. |
| TEMP-12 | [S] | The 1-Wire bus is driven by the **RMT peripheral** (hardware bit timing) instead of bit-banging with interrupts disabled, to avoid CRC errors caused by WiFi activity and jitter in other tasks. |

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
| PROT-11 | [M] | **Heatsink overtemperature:** derating and shutdown as defined in HSINK-08 and HSINK-09. |
| PROT-12 | [M] | **Heatsink sensor fault** → shutdown + fan forced on (see HSINK-10). |

---

## 11. Buzzer

**Non-critical module** — if it hangs, the core keeps running.

> **v0.7:** the OLED display and the buttons were removed. The **web UI (section 13) is the only user interface**; the buzzer stays for audible alarms. The removed requirements are kept in 11.2 as `[DROPPED]`, so their IDs are not reused.

### 11.1 Buzzer

| ID | Pr. | Requirement |
|---|---|---|
| UI-20 | [M] | Audible signal on alarm (overtemperature, sensor fault, protection trip). |
| UI-21 | [S] | ~~Short beep on button press.~~ **[DROPPED v0.7]** — no buttons. |
| UI-22 | [S] | The current alarm can be silenced from the web UI (admin, WEB-40). |
| UI-23 | [S] | The alarm sound can be disabled in the settings (admin); alarms are still shown in the web UI and written to the event log. |

### 11.2 Dropped requirements (OLED display and buttons)

| ID | Pr. | Requirement |
|---|---|---|
| UI-01 | [M] | ~~OLED main screen.~~ **[DROPPED v0.7]** |
| UI-02 | [M] | ~~OLED additional screens.~~ **[DROPPED v0.7]** |
| UI-03 | [S] | ~~OLED settings menu.~~ **[DROPPED v0.7]** |
| UI-04 | [S] | ~~OLED auto dimming.~~ **[DROPPED v0.7]** |
| UI-10 | [M] | ~~Button debounce.~~ **[DROPPED v0.7]** |
| UI-11 | [M] | ~~Short/long press detection.~~ **[DROPPED v0.7]** |
| UI-12 | [M] | ~~Up / Down / OK button meaning.~~ **[DROPPED v0.7]** |
| UI-13 | [S] | ~~Button hold for forced AP mode / factory reset.~~ **[DROPPED v0.7]** — replaced by WIFI-11. |

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
| WIFI-11 | [S] | **Recovery without buttons:** holding the XIAO's onboard **BOOT** button for ≥ 5 s at runtime forces AP mode; ≥ 15 s performs a factory reset (also clears the admin password, WEB-42). The same is available as a command over USB serial. (The button is only read at runtime — holding it during power-up enters USB download mode.) |

---

## 13. Web server and web UI

**Non-critical module.**

### 13.1 General

| ID | Pr. | Requirement |
|---|---|---|
| WEB-01 | [M] | Built-in web server, reachable in both STA and AP mode. |
| WEB-02 | [M] | All UI files (HTML/JS/CSS, chart library) are stored **on the controller** — no external CDNs, because there is no internet in AP mode. |
| WEB-03 | [S] | Files are stored compressed (gzip) for less space and faster loading, embedded in the firmware (FW-06). |
| WEB-04 | [M] | REST/JSON API for data and settings — the UI uses only this API. |
| WEB-05 | [M] | Settings are protected by an admin password or PIN — see section 13.5. |
| WEB-06 | [S] | The UI works well on a phone. |

### 13.2 Live data

| ID | Pr. | Requirement |
|---|---|---|
| WEB-10 | [M] | Live data: V, I, P, T, duty, mode, status, alarms, energy today, heatsink temperature, fan state. |
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
| WEB-31 | [M] | Period selection (e.g. last hour, 24 hours, by day). The data source is chosen automatically: the last hour from the 1-second RAM buffer (HIST-02), longer periods from the flash tiers (HIST-10). |
| WEB-32 | [S] | Daily energy chart (bars, kWh/day). |
| WEB-33 | [S] | Event log (alarms, protection trips, reboots). |
| WEB-34 | [C] | Download history as CSV. |

### 13.5 Access control

| ID | Pr. | Requirement |
|---|---|---|
| WEB-40 | [M] | Two access levels. **Viewer** (no login): live data, charts, history, event log — read only. **Admin** (password or PIN): everything that changes state — settings, mode and manual duty, calibration, WiFi, history intervals, fan test, alarm silence, reboot, factory reset. |
| WEB-41 | [M] | Access is enforced **on the controller** for every API call that changes state; hiding buttons in the browser is not protection. |
| WEB-42 | [M] | No default password. On first start (no admin password stored) the setup page requires creating one before any other setting can be changed. A factory reset clears it. |
| WEB-43 | [M] | Password/PIN: minimum 4 characters; stored only as a salted hash in NVS, never as plain text. |
| WEB-44 | [M] | After login the browser gets a session token (cookie) that expires after inactivity (proposal 15 min); explicit logout is available. |
| WEB-45 | [S] | After 5 failed login attempts further attempts are blocked for 1 minute (protection against guessing a short PIN). |
| WEB-46 | [C] | Because the UI uses plain HTTP (FW-05), the password could be sniffed on the local network; a challenge-response login (password never sent in plain text) can be added later. |

---

## 14. Data history

**Non-critical module.** The core only publishes the snapshot; history is collected by a separate task.

| ID | Pr. | Requirement |
|---|---|---|
| HIST-01 | [M] | History is kept in the controller's memory. |
| HIST-02 | [M] | **Detailed history in RAM — the last 60 minutes at 1-second resolution.** Each sample is the average of the 50 control cycles of that second (V, I, P, T, heatsink T, duty, fan, flags; 16 B → ≈ 56 KB of RAM). This buffer is not saved to flash and starts empty after a reboot (HIST-04). |
| HIST-03 | [M] | **Long-term history in flash** (LittleFS data partition, FW-11) — stored in tiers with decreasing resolution, see HIST-10. |
| HIST-04 | [S] | ~~Short-term history is periodically saved to flash.~~ **[DROPPED v0.7]** — the detailed RAM hour is not persisted (it would wear the flash for little value); everything older is in the flash tiers (HIST-10). |
| HIST-05 | [M] | **Event log** (ring, in flash): alarms, protection trips, reboots and their reason, settings changes. |
| HIST-06 | [M] | Records are timestamped. Without synchronized time (no NTP), relative time since boot is stored and corrected once time becomes available. |
| HIST-07 | [M] | Lifetime energy counter (kWh) — not reset on reboot. |
| HIST-08 | [M] | Full history storage never stops anything — the oldest records are overwritten. |
| HIST-09 | [M] | History and logs are stored as **fixed-size binary records** (values in integer units, e.g. 0.1 V, 0.1 °C) — no text, JSON or CSV in flash. Conversion to JSON/CSV happens only when data is sent to the web UI. |
| HIST-10 | [M] | **Two configurable flash tiers + daily summaries.** Tier 1 "fine" (default **5 min**, choice 1–60 min) and tier 2 "coarse" (default **60 min**, choice 15–240 min, longer than tier 1), each with avg/min/max per interval, plus fixed daily summaries. Each tier has a fixed share of the data partition; retention = share / record size × interval, so a longer interval gives a longer history (tables below). A larger partition automatically gives a longer history. |
| HIST-11 | [M] | Writes are **batched in RAM** and flushed at most every 10 minutes (configurable) or when a flash page fills. The maximum data loss on a power cut is one flush interval. The lifetime energy counter is saved separately in NVS (e.g. every 15 min or every 0.1 kWh). |
| HIST-12 | [S] | **Debug/text log:** ring file with a size cap (proposal 128 KB). Only Warning/Error are stored by default; the level can be changed at runtime. The log can be viewed and downloaded from the web UI. |
| HIST-13 | [S] | With the planned write rates the data partition must last **≥ 20 years** (flash endurance ≈ 100 000 erase cycles per sector, LittleFS wear levelling). |
| HIST-14 | [M] | The tier intervals are set from the web UI (admin). Before saving, the UI shows the resulting retention of each tier. |
| HIST-15 | [M] | Changing an interval does **not delete** existing history: every record stores its own start time and interval length, so old and new records coexist and the old ones age out normally. |
| HIST-16 | [M] | Every tier is computed from the 1-second RAM samples with its own accumulator: avg/min/max of V, I, P, T and heatsink T, energy (Wh) of the interval, average duty and fan on-time. |

**Storage budget (proposal, 2.375 MB data partition, default intervals):**

| Item | Record | Interval | Space | Retention |
|---|---|---|---|---|
| Detailed hour (RAM only, not in flash) | 16 B | 1 s | ≈ 56 KB RAM | 60 min |
| Tier 1 — fine (avg/min/max) | 40 B | 5 min | ≈ 790 KB | ≈ 70 days |
| Tier 2 — coarse (avg/min/max) | 40 B | 60 min | ≈ 790 KB | ≈ 2.3 years |
| Daily summaries (energy, max T, max P, hours) | 32 B | 1 day | ≈ 115 KB | 10 years |
| Event log | 16 B | on event | 128 KB | 8 192 events |
| Debug/text log | text | on event | 128 KB | ring |
| Free for LittleFS | | | ≈ 485 KB (20 %) | |

**Retention of one tier by interval** (≈ 20 000 records per tier):

| Interval | 1 min | 5 min | 10 min | 15 min | 30 min | 60 min | 120 min | 240 min |
|---|---|---|---|---|---|---|---|---|
| Retention | 14 days | 70 days | 140 days | 210 days | 1.2 years | 2.3 years | 4.6 years | 9.2 years |
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
| Temperature | sensor type, target T, hysteresis, T_max, offset, read interval, staleness limit |
| Voltage | gain/calibration, V_min, V_max |
| Current | sensitivity, offset, I_max |
| Heatsink / fan | sensor type, read interval, fan on T, fan off T, fan minimum on time, derating start T, heatsink T_max, re-enable hysteresis, offset |
| MPPT | *(to be added with the algorithm description)* |
| WiFi | SSID, password, connect timeout, AP password, hostname |
| Web | admin password/PIN, session timeout |
| Buzzer | alarm sound on/off |
| History | tier 1 interval, tier 2 interval, flush interval |

---

## 16. MOSFET heatsink thermal management

**Criticality:** CRITICAL — runs in `ControlTask`, because an overheated MOSFET can fail shorted and keep the heater on permanently.

### 16.1 Description

A temperature sensor on the MOSFET heatsink and an on/off fan cooling it. The fan is switched on and off by heatsink temperature with hysteresis. If the heatsink still gets too hot, the controller first reduces the maximum duty (derating) and then turns PWM off completely.

```
 heatsink T ─────────────────────────────────────────────────────────────▶
        fan OFF │ hysteresis │ fan ON     │ derating          │ SHUTDOWN
                ▲            ▲            ▲                   ▲
          fan off T     fan on T     derating start T    heatsink T_max
          (e.g. 40 °C)  (e.g. 50 °C)  (e.g. 70 °C)        (e.g. 85 °C)
```

### 16.2 Requirements

| ID | Pr. | Requirement |
|---|---|---|
| HSINK-01 | [M] | The heatsink temperature is measured by `TempTask` (DS18B20) or by `ControlTask` (NTC on ADC), with the same freshness rule (ARCH-16). Read interval configurable, **default 2 s** (the heatsink heats up much faster than the water). |
| HSINK-02 | [M] | The heatsink sensor is on **its own pin**, separate from the boiler sensor, so a fault on one sensor's wiring cannot take down the other. Sensor type (DS18B20 or NTC on ADC) — see Q-13. |
| HSINK-03 | [M] | Sensor fault detection as in TEMP-05 (missing, CRC error, out of range, implausible jump). |
| HSINK-04 | [M] | The fan is controlled by heatsink temperature with hysteresis: **on** at `fan on T` (proposal 50 °C), **off** below `fan off T` (proposal 40 °C). |
| HSINK-05 | [M] | Minimum fan on time (proposal 60 s), to avoid frequent switching. |
| HSINK-06 | [M] | The fan keeps running after PWM stops for as long as the heatsink is above `fan off T` (the heatsink stays hot after the load is removed). |
| HSINK-07 | [M] | The fan output is off at boot (HW-18) and is controlled only by `ControlTask`. |
| HSINK-08 | [M] | **Derating:** above `derating start T` (proposal 70 °C) the maximum allowed duty is reduced linearly, reaching 0 at `heatsink T_max`. |
| HSINK-09 | [M] | **Shutdown:** at or above `heatsink T_max` (proposal 85 °C) → PWM off + alarm + event log entry. PWM is allowed again only after the heatsink cools below `heatsink T_max − hysteresis`. |
| HSINK-10 | [M] | **Sensor fault:** if the heatsink temperature is invalid for longer than a configurable time → PWM off, fan **forced on**, alarm. (The core either works with a valid heatsink temperature or stays off.) |
| HSINK-11 | [S] | Fan test from the web UI (admin; forced on for a set time), without disabling any protection. |
| HSINK-12 | [S] | Heatsink temperature and fan state are published in the snapshot (web, history). |
| HSINK-13 | [C] | Fan failure plausibility check: if the fan has been on for a set time and the heatsink temperature keeps rising at an unchanged load → warning. |
| HSINK-14 | [C] | Fan run-hours counter (for maintenance). |

---

## 17. Firmware size and flash layout

### 17.1 Description

Goals: **the smallest possible compiled firmware**, **no OTA**, and **as much flash as possible for history and logs**. The XIAO ESP32-C6 has 4 MB of flash. Without OTA only one app slot is needed, and everything the app does not use goes to the data partition.

### 17.2 Firmware size

| ID | Pr. | Requirement |
|---|---|---|
| FW-01 | [M] | **Minimal binary size** is a design criterion: every library and feature is chosen with its flash cost in mind, and the lighter option is preferred when it meets the requirements. |
| FW-02 | [M] | Compiler optimization for size (`-Os`); assertions in silent mode in release builds. |
| FW-03 | [M] | Compile-time maximum log level **Warning** in release builds; Info/Debug/Verbose messages are compiled in only in debug builds. |
| FW-04 | [M] | Unused components are disabled: **Bluetooth/BLE, IEEE 802.15.4 (Zigbee/Thread), WPS, WPA-Enterprise, mesh, HTTPS/TLS server**, and any driver that is not used — as far as the chosen framework allows (see Q-11). |
| FW-05 | [M] | The web UI is served over **plain HTTP** on the local network; no TLS. |
| FW-06 | [M] | Web UI files are **minified + gzip-compressed and embedded in the firmware image**, so firmware and UI always match and a firmware update never touches the data partition. A lightweight chart library is used (e.g. uPlot, ≈ 50 KB minified, instead of Chart.js, ≈ 200 KB). |
| FW-07 | [S] | `printf` family: newlib-nano formatting where possible; numbers for JSON are formatted with fixed-point helpers instead of float `printf`. |
| FW-08 | [S] | Binary size is reported on every build (per component) and recorded for every release; an increase of more than 5 % must be justified. |

### 17.3 Flash layout

| ID | Pr. | Requirement |
|---|---|---|
| FW-09 | [M] | **No OTA:** a single `factory` app partition, no `otadata`, no second app slot, no OTA code in the firmware. |
| FW-10 | [M] | Firmware is updated **over USB only**, and only with the panels disconnected (DC isolator open, HW-26): the MCU ground is PV− (HW-16, HW-25). |
| FW-11 | [M] | The app partition is sized to the measured binary + **≥ 20 % margin**, rounded up to 64 KB. The **data partition takes all remaining flash**. |
| FW-12 | [M] | Flashing new firmware **does not erase** NVS or the data partition — settings and history survive updates. A full erase happens only on an explicit factory reset or an explicit flashing-tool option. |
| FW-13 | [C] | A 64 KB `coredump` partition for crash analysis; the crash reason after a reboot is written to the event log and shown in the web UI. |

**Proposed partition table (4 MB flash, app size to be adjusted per FW-11):**

| Name | Type / subtype | Offset | Size | Purpose |
|---|---|---|---|---|
| *(bootloader)* | — | 0x0 | 32 KB | second-stage bootloader |
| *(partition table)* | — | 0x8000 | 4 KB | |
| `nvs` | data / nvs | 0x9000 | 28 KB | settings, WiFi credentials, lifetime counters |
| `factory` | app / factory | 0x10000 | 1.5 MB | firmware (example size) |
| `coredump` | data / coredump | 0x190000 | 64 KB | optional (FW-13) |
| `storage` | data / littlefs | 0x1A0000 | **2.375 MB** | history and logs (section 14) |

---

## 18. Open questions

| ID | Question |
|---|---|
| Q-01 | What exactly is the MPPT algorithm? *(to be described)* |
| Q-02 | What are the heater power and the solar array parameters (number of panels, Vmp, Voc, Imp)? Needed to check SYS-03 and for sizing. |
| Q-03 | Thermocouple — which type (K?) and which amplifier? Analog (e.g. AD8495) or digital (MAX31855/MAX6675 over SPI — that would not be an analog input)? |
| Q-04 | Is the buzzer active (on/off only) or passive (different tones via PWM)? |
| Q-05 | ~~Exact OLED model?~~ **Obsolete v0.7:** no display. |
| Q-06 | How is the controller powered, and is there galvanic isolation between the MCU and the DC power circuit? |
| Q-07 | What gap duration is sufficient to extinguish the arc in the actual thermostat at maximum current? |
| Q-08 | Hall sensor — on the panel side (average current, better for MPPT) or the heater side (pulsed current)? Model and range? |
| Q-09 | If the current sensor fails — shut everything down, or continue in a limited voltage-only mode? |
| Q-10 | ~~Are OTA firmware updates via web needed?~~ **Resolved v0.6:** no OTA (FW-09, FW-10). |
| Q-11 | Arduino framework or pure ESP-IDF? Note: the size goal (FW-01, FW-04) needs a framework whose configuration (`sdkconfig`) can be trimmed — pure ESP-IDF or Arduino as an ESP-IDF component; the precompiled Arduino core cannot remove unused components. |
| Q-12 | Which exact GPIOs are the two free pins, and which two GPIOs are wired to the driver on the board? (All 11 header pins are already used in the table in 2.4, so the table must be checked against the actual board schematic.) Note v0.7: removing the OLED and buttons freed D4, D5, D7, D8 and D9. |
| Q-13 | Heatsink sensor type: DS18B20 (any digital pin) or NTC thermistor (needs an ADC-capable pin)? |
| Q-14 | Fan type and supply voltage (e.g. 12 V DC fan)? Is a tachometer signal needed later? |
| Q-15 | ~~AC gap implementation: (A) IRAM-safe ISR or (B) hardware gate with a second LEDC channel?~~ **Resolved v0.4:** (B), via the driver's enable input (GAP-05, HW-20). |
| Q-16 | Which MOSFET driver: IR2104/IR2184 (drop-in for the current footprint, LO output → inverted logic, HW-22), a dedicated low-side driver such as UCC27517 / UCC27524A (non-inverting, simpler for a low-side MOSFET, HW-25), or an isolated one (depends on Q-06)? |

---

## 19. Change log

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-10-18 | Initial draft: concept, hardware, architecture, PWM, AC simulation, measurements, protections, UI, WiFi, web, history, settings. |
| 0.2 | 2026-10-18 | Added section 16 — MOSFET heatsink thermal management (heatsink sensor + on/off fan): HW-09, HW-17, HW-18, HSINK-01…14, PROT-11, PROT-12, Q-12…14; updated ARCH-04, UI-02, WEB-10, HIST-02, settings list, pin table. Open questions → 17, change log → 18. |
| 0.3 | 2026-10-18 | Timing and task design: 20 ms GPTimer tick waking `ControlTask`, separate `TempTask` for temperature sensors with freshness check, ADC DMA averaging, gap independent of tasks and flash stalls. Added ARCH-11…16, TEMP-10…12, Q-15; updated ARCH-01, 02, 04, 07, 10, 20, GAP-05, VMEAS-04, TEMP-04, TEMP-06, HSINK-01, architecture diagram, settings list. |
| 0.4 | 2026-10-18 | MOSFET driver: new section 2.3 (HW-19…24), pin assignment moved to 2.4; IR2151 found unsuitable. AC gap decided as pure hardware via a second LEDC channel on the driver's enable input (Q-15 resolved); added GAP-09…11, Q-16; updated HW-02, HW-10, HW-11, ARCH-04, PWM-03, PWM-08, GAP-02, GAP-05, Q-12, pin table, diagram. |
| 0.5 | 2026-10-18 | Panel voltage divider: section 6 split into 6.1 requirements and 6.2 circuit (schematic, plain-language description, values, calculations); added VMEAS-08…14. MOSFET is low-side N-channel: added HW-25, HW-26; updated HW-15, Q-16. |
| 0.6 | 2026-10-18 | Added section 17 — firmware size and flash layout (FW-01…13, partition table, no OTA, Q-10 resolved). History: tiered binary storage with storage budget (HIST-09…13); updated HIST-03, HIST-04, WEB-03, Q-11. Open questions → 18, change log → 19. |
| 0.7 | 2026-10-18 | History: detailed last hour in RAM at 1 s, two user-configurable flash tiers + daily summaries (HIST-02, HIST-10 updated; HIST-04 dropped; HIST-14…16 added; new storage/retention tables). OLED display and buttons removed (HW-07, HW-08, ARCH-24, UI-01…04, UI-10…13, UI-21 dropped; pins D4, D5, D7, D8, D9 freed; section 11 is now Buzzer; UI-23, WIFI-11 added). Web access control: viewer without login, admin with password/PIN (section 13.5, WEB-40…46; WEB-05 now [M]). Updated ARCH-03, PWM-09, WEB-31, HSINK-11, HSINK-12, FW-07, Q-05, Q-12, settings list, diagram. |

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

After adding: list the feature in the [Feature list](#feature-list), record the change in the [Change log](#19-change-log), and mirror everything in [requirementsBG.md](requirementsBG.md) with the same IDs.
