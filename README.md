# 🔬 UV-C-Sterilization-System-for-NICU-beds
### Medical-Grade Closed-Loop NICU Sanitization Platform

[![Platform](https://img.shields.io/badge/Platform-ESP32-blue)](https://www.espressif.com/)
[![Language](https://img.shields.io/badge/Language-C%2B%2B%20%7C%20Arduino-orange)](https://www.arduino.cc/)
[![Protocol](https://img.shields.io/badge/Connectivity-Wi--Fi%20AP-green)](https://en.wikipedia.org/wiki/Wi-Fi)
[![License](https://img.shields.io/badge/License-Academic-lightgrey)](#)

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Problem Statement](#-problem-statement)
- [System Architecture](#-system-architecture)
- [Hardware Components](#-hardware-components)
- [Core Logic & Mathematical Models](#-core-logic--mathematical-models)
- [State Machine](#-safety-state-machine)
- [Web Dashboard](#-web-dashboard)
- [Firmware Structure](#-firmware-structure)
- [Pin Reference](#-pin-reference)
- [Setup & Deployment](#-setup--deployment)

---

## Overview

The **UV-C Sterilization System** is an embedded, medical-grade device engineered to safely sanitize Neonatal Intensive Care Unit (NICU) beds and equipment. Built on the **ESP32 microcontroller**, it replaces dumb legacy open-loop, timer-based sanitizers with a **closed-loop dosimetric approach**.

By fusing real-time Time-of-Flight distance sensing, continuous AC monitoring, and live UV intensity tracking, the system guarantees precise delivery of the required germicidal dose. A multi-state hardware safety interface and a fully offline, asynchronous Wi-Fi web dashboard enable rigorous control and remote monitoring without any cloud dependency.

---

## Problem Statement

Hospital-Acquired Infections (HAIs) pose a critical threat in NICU environments where premature and critically ill neonates have severely compromised immune systems. While 254 nm UV-C radiation is a clinically proven germicidal technology, commercial UV-C deployments frequently suffer from:

| Flaw | Description |
|------|-------------|
| **Static Timers** | Fixed run-time ignores Inverse Square Law — distance changes make the dose under or over. |
| **No Fault Detection** | Degraded or burned-out lamps go undetected, providing a false sense of security. |
| **Safety Gaps** | No hardware interlocks prevent accidental UV-C exposure to medical staff. |

This project addresses all three by engineering an intelligent, sensor-driven ecosystem that:
- Calculates **dynamic exposure time** based on physical distance
- Physically **verifies lamp operation** via AC current monitoring
- Enforces **rigorous, multi-stage safety protocols** through hardware interlocks

---

## System Architecture

<img width="1003" height="564" alt="image" src="https://github.com/user-attachments/assets/05ba2117-b0a3-4cc7-9548-15d9c4833736" />

The architecture separates concerns into two parallel, non-blocking execution paths:
- **Hardware Control Loop** — state machine, safety checks, lamp control, OLED rendering
- **Async Web Server** — telemetry JSON endpoints, dashboard serving, command handling

---

## Hardware Components

| Component | Model | Role |
|-----------|-------|------|
| Microcontroller | ESP32 | Core processing, Wi-Fi AP, I2C host |
| UV-C Lamps | T5 UV-C Tubes (×3) | Germicidal irradiation |
| Lamp Control | Solid State Relays (×3) | High-voltage AC switching |
| Distance Sensor | VL53L0X (ToF) | Millimeter-precision bed distance |
| Current Sensor | ACS712-30A | AC line current verification |
| UV Sensor | UV Photodiode (ADC pin 34) | Live UV intensity & dose accumulation |
| Display | SSD1306 OLED 128×64 (I2C) | Real-time system status UI |
| Safety Input | 3-button physical array | Unlock, start, and emergency stop |

---

## Core Logic & Mathematical Models

### 1. Dynamic Time Calculation — Inverse Square Law

Upon startup, the VL53L0X measures the exact distance to the NICU bed. Exposure time scales automatically:

$$E = \frac{I}{d^2}$$

```
multiplier = (measured_distance / nominal_distance)²
adjusted_time = base_time × multiplier
```

**Safety clamps:** `0.25 ≤ multiplier ≤ 3.00`

If the target is twice as far, the system **quadruples** the run time. Too close, it shortens it — preventing material degradation.

---

### 2. AC Current Fault Detection — EMA Filter

The ESP32's ADC is inherently noisy. Raw ACS712 readings are processed through a software **Exponential Moving Average (EMA)** filter:

```
EMA_n = α × sample_n + (1 - α) × EMA_(n-1)
```

The expected current is computed dynamically based on the active lamp count. If measured current drops below a defined tolerance threshold (dead ballast or shattered bulb), the system **instantly triggers an Emergency Stop (E-STOP)**.

---

### 3. Dose Accumulation

UV dose is accumulated over the sterilization cycle:

```
currentDose += liveUV × 0.1  (integrated per loop tick)
Target dose: 15.0 mJ/cm²
```

---

## Safety State Machine

The system enforces a strict, one-way state flow. No state can be skipped.

```
  ┌─────────┐   Hold A+B       ┌─────────┐   2 seconds    ┌──────┐
  │ LOCKED  │ ──────────────►  │ HOLDING │ ─────────────► │ IDLE │
  └────▲────┘                  └─────────┘                └──┬───┘
       │   Release buttons            ▲                      │ Press Start
       │◄─────────────────────────────┘                      ▼
  ┌────┴────┐   Auto after 5s  ┌──────┐   Timer Done   ┌─────────┐
  │ FAULT   │◄──── E-STOP ──── │ DONE │◄───────────────│ RUNNING │
  └─────────┘                  └──────┘                └─────────┘
```

| State | Description |
|-------|-------------|
| `LOCKED` | Boot default. All relays are off. The display shows a lock icon. |
| `HOLDING` | Operator holds Safety A + Safety B simultaneously for 2 seconds. |
| `IDLE` | System armed. Distance measured. Awaiting start command. |
| `RUNNING` | UV lamps active. Dose accumulating. Countdown active on OLED. |
| `DONE` | Cycle complete. Relays off. Auto-returns to LOCKED after 5 seconds. |
| `FAULT` | E-STOP triggered (software or hardware). Relays cut. Hold A+B to recover. |

---

## Web Dashboard

The ESP32 broadcasts a **standalone Wi-Fi Access Point** — no router, no internet required.

```
SSID:     NICU_UV_System
Password: 12345678
URL:      http://192.168.4.1
```

### Dashboard Features

- **Live telemetry:** distance, recommended exposure, current UV dose, system state, UV lamp status
- **Mode control:** Auto (distance-based) or Manual (preset / custom seconds)
- **Remote start / E-STOP:** direct HTTP command buttons
- **Offline canvas graph:** custom HTML5 Canvas graphing engine — no Chart.js, no CDN dependencies
  - Plots: Distance (cm), Exposure Time (s), Lamp Usage (hrs) — rolling 20-point window

### REST API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Serves full HTML dashboard |
| `/data` | GET | Returns live JSON telemetry |
| `/start` | GET | Triggers sterilization start (IDLE only) |
| `/estop` | GET | Activates Emergency Stop |
| `/setMode?m=auto` | GET | Sets Auto mode |
| `/setMode?m=manual&t=<s>` | GET | Sets Manual mode with time in seconds |



---

## Pin Reference

| Pin | GPIO | Function |
|-----|------|----------|
| UV Lamp 1 SSR | 25 | Solid State Relay — Lamp 1 |
| UV Lamp 2 SSR | 26 | Solid State Relay — Lamp 2 |
| UV Lamp 3 SSR | 27 | Solid State Relay — Lamp 3 |
| Safety Button A | 17 | Unlock interlock — INPUT_PULLUP |
| Safety Button B | 18 | Unlock interlock — INPUT_PULLUP |
| Start Button | 19 | Cycle start — INPUT_PULLUP |
| UV Photodiode | 34 | ADC — UV intensity (analog) |
| ACS712 | 35 | ADC — AC current sensing |
| VL53L0X | SDA/SCL | I2C — ToF distance sensor |
| SSD1306 OLED | SDA/SCL | I2C — Display (shared bus) |

---

## Hardware Implementation

### Circuit Diagram
<img width="790" height="538" alt="image" src="https://github.com/user-attachments/assets/713f5610-fda6-47c4-a814-97453c10328d" />

### Fabricated PCB
<img width="715" height="408" alt="image" src="https://github.com/user-attachments/assets/69c2f63c-5862-4c62-b3da-903fc3974b42" />

PCB of Control Unit featuring esp32, ac-dc converter and 3.3v regulator with JST Connectors to connect the SSR’s and Sensors.

## Setup & Deployment

### Dependencies

Install via Arduino Library Manager or PlatformIO:

```
U8g2           >= 2.35
Adafruit_VL53L0X >= 1.2
ACS712         >= 1.0
WiFi           (ESP32 built-in)
WebServer      (ESP32 built-in)
Wire           (ESP32 built-in)
```

### Build & Flash

```bash
# PlatformIO (recommended)
pio run --target upload

# Arduino IDE
# Board: ESP32 Dev Module
# Partition: Default 4MB
# Flash Frequency: 80MHz
```

### First Boot Checklist

  VL53L0X sensor connected on I2C (SDA/SCL),
  OLED connected on the same I2C bus,
  ACS712 output wired to GPIO 35,
  UV photodiode wired to GPIO 34,
  SSR control wires on GPIO 25, 26, 27,
  Safety buttons on GPIO 17, 18 (to GND),
  Start button on GPIO 19 (to GND),
  Serial monitor @ 115200 baud to verify boot

On successful boot, Serial will print:

```
----------------------------------
🌐 WEB SERVER INITIALIZED
Dashboard IP: http://192.168.**.**
----------------------------------
```

## ⚠️ Safety Disclaimer

> UV-C radiation at 254 nm is harmful to human skin and eyes. This system is designed for **unoccupied NICU bays only**. All physical safety interlocks must be validated before each deployment. The Emergency Stop relay cutoff (`/estop`) must be tested and confirmed responsive before clinical use. This firmware is provided for academic and research purposes.

---

*Built with ESP32 · Arduino Framework · U8g2 · Adafruit VL53L0X · ACS712*

