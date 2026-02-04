# ATRover

A real-time rover control platform for Android-based rovers with Arduino motor controllers. The system enables remote operators to control rovers via a web dashboard while receiving live video and audio feeds over WebSocket connections.

## Architecture Overview

```
                          ┌─────────────────────┐
                          │   Web Dashboard      │
                          │  (Built-in HTML UI)  │
                          └────────┬─────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │                             │
             ┌──────▼───────┐            ┌────────▼──────┐
             │ Command Server│            │ Media Server  │
             │  (Port 8080)  │            │  (Port 8081)  │
             └──────┬───────┘            └────────┬──────┘
                    │          Redis               │
                    │    ┌──────────────┐          │
                    └────►  State Store ◄──────────┘
                         │  Pub/Sub     │
                         └──────────────┘
                    ┌──────────────────────────────┐
                    │                              │
             ┌──────▼───────┐            ┌────────▼──────┐
             │ Command Socket│            │ Media Socket  │
             │  (WebSocket)  │            │  (WebSocket)  │
             └──────┬───────┘            └────────┬──────┘
                    │                             │
                    └──────────────┬──────────────┘
                          ┌────────▼────────┐
                          │  Android Client  │
                          │  (Rover Device)  │
                          └────────┬────────┘
                                   │ USB Serial
                          ┌────────▼────────┐
                          │    Arduino       │
                          │  Motor Shield    │
                          └─────────────────┘
```

## Repositories

### [atrover-backend](https://github.com/your-org/atrover-backend)

Real-time WebSocket backend server for rover command routing and media streaming.

**Tech Stack:** TypeScript, Node.js, WebSocket (ws), Redis (ioredis)

**Key Features:**
- Dual WebSocket server architecture (command + media separation)
- Command server with rover registration, heartbeat keep-alive, and motor command routing
- Media server with binary frame protocol for efficient video/audio streaming
- Redis-backed state persistence with 5-minute TTL and cross-instance pub/sub
- Built-in web dashboard with WASD controls, live video display, motor configuration, and FPS monitoring
- Multi-instance scalability via Redis coordination
- REST API for rover listing and motor config management
- AsyncAPI documentation for both servers

**Servers:**
| Server | Default Port | Purpose |
|--------|-------------|---------|
| Command | 8080 | Rover registration, heartbeat, command routing, dashboard UI |
| Media | 8081 | Binary video/audio frame forwarding to subscribers |

**HTTP Endpoints:**
| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Health check |
| GET | `/api/rovers` | List connected rovers |
| GET/POST | `/api/rovers/:id/motor-config` | Motor mapping configuration |
| GET | `/dashboard` | Rover list dashboard |
| GET | `/dashboard/stream?roverId=<id>` | Live control panel for a rover |
| GET | `/docs/command` | AsyncAPI interactive docs (command) |
| GET | `/docs/media` | AsyncAPI interactive docs (media) |

**Quick Start:**
```bash
cd atrover-backend
npm install
redis-server --dir ./redis --daemonize yes
npm run dev
```

**Environment Variables:**
| Variable | Default | Description |
|----------|---------|-------------|
| `COMMAND_PORT` | `8080` | Command server port |
| `MEDIA_PORT` | `8081` | Media server port |
| `REDIS_URL` | `redis://localhost:6379` | Redis connection URL |
| `INSTANCE_ID` | `instance-{PID}` | Unique instance identifier |

---

### [atrover-client](https://github.com/your-org/atrover-client)

Android application for controlling a 4-wheel Arduino rover via USB serial and remote backend integration.

**Tech Stack:** Kotlin 2.0, Jetpack Compose (Material 3), CameraX, WebRTC, OkHttp WebSocket, Kotlinx Serialization

**Key Features:**
- Three control modes accessible via tab navigation:
  - **Motor Control** - Direct USB serial communication with Arduino (4 independent motors, speed sliders 0-255)
  - **Camera Streaming** - WebRTC-based live video from device camera with SDP/ICE signaling
  - **Remote Control** - Backend-integrated operation with dual WebSocket connections (command + media)
- USB serial protocol supporting single-motor, multi-motor, and direct control modes
- Camera capture with YUV-to-JPEG compression streamed as binary WebSocket frames
- Auto-reconnection with exponential backoff (capped at 30s)
- Heartbeat mechanism for connection monitoring
- Motor mapping persistence via SharedPreferences
- Reactive state management with Kotlin StateFlow

**Android Requirements:**
| Spec | Value |
|------|-------|
| Min SDK | 24 (Android 7.0) |
| Target SDK | 35 (Android 15) |
| USB Host | Required |
| Camera | Optional (for streaming) |

**USB Serial Protocol:**
```
Single Motor:  <[id][cmd][spd][chk]>
Multi Motor:   <M[n][cmd1]..[cmdn][spd][chk]>
Direct:        <D[n][id1][cmd1][spd1]..[idn][cmdn][spdn][chk]>
```

**Quick Start:**
```bash
cd atrover-client
./gradlew assembleDebug
./gradlew installDebug
```

**Arduino Sketches:** Included in `/arduino/` with firmware for dual motor control and a TFT LCD simulator for testing without hardware.

---

## Communication Protocols

### Command Protocol (JSON over WebSocket)

```
Rover → Server:   register, heartbeat, motor_config_update
Server → Rover:   registered (with UUID), heartbeat_ack, command, motor_config
Dashboard → Server: command (move/stop/rotate/calibrate)
```

### Media Protocol (Binary over WebSocket)

```
Control messages: JSON (register, start_stream, stop_stream)
Media frames:     [0x01 = video | 0x02 = audio][frame data]
```

### Rover Command Actions

| Action | Description |
|--------|-------------|
| `move` | Move in direction (forward, backward, left, right) with optional speed |
| `stop` | Stop all motors |
| `rotate` | Rotate by degrees |
| `calibrate` | Calibrate motor system |

## Development

**Prerequisites:**
- Node.js >= 18 and Redis for the backend
- Android Studio with SDK 35 for the client
- Arduino IDE for firmware upload

**Mock Clients:** The backend includes mock rover and viewer clients in `atrover-client-mock/` for testing without hardware:

```bash
cd atrover-backend/atrover-client-mock
npm install
npm run rover          # Simulates a rover with video/audio
npm run viewer -- <id> # Subscribes to a rover's stream
```
