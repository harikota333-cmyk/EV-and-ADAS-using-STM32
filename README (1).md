<div align="center">

# 🚗 Real-Time EV Dashboard & ADAS Warning System using STM32

**Embedded firmware for an Electric Vehicle controller with ultrasonic collision / blind-spot warning and a live Python dashboard over UART**

![MCU](https://img.shields.io/badge/MCU-STM32F103C8T6-03234B?logo=stmicroelectronics&logoColor=white)
![Language](https://img.shields.io/badge/Firmware-Embedded%20C-00599C?logo=c&logoColor=white)
![IDE](https://img.shields.io/badge/IDE-STM32CubeIDE-03234B)
![Library](https://img.shields.io/badge/Library-STM32%20HAL-blue)
![Simulator](https://img.shields.io/badge/Simulator-PICSimLab-orange)
![Dashboard](https://img.shields.io/badge/Dashboard-Python-3776AB?logo=python&logoColor=white)
![Tests](https://img.shields.io/badge/Test%20Cases-17%2F17%20Passed-brightgreen)

</div>

---

## 📑 Table of Contents
1. [Overview](#-overview)
2. [Objectives](#-objectives)
3. [Hardware & Software](#-hardware--software)
4. [System Architecture](#-system-architecture)
5. [Pin & Peripheral Mapping](#-pin--peripheral-mapping)
6. [Working Principle](#-working-principle-step-by-step)
7. [Implemented Features](#-implemented-features)
8. [UART Telemetry Protocol](#-uart-telemetry-protocol)
9. [Repository Structure](#-repository-structure)
10. [Setup & Execution](#-setup--execution)
11. [Testing & Results](#-testing--results)
12. [Screenshots](#-screenshots)
13. [Real-World Applications](#-real-world-applications)
14. [Learning Outcomes](#-learning-outcomes)
15. [Future Improvements](#-future-improvements)
16. [Author](#-author)

---

## 📌 Overview

This project is a **Real-Time Electric Vehicle (EV) Dashboard and Advanced Driver Assistance System (ADAS) Warning System** built on the **STM32F103C8T6 ("Blue Pill")** microcontroller. It was developed as the capstone of a 4-week online Embedded Systems internship at **Emertxe Information Technologies, Bangalore** (Cohort EI26_016, 2 June 2026 – 3 July 2026).

The firmware combines:
- a **physics-based EV controller** (speed, state-of-charge, torque, regenerative braking, range),
- an **ultrasonic ADAS engine** (collision warning, blind-spot detection, parking assist),
- a **five-state vehicle state machine** with a **fault manager**, and
- a **CRC16-protected UART telemetry link** feeding a **live Python dashboard**.

> ℹ️ The project was developed and validated in the **PICSimLab** hardware simulator using virtual LEDs, potentiometers and HC-SR04 sensors. It has **not** been deployed on physical hardware.

---

## 🎯 Objectives

1. Gain practical experience in STM32 firmware development using **STM32CubeIDE** and the **HAL library**.
2. Implement a physics-based **EV controller** together with an **ultrasonic-sensor-driven ADAS** collision and blind-spot warning engine.
3. Build a **real-time UART-connected Python dashboard** for live vehicle and safety data.
4. Meet real-time targets: **control loop < 5 ms** (NFR-01) and **sensor read < 100 ms** (NFR-02).
5. Write modular firmware and validate it against a structured **17-test-case plan**.

---

## 🧰 Hardware & Software

### Hardware (simulated in PICSimLab)

| Component | Qty | Purpose |
|---|---|---|
| STM32F103C8T6 (Blue Pill), Cortex-M3 @ 72 MHz, 64 KB Flash, 20 KB SRAM | 1 | Main controller |
| HC-SR04 ultrasonic sensor | 3 | Front, left, right distance sensing |
| Potentiometers | 4 | Accelerator, brake, state-of-charge, motor temperature |
| Status LEDs | — | Vehicle / alarm / fault indication (PB8–PB11) |
| Buzzer and motor-drive output | — | Audible alerts and motor PWM (TIM4) |

### Software

| Tool | Use |
|---|---|
| STM32CubeIDE + STM32CubeMX | Project config, code generation, build |
| STM32 HAL drivers | Peripheral access |
| PICSimLab | Hardware simulation (loads the compiled `.hex`) |
| ST-Link / SWD | Flashing / debugging, as used in the internship workflow |
| Python 3 with `pyserial`, `matplotlib` | Real-time dashboard and test scripts |

---

## 🏗️ System Architecture

```
 ┌──────────────────────┐      ┌────────────────────────────────┐      ┌───────────────────────┐
 │  PERCEPTION          │      │  PROCESSING (STM32F103C8T6)    │      │  ACTUATION            │
 │  • 4 potentiometers  │ ───► │  • EV controller               │ ───► │  • Status LEDs        │
 │  • 3× HC-SR04        │      │  • ADAS engine                 │      │  • Buzzer             │
 └──────────────────────┘      │  • Fault manager               │      │  • Motor PWM          │
                               │  • Vehicle state machine       │      └───────────────────────┘
                               └───────────────┬────────────────┘
                                               │ USART1 @ 115200 bps (DMA)
                                               ▼
                               ┌────────────────────────────────┐
                               │  HOST: Python dashboard        │
                               │  (pyserial + matplotlib)       │
                               └────────────────────────────────┘
```

---

## 🔌 Pin & Peripheral Mapping

| Function | Pin(s) / Peripheral | Notes |
|---|---|---|
| Status LEDs | PB8 – PB11 | GPIO outputs |
| ADC inputs | PA0 – PA3 (ADC1, 12-bit) | Accelerator, brake, SOC, motor temperature |
| HC-SR04 Front | TRIG PB0, ECHO PB1 | |
| HC-SR04 Left | TRIG PB2, ECHO PB3 | |
| HC-SR04 Right | TRIG PB4, ECHO PB5 | |
| Microsecond timer | TIM2 | 1 MHz free-running counter for echo timing |
| Control-loop tick | TIM1 | Interrupt every **10 ms** |
| Sensor scheduler tick | TIM3 | Interrupt every **100 ms** |
| Buzzer / motor PWM | TIM4 | Motor PWM 20 kHz on PA8 |
| Telemetry | USART1, 115200 bps, DMA TX | |

---

## ⚙️ Working Principle (Step by Step)

1. **Initialisation** – HAL, clocks, GPIO, ADC1, TIM1–TIM4, USART1 (DMA) and all modules are initialised.
2. **Cooperative scheduling (ISR-flag pattern)** – TIM1 (10 ms) and TIM3 (100 ms) ISRs only set flags; `main()` checks and clears them, keeping ISRs short and timing deterministic.
3. **Input acquisition** – ADC1 reads the accelerator, brake, SOC and motor-temperature potentiometers; counts are linearly scaled to % pedal travel, % SOC and °C.
4. **Ultrasonic ranging (every 100 ms)** – For each sensor, a 10 µs pulse is sent on TRIG; TIM2 measures the ECHO pulse width and
   `distance_cm = (pulse_us × 0.0343) / 2`, clamped to **2–400 cm** with a **30 ms timeout** for a disconnected sensor. The three sensors are triggered sequentially (50 µs gap) to avoid cross-talk.
5. **EV control (every 10 ms)** – Speed is computed with an inertia model from accelerator/brake inputs; SOC is updated by energy integration; range is estimated from SOC, battery capacity and drive mode. Brake above 5 % applies regenerative (negative) torque, down to −80 Nm.
6. **ADAS evaluation** – Time-To-Collision (TTC) is computed from front distance and speed and compared with the WARNING / CRITICAL thresholds. Side sensors drive blind-spot detection; low-speed parking assist uses all three sensors. A 300 ms hysteresis (three consecutive readings) prevents alarm flicker.
7. **State machine & fault handling** – The vehicle FSM updates from pedal thresholds. A CRITICAL collision or any other fault flag forces the **FAULT** state and **cuts motor PWM immediately**.
8. **Outputs** – LEDs and buzzer are driven according to alarm priority (P0 none … P3 advisory).
9. **Telemetry** – Packets are framed with a sync word and CRC16 and sent over USART1 via DMA.
10. **Dashboard** – The Python host verifies sync + CRC16, decodes the packets and refreshes four live panels at 10 Hz.

---

## ✨ Implemented Features

### 🔋 EV Controller (`ev_control.c/.h`)
- Physics-based inertia speed model, clamped to **200 km/h**
- SOC estimation by energy integration (power × time)
- Estimated driving range from SOC, battery capacity and drive mode
- Drive modes: **ECO (0.6× torque)**, **NORMAL (1.0×)**, **SPORT (1.3×)**
- **Regenerative braking** when brake > 5 % (down to −80 Nm), energy returned to SOC

### 🛡️ ADAS Engine (`adas.c/.h`)

| Condition | Level |
|---|---|
| Front distance < 50 cm **or** TTC < 3 s | ⚠️ WARNING |
| Front distance < 20 cm **or** TTC < 1.5 s | 🚨 CRITICAL (triggers `FAULT_COL`) |
| Side sensor < 30 cm at speed > 20 km/h | Blind-spot alert |
| Speed < 10 km/h | Parking-assist mode (all 3 sensors active) |

```c
// Collision alert classification (ADAS engine)
float ttc_s = (speed_kmh > 1.0f) ? (front_cm / 100.0f) / (speed_kmh / 3.6f) : 99.0f;

if (front_cm < 20.0f || ttc_s < 1.5f) {
    collision_level = ADAS_CRITICAL;   // -> triggers FAULT_COL
} else if (front_cm < 50.0f || ttc_s < 3.0f) {
    collision_level = ADAS_WARNING;
} else {
    collision_level = ADAS_NONE;
}
```

### 🚦 Vehicle State Machine & Fault Manager (`fault.c/.h`)
- States: `PARKED → READY → DRIVING ⇄ REGEN`, and `FAULT` (fault clear → `PARKED`)
- Fault flags: `FAULT_OT` (over-temperature), `FAULT_SOC` (low SOC), `FAULT_COL` (collision), `FAULT_SEN` (sensor failure), `FAULT_COM` (communication loss)
- Entering `FAULT` cuts motor PWM immediately

### 📡 HC-SR04 Driver
```c
// Distance calculation from HC-SR04 echo pulse width
float HCSR04_CalcDistance(uint32_t pulse_us) {
    float distance_cm = (pulse_us * 0.0343f) / 2.0f;
    if (distance_cm < 2.0f)   distance_cm = 2.0f;
    if (distance_cm > 400.0f) distance_cm = 400.0f;
    return distance_cm;
}
```
`HCSR04_ReadAll()` samples all three sensors without blocking the 10 ms control loop.

### 💻 UART Command Shell
Host commands: `mode`, `speed set`, `soc set`, `obstacle`, `fault inject`, `fault clear`, `status`, `reset` — used to drive test scenarios from a serial terminal or Python scripts without changing PICSimLab components manually.

### 📊 Python Real-Time Dashboard (10 Hz)
- Speedometer arc gauge (0–200 km/h, green → amber → red)
- Battery SOC bar with estimated range
- Metrics panel: power, motor temperature, torque, drive-mode badge
- ADAS bird's-eye view: ego vehicle, colour-coded obstacle arcs, blind-spot indicators, full-width FAULT banner

---

## 📡 UART Telemetry Protocol

USART1 · 115200 bps · DMA transmit

```
| SYNC 0xAA55 | ID 0x01–0x04 | LEN | PAYLOAD | CRC16 |
```

| ID | Packet | Content |
|---|---|---|
| `0x01` | EV Metrics | speed, SOC, torque, power, range (five 32-bit floats) |
| `0x02` | ADAS Alerts | sensor distances, collision level, blind-spot flags, TTC |
| `0x03` | Vehicle State | sent on state change |
| `0x04` | Acknowledgement | sent on request |

---

## 📁 Repository Structure

> ⚠️ Adjust names to match your final files (see notes in the chat reply).

```
EV-ADAS-STM32/
├── README.md
├── Firmware/                      # STM32CubeIDE project
│   ├── <project_name>.ioc         # CubeMX configuration
│   ├── Core/
│   │   ├── Inc/
│   │   │   ├── ev_control.h
│   │   │   ├── adas.h
│   │   │   ├── fault.h
│   │   │   └── (ultrasonic driver + UART shell headers)
│   │   └── Src/
│   │       ├── main.c
│   │       ├── ev_control.c
│   │       ├── adas.c
│   │       ├── fault.c
│   │       └── (ultrasonic driver + UART shell sources)
│   └── Debug/<project_name>.hex   # firmware loaded into PICSimLab
├── Simulation/
│   └── PICSimLab/                 # saved PICSimLab configuration
├── Dashboard/
│   └── (Python dashboard script)
├── Tests/
│   └── (test scripts / test report, TC-01 … TC-17)
├── Docs/
│   └── Internship_Report_Hari_Kota.pdf
└── Images/
    ├── dashboard_output.png
    ├── picsimlab_setup.png
    ├── architecture.png
    └── state_machine.png
```

---

## 🚀 Setup & Execution

### Prerequisites
- STM32CubeIDE
- PICSimLab (STM32F103 Blue Pill board)
- Python 3 with:
  ```bash
  pip install pyserial matplotlib
  ```

### 1. Clone
```bash
git clone https://github.com/harikota333-cmyk/<repo-name>.git
cd <repo-name>
```

### 2. Build the firmware
1. Open **STM32CubeIDE** → *File → Open Projects from File System* → select `Firmware/`.
2. Build with **Project → Build All** to generate the `.hex` file.

### 3. Run in PICSimLab
1. Open PICSimLab, select the **STM32F103 Blue Pill** board, and load your saved configuration from `Simulation/PICSimLab/`.
2. Load the generated `.hex` file.
3. Confirm the virtual parts are wired as in the [pin table](#-pin--peripheral-mapping): 3× HC-SR04, 4 potentiometers, LEDs on PB8–PB11.

### 4. Run the dashboard
```bash
cd Dashboard
python <dashboard_script>.py
```
Set the serial port in the script to the port connected to USART1 (115200 bps).

### 5. Try it
- Move the accelerator / brake potentiometers → speed, SOC and regen behaviour change.
- Reduce a front sensor distance below 50 cm / 20 cm → WARNING / CRITICAL → FAULT.
- Use shell commands such as `fault inject`, `fault clear`, `status` over the serial terminal.

---

## ✅ Testing & Results

Validated against a 17-test-case plan (TC-01 to TC-17):

| Test Case | Description | Result |
|---|---|---|
| TC-01 – TC-06 | Sensor accuracy, collision WARNING/CRITICAL, blind-spot triggers | ✅ Pass |
| TC-07 – TC-09 | Fault-state entry, PWM cut-off, drive-mode torque scaling | ✅ Pass |
| TC-10 – TC-12 | Regenerative braking, watchdog reset, EV loop timing (NFR-01) | ✅ Pass |
| TC-13 | UART packet integrity — 0 CRC errors over 1000 packets | ✅ Pass |
| TC-14 | Sensor timeout — ECHO disconnected → 400 cm, valid = 0 | ✅ Pass |
| TC-15 | Fault clear recovery — FAULT → PARKED, all LEDs off | ✅ Pass |
| TC-16 | Alarm hysteresis — no rapid toggling near 50 cm | ✅ Pass |
| TC-17 | Dashboard end-to-end refresh latency (< 150 ms) | ✅ Pass |

**Timing targets:** control loop < 5 ms (NFR-01) and sensor read < 100 ms (NFR-02) confirmed by direct timing measurement during integration testing.

---

## 🖼️ Screenshots

| PICSimLab Setup | Python Dashboard Output |
|---|---|
| ![PICSimLab](Images/picsimlab_setup.png) | ![Dashboard](Images/dashboard_output.png) |

| System Architecture | Vehicle State Machine |
|---|---|
| ![Architecture](Images/architecture.png) | ![State Machine](Images/state_machine.png) |

### Hardware Images
> 📷 *Not applicable — project was validated in simulation only. Add photos here if you later build it on a physical Blue Pill.*

---

## 🌍 Real-World Applications

- **EV instrument clusters** – live speed, SOC, range and fault indication
- **ADAS functions** – forward-collision warning, blind-spot detection, parking assist
- **Functional-safety patterns** – fault detection and safe-state entry (motor cut-off)
- **Regenerative braking** logic for energy recovery
- **Teaching / prototyping** automotive embedded concepts in a simulation-first workflow

---

## 🎓 Learning Outcomes

- STM32 peripheral configuration (GPIO, ADC, Timers, USART + DMA) with CubeMX and HAL
- Microsecond-accurate ultrasonic ranging with a hardware timer
- Interrupt-driven cooperative scheduling with the ISR-flag pattern
- Finite state machine and fault-manager design for safety behaviour
- Designing a framed, CRC16-checked serial protocol
- Building a real-time Python visualisation dashboard
- Requirement-driven testing (functional + timing) and simulation-based validation

---

## 🔮 Future Improvements

*(Directions suggested in the internship report conclusion.)*
- Deployment on **real hardware** with physical sensors
- **CAN bus** integration
- A **mobile dashboard**

---

## 👤 Author

**Hari Kota**
🎓 B.Tech – Electronics and Communication Engineering
🏫 Parul University (Parul Institute of Engineering and Technology)
🔗 GitHub: [harikota333-cmyk](https://github.com/harikota333-cmyk)
🔗 LinkedIn: [kotahari](https://linkedin.com/in/kotahari)

---

<div align="center">⭐ If you found this project useful, consider giving it a star! ⭐</div>
