<div align="center">

# 🛡️ RoadGuard AI
### Smart Helmet & Bike ECU Triple-Lock Ignition System

**1 Person (`R001`) • 1 Helmet (`RH001`) • 1 Bike (`KA00XX0000`)**

![Hardware](https://img.shields.io/badge/Hardware-ESP8266_%2B_ESP32--S3-0ea5e9?style=for-the-badge)
![Cloud](https://img.shields.io/badge/Cloud-Firebase_RTDB-f59e0b?style=for-the-badge)
![Security](https://img.shields.io/badge/Security-MAC_%2B_Hotspot_%2B_IMU-22c55e?style=for-the-badge)

*An IoT rider safety and anti-theft ignition interlock that prevents a motorcycle from starting unless the verified owner is wearing their paired helmet, sober, and seated safely.*

</div>

---

## 📌 Quick Navigation
- [✨ Why RoadGuard AI?](#-why-roadguard-ai)
- [🔄 How It Works (Visual Flow)](#-how-it-works-visual-flow)
- [🔐 The 4 Ignition Lock Gates](#-the-4-ignition-lock-gates)
- [🔌 Hardware & Pin Mapping](#-hardware--pin-mapping)
- [🚦 ESP32-S3 RGB LED & LCD Status Guide](#-esp32-s3-rgb-led--lcd-status-guide)
- [🗄️ Firebase Cloud Endpoints](#️-firebase-cloud-endpoints)
- [⚙️ Arduino IDE Setup](#️-arduino-ide-setup)
- [🎬 60-Second Live Demo Walkthrough](#-60-second-live-demo-walkthrough)

---

## ✨ Why RoadGuard AI?

| Problem | RoadGuard AI Solution |
| :--- | :--- |
| **Bike Theft / Hotwiring** | Ignition stays physically locked unless both the owner's hotspot key (`R001`) and bound helmet (`RH001`) verify together. |
| **Helmet Spoofing / Clones** | The Bike ECU permanently binds to the helmet's unique hardware **MAC address** on first pairing. |
| **Helmet Removal After Start** | An **MPU6050 IMU** detects removal or tilt $\ge 55^\circ$ for $2.5\text{ s}$ and immediately cuts ignition. |
| **Drunk Riding & Crash Delay** | An **MQ-3 alcohol sensor** blocks ignition before start, while high-G impact detection ($\ge 3.2\text{ G}$) logs emergency alerts with live **GPS** coordinates. |

---

## 🔄 How It Works (Visual Flow)

```text
  📱 OWNER PHONE KEY                  🪖 HELMET UNIT (ESP8266)               🏍️ BIKE ECU (ESP32-S3-N8R2)
 ┌───────────────────┐               ┌────────────────────────┐              ┌───────────────────────────┐
 │ SSID: vivo T3x 5G │──(Wi-Fi Key)─▶│ • Verifies Owner R001  │              │ • Reads /helmet.json (2s) │
 │ Rider ID: R001    │               │ • Checks Wear (< 55°)  │              │ • Verifies Helmet MAC     │
 └───────────────────┘               │ • Reads MQ-3 & GPS     │              │ • Checks IR Seat Sensor   │
                                     └───────────┬────────────┘              └─────────────┬─────────────┘
                                                 │                                         │
                                          HTTPS PUT (5s)                            HTTPS GET/PUT (2s)
                                                 │                                         │
                                                 ▼                                         ▼
                                     ┌───────────────────────────────────────────────────────────────────┐
                                     │                   ☁️ FIREBASE REALTIME DATABASE                    │
                                     │             /helmet.json   •   /bikePairing.json   •   /bikeEcu.json│
                                     └─────────────────────────────────┬─────────────────────────────────┘
                                                                       │
                                                                REST Poll (1.5s)
                                                                       ▼
                                                        ┌─────────────────────────────┐
                                                        │  💻 LIVE WEB DASHBOARD       │
                                                        │  • Ignition Banner (Red/Grn)│
                                                        │  • 4 Security Gate Cards    │
                                                        │  • GPS Map + Demo Simulator │
                                                        └─────────────────────────────┘
```

---

## 🔐 The 4 Ignition Lock Gates

Every single condition below must be **TRUE** for the Bike ECU (`KA00XX0000`) to unlock ignition:

1. **Gate 1 — 1:1 Hardware MAC Binding:** Live `helmetMAC` must match `authorizedMAC` stored in `/bikePairing.json`.
2. **Gate 2 — Owner Hotspot Key (`R001`):** Helmet must stay connected to `vivo T3x 5G` (losing Wi-Fi for $> 5\text{ s}$ triggers `OWNER_WIFI_LOST`).
3. **Gate 3 — Anti-Swap Wear Sensor:** Helmet must remain upright ($< 55^\circ$ tilt from worn reference); tilting $\ge 55^\circ$ for $2.5\text{ s}$ triggers `REMOVED-RELOCKED`.
4. **Gate 4 — Sobriety, Crash & Stand/Seat Check:** MQ-3 alcohol reading must be $< 550$, G-force $< 3.2\text{ G}$, no SOS active, and the Bike ECU's **HW-201 IR sensor** (`GPIO 6`) must be covered.

---

## 🔌 Hardware & Pin Mapping

### 1️⃣ Smart Helmet Unit — NodeMCU ESP8266 (`roadguard_helmet_nodemcu.ino`)

| Module | Signal / Pin | NodeMCU Pin | Function |
| :--- | :--- | :--- | :--- |
| **16x2 I2C LCD** | `SDA` / `SCL` | `D2` / `D1` | Live status display ($50\text{ kHz}$ I2C bus) |
| **MPU6050 IMU** | `SDA` / `SCL` | `D2` / `D1` | Tilt removal detection ($\ge 55^\circ$) & crash G-force ($\ge 3.2\text{ G}$) |
| **MQ-3 Sensor** | `AO` | `A0` | Alcohol detection (trigger $\ge 550$, clear $\le 510$) |
| **Active Buzzer** | `+` / `-` | `D0` / `GND` | Audio feedback on lock/unlock and safety alerts |
| **NEO-6M GPS** | `TX` / `RX` | `D6` / `D5` | Live latitude, longitude, and speed (`9600` baud) |

### 2️⃣ Bike ECU Unit — ESP32-S3-N8R2 (`roadguard_esp32s3_bike_ecu.ino`)

| Module | Signal / Pin | ESP32-S3 GPIO | Function |
| :--- | :--- | :--- | :--- |
| **Board Power** | Regulated $5.0\text{ V}$ | `USB-C` | Direct $5\text{ V}$ USB-C supply *(no multimeter needed)* |
| **Onboard RGB LED** | `WS2812 DATA` | `GPIO 48` *(or `38`)* | Visual ignition lock/unlock status indicator |
| **HW-201 IR Sensor** | `OUT` (`VCC` → `3V3`) | `GPIO 6` | Kickstand / rider-seated interlock (`LOW` = Ready) |
| **Starter Button** | Signal (`GND` return) | `GPIO 5` | Simulates bike starter switch (`INPUT_PULLUP`) |
| **Ignition Output** | Relay / Logic Out | `GPIO 4` | `HIGH` = Ignition Enabled, `LOW` = Ignition Cut |

---

## 🚦 ESP32-S3 RGB LED & LCD Status Guide

| System State | Helmet 16x2 LCD | ESP32-S3 RGB LED | Starter Button (`GPIO 5`) |
| :--- | :--- | :--- | :--- |
| **Connecting Wi-Fi** | `CHECKING KEY...` | 🟠 **Amber** | Denied (`START DENIED!`) |
| **Hotspot OFF (Locked)** | `LOCKED [NO KEY]` | 🔴 **Red** | Denied (`START DENIED!`) |
| **Helmet Removed ($\ge 55^\circ$)** | `REMOVED-RELOCKED` | 🔴 **Red** | Denied (`START DENIED!`) |
| **Alcohol / Crash Alert** | `ALCOHOL DETECTED` / `CRASH!` | 🔴 **Red** + Buzzer | Denied (`START DENIED!`) |
| **All Checks Passed** | `ACTIVE [OWNEROK]` | 🟢 **Green** | 🔵 **Blue Flash** + `ENGINE STARTED!` |

---

## 🗄️ Firebase Cloud Endpoints

- **Database Root:** `https://roadguard-ai-a7b58-default-rtdb.firebaseio.com`

| Endpoint | Updated By | Purpose |
| :--- | :--- | :--- |
| `/helmet.json` | Helmet NodeMCU | Streams `status`, `helmetMAC`, `ownerVerified`, `helmetWorn`, `tiltDeg`, `alcoholValue`, `gForce`, and GPS coordinates. |
| `/bikePairing.json` | ESP32-S3 Bike ECU | Stores permanent `authorizedMAC` binding between Helmet `RH001` and Bike `KA00XX0000`. |
| `/bikeEcu.json` | ESP32-S3 Bike ECU | Reports `ignitionUnlocked`, `irStandReady`, `starterPressed`, and `pairedMAC` to the web dashboard. |

---

## ⚙️ Arduino IDE Setup

1. **Helmet NodeMCU (`ESP8266`):**
   - **Board:** `NodeMCU 1.0 (ESP-12E Module)`
   - **Required Libraries:** `hd44780` (`hd44780_I2Cexp`), `TinyGPSPlus`
   - **Baud Rate:** `115200`
2. **Bike ECU (`ESP32-S3-N8R2`):**
   - **Boards Manager URL:** `https://espressif.github.io/arduino-esp32/package_esp32_index.json`
   - **Board:** `ESP32S3 Dev Module` • **PSRAM:** `OPI PSRAM` • **USB CDC On Boot:** `Enabled`
   - **Baud Rate:** `115200`
3. **Web Dashboard (`roadguard_working_website.html`):**
   - Open directly in any modern browser or embed in Google Sites (**Insert → Embed → Embed code**).

---

## 🎬 60-Second Live Demo Walkthrough

1. **Show Locked State (0:00 – 0:15):** Keep `vivo T3x 5G` hotspot **OFF**. Point out `LOCKED [NO KEY]` on the helmet LCD, **RED** RGB LED on the ESP32-S3, and **RED (`IGNITION LOCKED`)** on the dashboard. Press `GPIO 5` to show engine start is blocked.
2. **Show Owner Verification & 1:1 Pairing (0:15 – 0:35):** Turn `vivo T3x 5G` (`2.4 GHz`) **ON** and cover the bike IR sensor (`GPIO 6`). Watch the helmet show `ACTIVE [OWNEROK]`, bind its MAC, turn the ESP32-S3 RGB **GREEN**, and unlock the dashboard. Press `GPIO 5` to show `ENGINE STARTED!`.
3. **Show Anti-Swap & Safety Interlocks (0:35 – 1:00):** Tilt the helmet $\ge 55^\circ$ for $2.5\text{ s}$ (or turn off the hotspot for $5\text{ s}$) to show instant relock back to **RED**. *(Tip: Use the dashboard's **Demo Simulator** toggle if hall Wi-Fi fluctuates.)*