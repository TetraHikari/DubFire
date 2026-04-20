# Dub-Fire — Autonomous Fire Detection & Suppression System

An intelligent fire suppression turret that uses dual-camera confirmation (RGB + thermal) powered by YOLOv8, an ESP32-controlled servo/weapon platform, and a real-time Next.js dashboard with push notifications.

---

## Overview

Dub-Fire is a senior capstone project combining computer vision, embedded systems, IoT, and full-stack web development. The system autonomously detects fires, aims a suppression mechanism at the target, and notifies users in real time via email, LINE Messaging, and web push.

**Fire confirmation logic:**

| RGB (YOLO) | Thermal (>100°C) | Status |
|---|---|---|
| ✅ | ✅ | **FIRE DETECTED** — arm + shoot |
| ❌ | ✅ | **THERMAL ALERT** — high confidence, no shoot |
| ✅ | ❌ | **FALSE ALARM** — no heat detected |
| ❌ | ❌ | **No fire** — scanning |

---

## Architecture

```
┌──────────────────────────────────────────────┐
│              Next.js Web App (port 3000)      │
│  Dashboard · Map · Settings · Notifications   │
└──────────────────┬───────────────────────────┘
                   │ HTTP proxy
┌──────────────────▼───────────────────────────┐
│         Python AI Server (port 5001)          │
│  YOLOv8 · Thermal analysis · MJPEG streams   │
└──────────────────┬───────────────────────────┘
                   │ Serial (115200 baud)
┌──────────────────▼───────────────────────────┐
│              ESP32 DevKit                     │
│  Pan/Tilt servos · ESCs · Feeder · Sensors   │
└──────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15, React 19, TypeScript, Tailwind CSS |
| State | Zustand, React Context (event-driven pub/sub) |
| Backend | Next.js API routes, Supabase (PostgreSQL + Auth) |
| AI/CV | Python 3.11, YOLOv8 (Ultralytics), OpenCV, NumPy |
| Hardware | ESP32 DevKit, Arduino Nano, MPU6050, SHTC3, TOF sensor |
| Notifications | SendGrid / Resend (email), LINE Messaging API, Web Push |
| Infrastructure | Docker, Docker Compose |
| Testing | Playwright (E2E) |

---

## Features

- **Dual-camera fire detection** — YOLO on RGB + temperature thresholding on thermal (Y16 radiometric)
- **Autonomous turret control** — pan servo, tilt servo, dual ESCs, feeder sequence
- **Real-time MJPEG streams** — RGB and thermal feeds proxied directly to browser
- **Sensor telemetry** — IMU (pitch/roll), temperature/humidity, TOF distance
- **Web dashboard** — arm/disarm, manual pan/tilt override, threshold controls, FPS monitor
- **Fire location map** — Leaflet map with live fire markers
- **Multi-channel notifications** — email, LINE bot, web push (PWA-ready)
- **Emergency stop** — hardware E-stop with software mirror
- **Docker deployment** — containerized web and AI services

---

## Repository Structure

```
dub-fire-main/
├── software/          # Next.js app (frontend + API routes)
├── AI/                # Python detection server + serial bridge
│   ├── main.py        # Entry point
│   ├── app.py         # Core detection loop
│   ├── detection.py   # Fire detection logic (RGB + thermal)
│   ├── stream_server.py  # MJPEG + control HTTP server
│   ├── serial_bridge.py  # ESP32 communication
│   └── models/        # YOLOv8 model weights (best.pt)
├── firmware/
│   ├── esp32.ino      # Main ESP32 firmware
│   └── arduino_nano.ino  # Sensor node firmware
├── hardware/          # Bill of materials
├── 3d_files/          # Fusion 360 + STL turret models (v1, v2)
└── docker-compose.yml
```

---

## Getting Started

### Prerequisites

- Node.js 20+
- Python 3.11+
- Docker & Docker Compose (optional)
- Supabase project (database + auth)
- SendGrid or Resend API key (email notifications)
- LINE Messaging API credentials (LINE notifications)
- VAPID keys for web push

### 1. Environment Variables

Copy the template and fill in your credentials:

```bash
cp .env.template .env
```

```env
NEXT_PUBLIC_SUPABASE_URL=""
NEXT_PUBLIC_SUPABASE_ANON_KEY=""
LINE_CHANNEL_ACCESS_TOKEN=""
LINE_USER_ID=""
NEXT_PUBLIC_VAPID_PUBLIC_KEY=""
VAPID_PRIVATE_KEY=""
```

### 2. Run the Web App

```bash
cd software
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

### 3. Run the AI Detection Server

```bash
cd AI
pip install -r requirements.txt
python main.py
```

Streams and control API available at [http://localhost:5001](http://localhost:5001).

| Endpoint | Description |
|---|---|
| `/rgb_feed` | Live RGB MJPEG stream |
| `/thermal_feed` | Live thermal MJPEG stream |
| `/sensor` | Sensor snapshot (JSON) |
| `/control` | Command API (POST) |

### 4. Docker (Recommended for Production)

**Windows** — run `web` in Docker, AI on host (Docker Desktop does not reliably expose USB cameras and COM ports to Linux containers):

```bash
docker compose up --build web
python AI/main.py        # on host
```

**Linux / WSL with device passthrough:**

```bash
docker compose up --build
```

Services:
- Web: `http://127.0.0.1:3000`
- AI: `http://127.0.0.1:5001`

---

## Hardware

| Component | Role |
|---|---|
| ESP32 DevKit | Main controller — servos, ESCs, serial bridge |
| Arduino Nano | Sensor node — MPU6050, SHTC3 over I2C |
| RGB USB Camera | YOLO fire detection input |
| Thermal USB Camera | Radiometric Y16 thermal input |
| Pan servo | Continuous rotation, PWM 1100–1900 µs |
| Tilt servo (MG945) | Positional, 0–180° |
| 2× ESC | Suppression mechanism drive |
| Feeder servo | Ammunition feed sequencer |
| MPU6050 | Pitch/roll stabilization |
| SHTC3 | Ambient temperature + humidity |
| TOF Range Sensor | Distance measurement (mm) |

GPIO assignments and wiring are documented in [`software/documents/GPIO_CONNECTIONS.md`](software/documents/GPIO_CONNECTIONS.md).

---

## Firmware

Flash [`firmware/esp32.ino`](firmware/esp32.ino) to the ESP32 and [`firmware/arduino_nano.ino`](firmware/arduino_nano.ino) to the Arduino Nano using the Arduino IDE or PlatformIO.

Serial protocol (sent from Python to ESP32, 115200 baud):

| Command | Action |
|---|---|
| `A:1` / `A:0` | Arm / Disarm |
| `T:1` / `T:0` | Enable / Disable tracking |
| `S:1` | Shoot |
| `F:<x>,<y>` | Fire target coordinates (normalized) |

---

## Testing

```bash
cd software
npm run test:e2e         # Run Playwright E2E tests
npm run test:report      # View test report
```

---

## Event-Driven Architecture

The web app uses a pub/sub event bus for real-time updates across the dashboard, map, and notification services. See [`software/documents/ARCHITECTURE.md`](software/documents/ARCHITECTURE.md) for details.

Core events: `fire:status-changed`, `fire:location-added`, `fire:location-updated`, `fire:location-removed`, `fire:all-cleared`.

---

## 3D Models

Printable turret models (Fusion 360 `.f3d`, `.stl`, `.obj`) are in [`3d_files/`](3d_files/). Two versions: v1 (prototype) and v2 (final).

---

## License

This project is licensed under the [MIT License](LICENSE).
# DubFire
