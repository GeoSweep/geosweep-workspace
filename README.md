# GeoSweep

[![Organization](https://img.shields.io/badge/GitHub_Organization-GeoSweep-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/GeoSweep)
[![Backend](https://img.shields.io/badge/Backend-Node.js_%2B_Express_%2B_TS-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)](https://github.com/GeoSweep/geosweep-backend)
[![Web](https://img.shields.io/badge/Web_Portal-React_%2B_Vite_%2B_Tailwind-61DAFB?style=for-the-badge&logo=react&logoColor=black)](https://github.com/GeoSweep/geosweep-web)
[![Mobile](https://img.shields.io/badge/Mobile_App-React_Native_%2B_Expo-000020?style=for-the-badge&logo=expo&logoColor=white)](https://github.com/GeoSweep/geosweep-mobile)
[![Firmware](https://img.shields.io/badge/Hardware-ESP32_%2B_GPS_%2B_GSM-E7352C?style=for-the-badge&logo=espressif&logoColor=white)](https://github.com/GeoSweep/geosweep-firmware)
[![Database](https://img.shields.io/badge/Database-Supabase_PostgreSQL_%2B_PostGIS-3ECF8E?style=for-the-badge&logo=supabase&logoColor=white)](https://supabase.com/)
[![MQTT](https://img.shields.io/badge/Broker-HiveMQ_Cloud_TLS-F25F22?style=for-the-badge&logo=mqtt&logoColor=white)](https://www.hivemq.com/)

> **Industrial-grade, IoT-powered municipal solid-waste collection, automated geofencing, route compliance verification, approach notification radar, and operational accountability platform for Urban Local Bodies (ULBs).**

---

## 🏛️ Polyrepo Architecture & Repository Directory

GeoSweep is organized as an enterprise polyrepo under the [GeoSweep GitHub Organization](https://github.com/GeoSweep). Each component is independently versioned, typed, tested, and deployable:

| Repository | GitHub URL | Technology Stack | Core Responsibility |
| :--- | :--- | :--- | :--- |
| **`geosweep-backend`** | [`GeoSweep/geosweep-backend`](https://github.com/GeoSweep/geosweep-backend) | Node.js, Express, TypeScript, Supabase PostGIS, Socket.io, HiveMQ | Telemetry ingestion, data normalization, RBAC & municipal scope enforcement, REST API, WebSocket streams |
| **`geosweep-web`** | [`GeoSweep/geosweep-web`](https://github.com/GeoSweep/geosweep-web) | React 18, Vite, TypeScript, Tailwind CSS, Leaflet, Lucide React | Municipal command center, real-time vehicle radar, route/stop planner, and public citizen landing page |
| **`geosweep-mobile`** | [`GeoSweep/geosweep-mobile`](https://github.com/GeoSweep/geosweep-mobile) | React Native, Expo SDK, TypeScript, Firebase Cloud Messaging | Citizen 500m approach radar notifications, society ETA, and driver sequential stop collection checklist |
| **`geosweep-firmware`**| [`GeoSweep/geosweep-firmware`](https://github.com/GeoSweep/geosweep-firmware) | ESP32 C++, PlatformIO, Arduino, TinyGSM, TinyGPSPlus | Physical vehicle tracking unit, GPS NMEA parsing, cellular SIM800L / WiFi failover, OLED diagnostics |
| **`geosweep-workspace`**| [`GeoSweep/geosweep-workspace`](https://github.com/GeoSweep/geosweep-workspace) | npm workspaces, concurrently | Central developer workspace, multi-service dev orchestration, and master development roadmap |

---

## 🛰️ End-to-End System Data Flow

```mermaid
flowchart TD
    subgraph Vehicle Tracking Hardware
        GPS[NEO-6M GPS Receiver] -->|NMEA Coordinates| ESP32[ESP32 Hardware Node\nvehicle tracker in PCMC]
        SIM[SIM800L Cellular / WiFi] <--> ESP32
        ESP32 -->|OLED Live HUD| Screen[SSD1306 Diagnostic Display]
    end

    subgraph Secure Cloud Broker
        ESP32 -->|TLS MQTTS :8883\ngeosweep/pcmc/truck_001/telemetry| HiveMQ[HiveMQ Cloud Broker]
    end

    subgraph Node.js Backend Engine :4000
        HiveMQ -->|TLS Subscriber| Ingestion[MQTT Ingestion Service\nsrc/mqtt/mqtt.service.ts]
        Ingestion --> TelemetryStore[Normalized Telemetry Store]
        TelemetryStore --> PostGIS[(Supabase PostgreSQL + PostGIS\nSpatial Indexes & Tables)]
        TelemetryStore -->|Realtime Stream| SocketServer[Socket.io Authorized Rooms\nsrc/realtime/socket.ts]
        ExpressAPI[Express REST API\n/api/auth, /api/fleet, /api/cities] --> SocketServer
    end

    subgraph Client Applications
        SocketServer -->|telemetry:update| WebAdmin[geosweep-web :5173\nLeaflet Fleet Command Center]
        SocketServer -->|telemetry:update| MobileApp[geosweep-mobile\nCitizen Radar & Driver Checklist]
        WebAdmin -->|Vite Proxy /api| ExpressAPI
        MobileApp -->|REST API /api| ExpressAPI
    end
```

---

## ⚡ Quickstart: Running the Entire Platform Locally

The root workspace includes unified developer orchestration with `concurrently`. You can start both the backend and web dashboard simultaneously:

```bash
# 1. Clone the master workspace
git clone https://github.com/GeoSweep/geosweep-workspace.git GeoSweep
cd GeoSweep

# 2. Install workspace dependencies
npm install

# 3. Start both Backend (:4000) and Web (:5173) with a single command:
npm run dev
```

### Available Root Commands

| Command | Action |
| :--- | :--- |
| `npm run dev` | **Runs both Node.js Backend and React Web Dashboard concurrently** |
| `npm run dev:backend` | Runs only `geosweep-backend` on port `4000` |
| `npm run dev:web` | Runs only `geosweep-web` on port `5173` |
| `npm run build` | Builds production artifacts for both backend and web |
| `npm run typecheck` | Strict TypeScript compilation checks across all TypeScript code |
| `npm test` | Runs backend unit and integration test suite |

---

## 📋 Phased Implementation Roadmap

GeoSweep is developed following a rigorous 18-phase implementation specification documented in [`PHASES.md`](./PHASES.md):

- [x] **Phase 00:** Polyrepo & Developer Foundation
- [x] **Phase 01:** Supabase Database & Schema Foundation (PostGIS)
- [o] **Phase 02:** Authentication, Profiles, RBAC & Scope Enforcement
- [o] **Phase 03:** Super Admin & Municipal Tenant Onboarding
- [o] **Phase 04:** Zones, Wards, Societies & Scope Filters
- [o] **Phase 05:** Fleet & Device Registry
- [o] **Phase 06:** Routes & Collection Stops (Spatial Ordering)
- [o] **Phase 07:** Admin Dashboard & Route-Planning Map
- [o] **Phase 08:** Telemetry Simulator & Data Normalization
- [o] **Phase 09:** Realtime Updates with Socket.io
- [ ] **Phase 10:** Geofencing, Approach Events & Incidents
- [x] **Phase 11:** Physical MQTT & ESP32 Integration (Live Hardware Verified)
- [ ] **Phase 12:** Push Notifications (Firebase FCM)
- [ ] **Phase 13:** Mobile Applications (Citizen & Driver Expo Apps)
- [o] **Phase 14:** Reports, Audit Trail & Accountability
- [o] **Phase 15:** Public Website & Landing Presentation
- [ ] **Phase 16:** Production Hardening & Deployment
- [ ] **Phase 17:** Controlled Real-World Municipal Pilot

---

## 📖 Architecture Decision Records (ADRs)

Architecture decisions are documented under [`docs/decisions/`](./docs/decisions/):
- [`0001-monorepo-and-technology-stack.md`](./docs/decisions/0001-monorepo-and-technology-stack.md): Polyrepo structure, Node.js + Express, Supabase PostGIS, React/Vite, React Native/Expo, and ESP32 PlatformIO stack commitment.
