# RoadGuard AI — Smart Helmet & Bike ECU System

A **1-Person (`R001`) • 1-Helmet (`RH001`) • 1-Bike (`KA00XX0000`)** smart rider safety and anti-theft ignition interlock system connecting a **NodeMCU ESP8266** helmet unit, an **ESP32-S3-N8R2** bike ECU unit, **Firebase Realtime Database**, and a single-file web dashboard (`roadguard_working_website.html`).

---

## System Architecture & Communication Flow

1. **Owner Hotspot Key (`vivo T3x 5G`):**
   - SSID: `vivo T3x 5G` (set to **2.4 GHz** or dual-band so the ESP8266 NodeMCU connects reliably)
   - Password: `Sharan@2005`
   - Acts as the rider authentication key (`R001`). Losing hotspot connectivity for **5 seconds** (`WIFI_LOSS_LOCK_MS = 5000`) locks the helmet (`OWNER_WIFI_LOST`).
2. **Helmet Telemetry (`GET` / `PUT` to `/helmet.json`):**
   - The **Helmet NodeMCU** publishes `status`, `helmetId` (`RH001`), `riderId` (`R001`), `bikeId` (`KA00XX0000`), `helmetMAC`, `ownerVerified`, `helmetWorn`, `tiltDeg`, `lockReason`, `alcohol`, `alcoholValue`, `crash`, `sos`, `gForce`, `gpsLocked`, `lat`, `lon`, and `speed`.
3. **1-Helmet ↔ 1-Bike Pairing (`/bikePairing.json`):**
   - The **ESP32-S3 Bike ECU** reads `/bikePairing.json` and `/helmet.json` every **2 seconds**. On first valid match when no MAC is bound yet, it binds permanently to `authorizedMAC`.
4. **Bike ECU Enforcement (`/bikeEcu.json`) & Web Dashboard:**
   - Ignition unlocks only when the helmet is online, IDs and MAC match, `ownerVerified` and `helmetWorn` are `true`, no alcohol/crash/SOS alert is active, and the bike's **HW-201 IR sensor** (`GPIO 6`) is covered (`irStandReady = true`).
   - `roadguard_working_website.html` polls `/helmet.json`, `/bikePairing.json`, and `/bikeEcu.json` every **1.5 seconds** and also provides a **Demo Simulator** toggle for offline/presentation testing.

---

## Hardware & Pin Mapping

### 1. Helmet Unit — NodeMCU ESP8266 (`roadguard_helmet_nodemcu.ino`)
- **Board Setting (Arduino IDE):** `NodeMCU 1.0 (ESP-12E Module)`
- **16x2 I2C LCD + MPU6050:** `SDA` → `D2`, `SCL` → `D1`
- **MQ-3 Alcohol Sensor:** `AO` → `A0` (threshold `550`, hysteresis `40`)
- **Buzzer:** `+` → `D0`, `-` → `GND`
- **GPS Module (9600 baud):** `TX` → `D6`, `RX` → `D5`
- **Crash / Removal Thresholds:** Crash impact `>= 3.2 G`; helmet removal/tilt `>= 55°` held for `2500 ms` (`REMOVED-RELOCKED`).

### 2. Bike ECU Unit — ESP32-S3-N8R2 (`roadguard_esp32s3_bike_ecu.ino`)
- **Board Settings (Arduino IDE):** `ESP32S3 Dev Module`, `PSRAM: OPI PSRAM`, `USB CDC On Boot: Enabled`
- **Power:** Use a regulated **5V USB-C** connection for bench/demo testing without a multimeter. *(If using an LM2596 buck converter on a battery, verify `5.0V` output first before connecting to `5V`/`5VIN` and `GND`.)*
- **HW-201 IR Sensor (Stand / Seat Interlock):** `VCC` → `3V3`, `GND` → `GND`, `OUT` → `GPIO 6`
- **Starter / Ignition Push Button:** 1st leg → `GPIO 5`, diagonal leg → `GND` (internal pull-up)
- **Ignition Output Pin:** `GPIO 4` (`HIGH` = Unlocked, `LOW` = Locked)
- **Onboard WS2812 RGB LED:** `GPIO 48` (change to `38` on v1.1 board revisions if needed) — **Amber** = Wi-Fi connecting/lost, **Red** = Locked, **Green** = Unlocked, **Blue flash** = Starter button pressed while unlocked.

---

## Quick Setup & Live Demonstration Steps

1. **Flash Helmet NodeMCU:** Open `roadguard_helmet_nodemcu/roadguard_helmet_nodemcu.ino` in Arduino IDE, select `NodeMCU 1.0 (ESP-12E Module)` and its COM port, upload, and monitor at `115200` baud.
2. **Flash ESP32-S3 Bike ECU:** Install `esp32 by Espressif Systems` via Boards Manager, open `roadguard_esp32s3_bike_ecu/roadguard_esp32s3_bike_ecu.ino`, select `ESP32S3 Dev Module` (`OPI PSRAM`) and the board's COM port, and upload.
3. **Open Dashboard:** Open `roadguard_working_website.html` in a browser (or paste its HTML into a Google Sites embed).
4. **Run Demo:**
   - **Locked (`vivo T3x 5G` OFF):** Helmet shows `LOCKED [NO KEY]`, ESP32-S3 RGB is **Red**, dashboard banner is **Red**, and pressing `GPIO 5` is denied.
   - **Unlocked (`vivo T3x 5G` ON + IR `GPIO 6` covered):** Helmet shows `ACTIVE [OWNEROK]`, MAC binds in `/bikePairing.json`, ESP32-S3 RGB turns **Green**, and dashboard banner turns **Green**.
   - **Relock / Safety Test:** Tilt helmet `>= 55°` for `2.5s`, turn hotspot OFF for `5s`, or trigger MQ-3 alcohol / MPU6050 impact (`>= 3.2G`) to see immediate relock back to **Red**.