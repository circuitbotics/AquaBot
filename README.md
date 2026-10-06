🌊 AquaBot — Autonomous Water Quality Monitoring System
> \*\*Clean Water, Safe Future\*\*
> Team \*\*CircuitBotics\*\* · World Robot Olympiad (WRO) 2026 · Future Innovators – Senior · India 🇮🇳
AquaBot is a low-cost autonomous boat that patrols a lake or pond, measures water quality, automatically corrects pH using onboard dosing pumps, detects floating garbage with AI, and uploads everything over GSM/GPRS to a Cloudflare Worker live dashboard — no WiFi needed.
<!-- Replace with your best photo -->
![AquaBot](docs/images/aquabot-hero.jpg)
---
📌 Table of Contents
Problem & Solution
Key Features
Demo
System Architecture
Hardware
Repository Structure
Getting Started
How It Works
Challenges & Solutions
Roadmap
Social Impact & SDGs
Team
Use of AI Tools
Safety Disclaimer
License
---
🎯 Problem & Solution
Water pollution in lakes and ponds is a growing problem in India. Manual testing is slow, expensive and needs trained people on site, so pollution is often found too late.
AquaBot removes that gap: it monitors continuously, reacts instantly to pH problems, flags visible garbage, logs data over time and sends alerts over the mobile network. Commercial monitoring buoys cost lakhs of rupees, only report data and cannot move. AquaBot is mobile, autonomous and can act.
✨ Key Features
🧭 Autonomous geo-fenced patrol — lawnmower-pattern waypoints using the Haversine formula, running on the ESP32
🧪 Multi-parameter sensing — pH, TDS, turbidity, water temperature, rainfall, tilt/IMU (dissolved oxygen planned)
💧 Automatic pH correction — acid and alkaline dosing pumps triggered when pH < 6.5 or > 8.5
♻️ AI garbage detection — TensorFlow Lite on a Coral Edge TPU, alerts with image + GPS
📡 Works without WiFi — SIM800L GSM/GPRS uploads to a Cloudflare Worker dashboard
🖥️ On-boat 2.8" TFT display for on-site readings
🛑 Safety — geofence stop, 5 s heartbeat timeout, pump cooldown
🎬 Demo
<!-- Add your own links / files -->
Demo	Link
Full working demo	`media/videos/full-demo.mp4` / YouTube link
Sensor demonstration	`media/videos/sensors-demo.mp4` / YouTube link
Live dashboard	your-worker.workers.dev
Photos
	
![Front](docs/images/front.jpg)	![Top](docs/images/top.jpg)
![Side](docs/images/side.jpg)	![Electronics](docs/images/electronics.jpg)
🏗️ System Architecture
AquaBot uses a dual-controller design: real-time control is separated from sensing/AI/cloud so that GPS navigation is never interrupted by AI inference or network activity.
```
                    ┌──────────────────────────────┐
                    │   Cloudflare Worker (KV)     │
                    │   Live web dashboard         │
                    └──────────────▲───────────────┘
                                   │ HTTPS POST (GPRS)
                    ┌──────────────┴───────────────┐
                    │      SIM800L GSM module      │
                    └──────────────▲───────────────┘
                                   │ UART
┌──────────────────────────────────┴─────────────────────────────────┐
│ Raspberry Pi Zero 2W — Sensing \& Intelligence                       │
│ ADS1115 (pH, TDS, turbidity) · DS18B20 · MPU6050 · Rain sensor      │
│ Pi Camera + Coral Edge TPU · 2.8" TFT                               │
└──────────────────────────────────▲─────────────────────────────────┘
                                   │ Bluetooth Classic (SPP), JSON + checksum + heartbeat
┌──────────────────────────────────┴─────────────────────────────────┐
│ ESP32 — Real-time Navigation \& Actuation                            │
│ GPS NEO-6M · Motor driver (2× DC motors) · 2-ch relay (acid/alkali) │
└────────────────────────────────────────────────────────────────────┘
```
Controller / Service	Connected components	Link
Raspberry Pi Zero 2W	pH, TDS, turbidity, DS18B20, ADS1115, MPU6050, rain sensor, Pi Camera + Edge TPU, TFT, GSM	I2C / 1-Wire / CSI / SPI / UART
ESP32	GPS NEO-6M, motor driver, 2-ch relay	UART / PWM / GPIO
Pi ⇄ ESP32	Patrol boundary, pump triggers ↔ GPS + status	Bluetooth Classic SPP
GSM → Cloud	Readings, GPS trail, garbage alerts	GPRS → HTTPS POST
🔧 Hardware
Full list in `hardware/BOM.md`. Highlights:
Component	Purpose
Raspberry Pi Zero 2W	Sensing, AI inference, TFT, GSM/cloud
ESP32 Dev Board	GPS navigation, motors, relays
SIM800L GSM module	GPRS data upload
ADS1115 ADC	Analog sensor → digital
pH, TDS V1.0, Turbidity, DS18B20	Water quality
GPS NEO-6M, MPU6050, Rain sensor	Position, stability, rainfall
2× DC motors + tyres, motor driver	Propulsion
2-ch relay + 2 pumps + 2 tanks	Acid / alkaline dosing
Pi Camera + Coral Edge TPU	Garbage detection
2.8" TFT, LiPo + 5 V buck converter	Display, power
DO sensor	Planned (funding constrained)
Hull: waterproof Sunboard body, two side-mounted DC motors driving rubber wheels.
📁 Repository Structure
```
AquaBot/
├── README.md
├── LICENSE
├── .gitignore
├── requirements.txt          # Raspberry Pi Python dependencies
├── pi/                       # Raspberry Pi Zero 2W code (Python 3)
├── esp32/                    # ESP32 firmware (C++ / Arduino)
├── cloudflare-worker/        # Cloud backend + dashboard (JavaScript)
├── hardware/                 # BOM, wiring, schematics
├── docs/                     # Full WRO report, images
│   └── images/
└── media/videos/             # Demo \& sensor videos
```
🚀 Getting Started
1. Raspberry Pi Zero 2W
```bash
git clone https://github.com/<your-username>/AquaBot.git
cd AquaBot
sudo apt update
sudo apt install -y python3-pip python3-picamera2 python3-lgpio bluez libbluetooth-dev
pip install -r requirements.txt --break-system-packages
```
Enable I2C, SPI, 1-Wire and the serial port with `sudo raspi-config`. Then run the master script (starts sensor reader, TFT dashboard, AI vision, GSM uploader and auto-restarts crashed scripts):
```bash
python3 pi/start.py
```
> Copy `pi/config.example.py` to `pi/config.py` and fill in your Worker URL, API key and SIM APN. \*\*Never commit `config.py`.\*\*
2. ESP32
Install the Arduino IDE and the ESP32 board package.
Open the sketch in `esp32/`.
Select your ESP32 board, upload.
Pair the Pi with the ESP32 over Bluetooth SPP.
3. Cloudflare Worker
```bash
cd cloudflare-worker
npm install -g wrangler
wrangler login
wrangler kv namespace create AQUABOT\_KV   # bind it in wrangler.toml
wrangler deploy
```
Open your `\*.workers.dev` URL to see the live dashboard.
⚙️ How It Works
pH response — If pH < 6.5 or > 8.5, the Pi sends a dosing command over Bluetooth; the ESP32 runs the matching pump for 5 s, then enforces a 30 s cooldown. An alert is uploaded to the dashboard.
Garbage detection — Camera frames go to a MobileNet SSD / YOLOv5-Lite model on the Edge TPU. Above the confidence threshold the Pi captures an image, tags it with the latest GPS position and uploads it for human review.
Geo-fenced patrol — The operator draws a boundary on the dashboard (or enters coordinates). The ESP32 generates lawnmower waypoints (Haversine distance/bearing, 2 m tolerance) and drives itself. It stops the motors if it leaves the boundary or if no Pi heartbeat arrives for 5 s.
🛠️ Challenges & Solutions
Challenge	Solution
Lag / dropped Bluetooth messages	JSON protocol with checksum + heartbeat; motors stop on timeout
Unreliable GSM on weak signal	Retry with exponential backoff; buffer readings on SD card
GPIO busy error	Switched display to `lgpio`
pH readings out of range (20+)	Two-point calibration with slope/intercept
TFT showing 1/3 of screen	Larger SPI buffer (131072) + chunked 32 KB frames
Persisting GPS trail without a DB	Cloudflare Workers KV
Many scripts on the Pi	`start.py` with staggered start and auto-restart
Slow AI on Pi CPU	Coral Edge TPU: ~2 s → ~200 ms per frame
🗺️ Roadmap
[ ] Real-lake outdoor testing and calibration
[ ] Dissolved oxygen (DO) sensor
[ ] Solar charging
[ ] Obstacle avoidance (ultrasonic on ESP32)
[ ] Brushless motors + ESCs
[ ] Hardened multi-boat cloud backend
[ ] Pilot deployments on 3 water bodies and regulatory clearance
🌍 Social Impact & SDGs
Supports SDG 6 (Clean Water & Sanitation), SDG 9 (Industry, Innovation & Infrastructure), SDG 14 (Life Below Water) and SDG 15 (Life on Land). Helps local communities, farmers, fishermen, municipal authorities, conservationists and schools. Prototype cost is estimated under ₹20,000.
👥 Team
CircuitBotics — DBRA CM Shri Janakpuri, Desu Colony, Janakpuri, New Delhi – 110058
Role	Name	Contribution
Team Leader	Vivek Kumar	System architecture, Pi + ESP32 code, GSM, Cloudflare Worker
Member	MD Arif	Mechanical design, assembly, wiring
Coach	Sanjay Bhatt	Guidance and mentorship
🤖 Use of AI Tools
In line with WRO 2026 Rule 6.5: the team used Claude (Anthropic) as a coding assistant for Python, ESP32 firmware and the Cloudflare Worker, and to help structure parts of the report. All hardware decisions, construction, wiring, testing and the core idea were the team's own.
⚠️ Safety Disclaimer
AquaBot is a research / educational prototype. Do not dose chemicals into any public or natural water body without proper calibration, supervision and Pollution Control Board / environmental approval.
📄 License
Released under the MIT License.
---
<p align="center">⭐ If you like this project, please star the repo! ⭐</p>
