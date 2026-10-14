# IoT-Based Smart Classroom Occupancy & Energy Optimization System (Edge AI)

Mini Project – Internet of Things | Final Year, Semester VII | AY 2026–27
**Walchand Institute of Technology, Solapur** – Dept. of Electronics and Computer Engineering
**Guide:** Dr. R. S. Khamitkar

| Team member | Roll No. | Enrollment No. |
|---|---|---|
| Fariyal Jamadar | 32 | 2303111054 |
| Tanvi Kanhed | 31 | 2303111040 |
| Riya Kapse | 33 | 2303111005 |

## Overview
Classrooms often leave lights and fans ON when empty. This project detects occupancy with
an ESP32, runs a small **Edge-AI model on the controller**, switches the light and fan
automatically, and publishes data over **MQTT** to a **Node-RED dashboard**.

> **Status:** developed and tested in **simulation only** (Wokwi). No physical hardware was tested.

## Architecture
```
PIR + DHT22 + LDR --> ESP32 (Edge-AI occupancy model) --> LED / Relay + Fan
                          |
                          | Wi-Fi + MQTT  (classroom/data)
                          v
                   MQTT broker --> Node-RED (parse, decide, dashboard)
                          ^                     |
                          +---- classroom/control (override / force-off)
```

## Hardware (simulated in Wokwi)
ESP32 DevKit V1, PIR sensor, DHT22, LDR, LED (light), relay module driving the fan (appliance).

| Component | ESP32 pin |
|---|---|
| PIR OUT | GPIO 27 |
| DHT22 DATA | GPIO 15 |
| LDR analog out | GPIO 34 |
| Relay IN (fan) | GPIO 26 |
| LED (light, via 220 Ω) | GPIO 25 |

## Repository structure
```
firmware/   ESP32 code (PlatformIO, Arduino framework) + Wokwi project files
ai/         Python script that trains the occupancy model and exports model.h
nodered/    Node-RED flow (import flow.json)
docs/       Screenshots and report (add yours here)
```

## How it works
1. Read PIR, DHT22 (temperature, humidity) and LDR (lux).
2. **Edge-AI:** logistic regression on `[motion, temp, humidity, lux]` → OCCUPIED / EMPTY with a confidence score.
   The PIR is pulsed, so "motion" means motion seen within the last 15 s (`MOTION_HOLD_MS`).
3. If occupied: fan ON; light ON only if lux < `LUX_THRESHOLD` (500). If empty: both OFF.
4. Publish JSON to `classroom/data`; listen on `classroom/control`.

**MQTT payload (`classroom/data`)**
```json
{"people":1,"temp":28.4,"hum":62.1,"lux":320,"light":1,"fan":1,"confidence":98,"mode":"auto"}
```
**Control messages (`classroom/control`)**
```json
{"mode":"manual","light":1,"fan":1}   // manual override
{"mode":"auto"}                       // back to Edge-AI control
{"action":"FORCE_OFF"}                // sent by Node-RED if room is empty but appliances are ON
```

## Setup

### 1. (Optional) Re-train the model
```bash
pip install numpy
python ai/train_model.py     # regenerates ai/classroom_dataset.csv and firmware/include/model.h
```
The model is trained on **synthetic** data, so the reported accuracy applies to that dataset only.
The same script runs unchanged in Google Colab.

### 2. Firmware (VS Code + PlatformIO)
```bash
cd firmware
pio run                      # build
```
To simulate: install the **Wokwi for VS Code** extension, open the `firmware` folder and run
*Wokwi: Start Simulator* (uses `diagram.json` and `wokwi.toml`). Check that wiring in
`diagram.json` matches the pin table above; adjust in the Wokwi editor if needed.

### 3. Node-RED dashboard
```bash
npm install -g node-red
cd ~/.node-red && npm install node-red-dashboard
node-red
```
Open `http://localhost:1880`, menu → *Import* → select `nodered/flow.json`, click *Deploy*,
then open `http://localhost:1880/ui`.

> The code uses the public broker `broker.hivemq.com`, which anyone can read.
> Change the topic prefix in `firmware/include/config.h` and the Node-RED MQTT nodes to
> something unique to avoid mixing with other users' data.

## Tests (simulation)
| # | Test | Result |
|---|---|---|
| 1 | Motion → OCCUPIED, light/fan ON | Passed |
| 2 | No motion → light/fan OFF | Passed |
| 3 | Sensor data published to MQTT and shown on dashboard | Passed |

## Tech stack
C/C++ (Arduino), Python (NumPy), Node-RED (JavaScript function node), MQTT, Wokwi, PlatformIO.
