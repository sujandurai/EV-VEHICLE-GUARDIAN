# ⚡ EV Guardian — Edge AI Battery Management System (BMS)

> A dual-processor, Edge AI Battery Management System featuring real-time tinyML inference, a physics-informed state-of-health pipeline, an offline local LLM copilot, and a 3D digital twin telemetry HUD. Built for the Snapdragon Multiverse Hackathon on the Arduino Uno Q (STM32 + QRB2210) & Snapdragon X platform.

[![Python Version](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/)
[![Platform](https://img.shields.io/badge/hardware-Arduino%20Uno%20Q%20%7C%20Snapdragon%20X-orange.svg)]()
[![Model Format](https://img.shields.io/badge/inference-ONNX%20Runtime-green.svg)]()
[![License](https://img.shields.io/badge/license-MIT-lightgrey.svg)](LICENSE)

---

## 🚀 Presentation Videography & Demos
* **3D Digital Twin Visualizer**: Web-based real-time Three.js dashboard displaying thermal heatmaps of battery cells in real-time.
* **Secondary Telemetry HUD**: Dark-mode Tkinter-based physical console running on the 7-inch HDMI display, showing live cell voltages, pack metrics, sensor trust index, and prompt diagnostic values.

---

## 🛠️ High-Level Technical Architecture

```
                                      [ E V   B A T T E R Y   C E L L S (1S-4S) ]
                                                            │
                                                            ▼ (Analog Signals)
                                               [ Arduino Uno Q (STM32U585 MCU) ]
                                               Realtime 14-bit ADC & Sensor Filtering
                                                            │
                                                            ▼ (DualSerial UART @ 115200)
    ┌───────────────────────────────────────────────────────┴───────────────────────────────────────────────────────┐
    ▼ (Option A: Standalone Kiosk)                                                   ▼ (Option B: Laptop-Hosted HUD)
[ QRB2210 Debian Linux MPU ]                                                     [ Snapdragon X PC Backend ]
├── local MQTT Broker (mosquitto)                                                 ├── Ingests packet telemetry
├── gateway_daemon.py (Mode: HW)                                                 ├── runs ONNX Inference pipelines
└── Chromium Kiosk window (DISPLAY=:0)                                           ├── Hosts Web server (port 8081)
                                                                                  └── Drives Tkinter Telemetry HUD (COM34)
```

---

## 💎 Primary Features

### 1. Dual-Processor Edge Topology (Arduino Uno Q)
* **Microcontroller Core (STM32U585)**: Runs high-frequency 10Hz Analog-to-Digital conversions with custom noise-reduction filters, serving as a low-power real-time sensor node.
* **Linux MPU Core (Qualcomm QRB2210)**: Acts as the high-speed gateway, running a MQTT publishing daemon, data cache pipelines, and low-latency local processing.

### 2. Multi-Model Edge AI Pipeline
* **Sensor Trust Model (ONNX)**: Runs an Anomaly Detection engine (Isolation Forest) on raw telemetry vectors (`[C1, C2, C3, C4, Current, Temp, Vibration, Gas]`) at the edge to check signal integrity and flag sensor drifts.
* **State of Health (SOH) Estimator**: Utilizes a Physics-Informed Neural Network (PINN) to gauge capacitance loss, and capacity degradation based on electro-chemical battery physics models.

### 3. Real-Time Telemetry HUD (7-inch HDMI Display)
* High-contrast, dark-mode Tkinter interface.
* Displays live cell voltages, Pack Current, Multi-point Temperatures, Gas counts, and IMU status.
* Employs a custom **Brace-Matching Stream Parser** to cleanly digest and display complex JSON diagnostic outputs without data drop-offs.
* Integrates a C++ macro wrapper (`DualSerial`) in the firmware to broadcast data to both USB and the internal MPU UART link simultaneously.

### 4. Offline LLM Diagnosis Copilot
* Ingests battery state buffers into a local **Llama-3-8B** network (via Ollama) to output diagnostic descriptions and actionable safety advice during emergencies.
* Includes a rule-based safety fallback engine that asserts immediate critical status even if the LLM is offline or busy.

---

## 📦 Directory Map
| Path | Description |
|---|---|
| `ev_guardian_uno_q/ev_guardian_uno_q.ino` | STM32 Firmware (14-bit ADC, auto-callibrate IMU, DualSerial wrapper) |
| `view_serial_dashboard.py` | Tkinter Realtime Telemetry HUD for the 7-inch HDMI monitor |
| `backend.py` | Core engine: MQTT + ONNX Inference + WebSockets + SQLite Local Cache |
| `serve_dashboard.py` | Dual-Stack HTTP server hosting the digital twin web app |
| `train_anomaly_model.py` | Isolation Forest anomaly detection training pipe |
| `train_soh_model.py` | SOH capability degradation training script |
| `llm_diagnostics.py` | Offline LLM Copilot (Ollama API / safety policy fallback) |
| `cloud_sync.py` | Cloud telemetry uploader (Qualcomm Cloud AI 100 mock) |
| `dashboard/` | Core 3D visualizer asset directory (HTML/CSS/Three.js) |

---

## ⚡ Quick Start & Run Guide

### 1. Install System Prerequisites
Install the required Python environment on your host PC:
```powershell
pip install paho-mqtt onnxruntime scikit-learn skl2onnx websockets aiohttp requests numpy pyserial
```

Download and start the **Mosquitto MQTT Broker**:
* **Windows**: Download at [mosquitto.org](https://mosquitto.org/download/) and start it using:
  ```powershell
  net start mosquitto
  ```

### 2. Flashing the Board
1. Open the Arduino IDE on your laptop.
2. Load and build `ev_guardian_uno_q/ev_guardian_uno_q.ino`.
3. Flash the firmware onto the **Arduino Uno Q**.
4. Close the Arduino IDE to free the serial port (`COM34`).

### 3. One-Click Launch (Simulated Suite)
If you want to run the suite locally with simulated inputs:
```powershell
python launch.py
```
This opens the simulation engines, local servers, and launching browser windows automatically.

### 4. Run the Real-Time Telemetry HUD (Physical Hardware Setup)
To run the live system reading cell measurements directly from your board via `COM34`:
1. Connect the **7-inch TFT HDMI input** directly to your **Laptop's HDMI port**.
2. Connect the **TFT screen's micro-USB power cord** directly to your **Laptop's USB port**.
3. Connect the **Arduino Uno Q** via its USB cable to your laptop.
4. Run the HUD dashboard script:
   ```powershell
   python view_serial_dashboard.py
   ```
5. Drag the dashboard window onto your 7-inch monitor and press **`F11`** to make it go full screen!

---

## 📋 Fault States & Simulation Logic
During standalone demonstration cycles, the system simulates cell-level failures to showcase Edge AI diagnostic reaction speeds:
* **Healthy Cycle (0s - 10s)**: Balanced cell voltages (3.78 - 3.84V), nominal temperatures (~34°C). The status will show **`HEALTHY`** (Green).
* **Thermal/Cell Fault Cycle (11s - 20s)**: Cell 3 drops to `1.25V` (Cell Imbalance), temperature spikes to `115°C` (Thermal Runaway). The HUD will immediately trigger a **`HIGH`** or **`CRITICAL`** alarm and display the anomaly flags.

---

> Created for the Snapdragon Multiverse Hackathon. Fully optimized for energy efficiency, reliability, and edge intelligence.
