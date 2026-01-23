# DJI Cloud API Demo - Complete Analysis

**Repository:** https://github.com/dji-sdk/DJI-Cloud-API-Demo  
**Version:** 1.10.0 (Released April 7, 2024)  
**Status:** DEPRECATED - Maintenance discontinued as of April 10, 2025

---

## Executive Summary

**DJI Cloud API Demo** is a reference implementation showing how to build a cloud platform that communicates with DJI commercial drones (M300 RTK, M30/M30T, Dock) via DJI Pilot 2 app.

**Key Concept:** Instead of building a custom mobile app, developers use DJI Pilot 2 (official DJI app) to communicate with their cloud platform via MQTT and HTTP.

**Target Users:**
- Enterprise drone operators (security, inspection, delivery)
- Developers building cloud-based fleet management
- Companies using DJI Dock (automated drone charging stations)

---

## Technology Stack Comparison

### DJI Cloud API Demo

| Component | Technology |
|-----------|------------|
| Language | Java 11 |
| Framework | Spring Boot 2.7.12 |
| Communication | MQTT 5.5.5 (bidirectional messaging) |
| Real-time | WebSocket |
| Database | MyBatis Plus (SQL) |
| Cache | Redis |
| File Storage | OSS (Aliyun/AWS S3/MinIO) |
| API Docs | Swagger UI |
| Build Tool | Maven |

### Our Multi-Drone System

| Component | Technology |
|-----------|------------|
| Language | C++17 |
| Framework | Custom (MAVSDK + httplib) |
| Communication | MAVLink (UDP) |
| Real-time | HTTP REST API |
| Database | None (in-memory) |
| Cache | None |
| File Storage | None |
| API Docs | Manual README |
| Build Tool | CMake |

---

## System Architecture

### DJI Cloud API Demo Architecture

```
┌─────────────────────────────────────────────────────────┐
│              DJI Pilot 2 Mobile App                     │
│         (Installed on pilot's tablet/phone)             │
└──────────────┬──────────────────────┬───────────────────┘
               │ MQTT                 │ HTTP
               ▼                      ▼
┌──────────────────────────────────────────────────────────┐
│            Your Cloud Platform (This Demo)               │
├──────────────────────────────────────────────────────────┤
│  ┌────────────┐  ┌────────────┐  ┌────────────┐         │
│  │ MQTT       │  │ WebSocket  │  │ HTTP REST  │         │
│  │ Subscriber │  │ Server     │  │ API        │         │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘         │
│        │                │               │                │
│  ┌─────┴────────────────┴───────────────┴──────┐         │
│  │        Spring Boot Application              │         │
│  │  • Device Management                        │         │
│  │  • Wayline/Mission Management               │         │
│  │  • Media/Livestream Management              │         │
│  │  • Remote Control (joystick)                │         │
│  │  • OTA Firmware Updates                     │         │
│  │  • Dock Control (charging, calibration)     │         │
│  └──────────┬──────────────┬─────────────┬──────┘         │
│             │              │             │                │
│     ┌───────▼──────┐  ┌───▼────┐  ┌────▼─────┐          │
│     │   Database   │  │ Redis  │  │   OSS    │          │
│     │  (MyBatis)   │  │ Cache  │  │ Storage  │          │
│     └──────────────┘  └────────┘  └──────────┘          │
└──────────────────────────────────────────────────────────┘
               ▲
               │ MQTT
┌──────────────┴───────────────────────────────────────────┐
│           DJI Drone Hardware                             │
│  • M300 RTK / M30 / M30T (drones)                        │
│  • DJI Dock (automated charging station)                 │
└──────────────────────────────────────────────────────────┘
```

### Our Multi-Drone System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                 JANAH Web UI                            │
│            (HTML/CSS/JavaScript)                        │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTP REST
                       ▼
┌──────────────────────────────────────────────────────────┐
│          C++ Backend Server                              │
├──────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌──────────────┐                  │
│  │ HTTP Server     │  │ DroneManager │                  │
│  │ (httplib)       │  │ (Registry)   │                  │
│  └────────┬────────┘  └──────┬───────┘                  │
│           │                   │                          │
│           └───────┬───────────┘                          │
│                   │                                      │
│        ┌──────────┴────────────┐                         │
│        ▼                       ▼                         │
│  ┌─────────────┐         ┌─────────────┐               │
│  │ Drone       │   ...   │ Drone       │               │
│  │ Controller  │         │ Controller  │               │
│  │ (MAVSDK)    │         │ (MAVSDK)    │               │
│  └──────┬──────┘         └──────┬──────┘               │
└─────────┼────────────────────────┼──────────────────────┘
          │ MAVLink (UDP)          │
          ▼                        ▼
┌──────────────────┐      ┌──────────────────┐
│  PX4/ArduPilot   │      │  PX4/ArduPilot   │
│  Drone Instance  │      │  Drone Instance  │
│  (SITL Sim)      │      │  (SITL Sim)      │
└──────────────────┘      └──────────────────┘
```

---

## Feature Comparison

### DJI Cloud API Demo Features

| Category | Features |
|----------|----------|
| Device Management | Drone/Dock online/offline status, Device topology, Property reporting (battery, GPS, etc.) |
| Mission/Wayline | Upload KMZ wayline files, Execute automated missions, Pause/resume/cancel missions, Mission progress tracking |
| Media Management | Photo/video upload from drone, Livestream control (start/stop), Media file download |
| Remote Control | Joystick control via WebSocket, Gimbal control, Camera control (zoom, focus) |
| Firmware Updates | Push firmware to drones, OTA update status tracking |
| Dock Operations | Open/close dock cover, Emergency stop, Charge control, Environmental monitoring (rain, wind) |

### Our Multi-Drone System Features

| Category | Features |
|----------|----------|
| Multi-Drone Discovery | Automatic drone detection by System ID, Dynamic drone registry |
| Basic Flight Control | ARM/DISARM, Takeoff (altitude control), Land, Return to Launch (RTL), GOTO waypoint |
| Real-time Telemetry | Position (GPS), Altitude, Battery, Flight mode, Speed, Heading |
| Multi-Map Interface | Individual drone maps, Unified "all drones" view, Click-to-navigate |
| Custom UI (JANAH) | Professional dark theme, Turquoise branding, Event logging |

---

## Key Differences

| Aspect | DJI Cloud API | Our System |
|--------|---------------|------------|
| Target Hardware | DJI commercial drones (M300, M30, Dock) | Open-source autopilots (PX4, ArduPilot) |
| Communication | MQTT (cloud-based) | MAVLink (direct UDP) |
| Mobile App | DJI Pilot 2 (official) | Not needed (direct backend) |
| Complexity | High (enterprise-grade) | Medium (research/dev) |
| Database | Required (MySQL/PostgreSQL) | Optional (in-memory only) |
| File Storage | Required (OSS/S3) | Not needed |
| Authentication | JWT tokens | None (local network) |
| Deployment | Cloud server (production) | Local machine (SITL simulation) |
| Mission Planning | KMZ wayline files | Simple lat/lon waypoints |
| Livestream | Full support (RTMP/WebRTC) | Not supported |
| OTA Updates | Full support | Not applicable (SITL) |
| Dock Integration | Full support | Not applicable |
| Learning Curve | Steep (Java + Spring + MQTT) | Moderate (C++ + MAVLink) |
| Use Case | Commercial fleets | Research/simulation |

---

## Feature Matrix

### What DJI System Has That Ours Doesn't

| Feature | Description |
|---------|-------------|
| Enterprise Fleet Management | Multi-organization support, Role-based access control (RBAC), User authentication (JWT) |
| Advanced Mission Planning | Wayline files (KMZ format), Complex flight paths, Mission templates |
| Media Management | Photo/video storage, Media download via OSS, Livestreaming (RTMP/RTSP/WebRTC) |
| Dock Integration | Automated charging, Weather monitoring, Remote dock control |
| Firmware Updates | OTA push to drones, Version management |
| Cloud Architecture | MQTT pub/sub messaging, Redis caching, Database persistence |

### What Our System Has That DJI's Doesn't

| Feature | Description |
|---------|-------------|
| Autopilot Agnostic | Works with PX4 + ArduPilot, Not locked to DJI hardware |
| Direct Control | No mobile app required, Lower latency (UDP direct) |
| Simulation-Ready | SITL integration (Gazebo, MAVProxy), No real hardware needed |
| Lightweight | No database required, Runs on single laptop, Fast startup (<1 second) |
| Custom UI (JANAH) | Fully customizable frontend, Not dependent on DJI Pilot 2 |
| Real-time Click-to-Navigate | Map-based waypoint selection, Integrated Offboard mode (PX4) |

---

## Setup Instructions

### Prerequisites

**Required:**
- Java 11+
- Maven 3.6+
- MySQL/PostgreSQL database
- Redis server
- OSS storage (Aliyun OSS / AWS S3 / MinIO)
- MQTT broker (e.g., EMQX)
- DJI Pilot 2 app (on tablet/phone)
- DJI drone or Dock hardware (M300/M30/Dock)

**Optional:**
- Docker (for containerized deployment)

### Installation Steps

**1. Clone Repository**
```bash
git clone https://github.com/dji-sdk/DJI-Cloud-API-Demo.git
cd DJI-Cloud-API-Demo
```

**2. Create Database**
```bash
mysql -u root -p < sql/cloud_api_sample.sql
```

**3. Configure Application**

Edit `sample/src/main/resources/application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/cloud_api_sample
    username: root
    password: yourpassword
  
  redis:
    host: localhost
    port: 6379

cloud-sdk:
  mqtt:
    broker-url: tcp://localhost:1883
    username: admin
    password: public
    inbound-topic: thing/product/*/services

oss:
  type: minio
  minio:
    endpoint: http://localhost:9000
    access-key: minioadmin
    secret-key: minioadmin
```

**4. Build and Run**
```bash
mvn clean package
cd sample/target
java -jar cloud-api-sample-1.10.0.jar
```

**5. Access Swagger UI**
```
http://localhost:6789/swagger-ui/index.html
```

**6. Configure DJI Pilot 2**
- Open DJI Pilot 2 app
- Go to Settings → Cloud Platform
- Enter server URL: `http://YOUR_IP:6789`
- Enter MQTT broker: `tcp://YOUR_IP:1883`

---

## When to Use Which System

### Use DJI Cloud API Demo When:

- You have DJI commercial hardware (M300, M30, Dock)
- Building production fleet management
- Need enterprise features (auth, RBAC, media)
- Want automated missions with waylines
- Need cloud deployment (AWS/Azure)
- Have Java/Spring Boot expertise
- Budget for OSS/database infrastructure

### Use Our Multi-Drone System When:

- Using open-source autopilots (PX4/ArduPilot)
- Research or education purposes
- SITL simulation (no hardware)
- Need lightweight local solution
- Want direct MAVLink control
- Prefer C++ over Java
- Lower learning curve for drone basics
- Budget-constrained (no cloud costs)

---

## Important Notice

**As of April 10, 2025, DJI has discontinued maintenance of this demo.**

**DJI's Warning:**
- Not production-ready (security risks)
- May have data leaks, unauthorized access
- Do NOT deploy to public internet
- Use for reference only

**For production, DJI recommends:**
- Contact DJI Developer Support: developer@dji.com
- Visit DJI Developer Community
- Review latest technical resources

---

## Summary

**DJI Cloud API Demo:**
- Production-grade enterprise fleet management
- For DJI commercial hardware only
- Complex but feature-rich
- Expensive (hardware + infrastructure)
- Steep learning curve

**Our Multi-Drone System:**
- Research/education focused
- Works with any PX4/ArduPilot drone
- Lightweight and fast
- Basic features only
- Easy to learn and modify

**Verdict:** These are DIFFERENT products for DIFFERENT use cases. DJI is for professional commercial operations with DJI hardware. Ours is for research/simulation with open-source autopilots. Not competitors, not interchangeable.
