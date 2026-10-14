# ADR 0001: Monorepo Architecture and Core Technology Stack

## Status
Accepted

## Date
2026-10-12

## Context
GeoSweep is an IoT-powered municipal solid-waste collection, route planning, fleet tracking, geofencing, and accountability platform. It connects municipal administrators, zonal officers, sanitary inspectors, society secretaries, drivers, and citizens through role-based and geographic-scope-based access control.

To ensure rapid, reliable development and clear boundary separation between the municipal dashboard and backend business logic, we need an agreed monorepo architecture and technology stack that avoids architectural regressions (e.g., reverting to Java/Spring Boot or mixing server secrets into client bundles).

## Decision

1. **Repository Layout**:
   - Organized as a multi-repository (polyrepo) ecosystem under the `GeoSweep` GitHub organization:
     - `geosweep-backend/`: Node.js, Express, TypeScript REST API and Socket.io engine (`https://github.com/GeoSweep/geosweep-backend.git`).
     - `geosweep-web/`: React, Vite, TypeScript, Tailwind CSS municipal administration client & dashboard (`https://github.com/GeoSweep/geosweep-web.git`).
     - `geosweep-mobile/`: React Native / Expo citizen and driver mobile applications (`https://github.com/GeoSweep/geosweep-mobile.git`).
     - `geosweep-firmware/`: ESP32 C++ hardware telemetry module (`https://github.com/GeoSweep/geosweep-firmware.git`).
   - Every repository maintains its own standalone configuration, `README.md`, `PHASES.md`, `.gitignore`, and `.env.example`.

2. **Core Technology Stack**:
   - **Backend**: Node.js (v20+), Express, TypeScript in strict mode, Zod for validation, Helmet and CORS for security.
   - **Database**: Supabase PostgreSQL + PostGIS (with SQL migrations as the single source of truth).
   - **Authentication & Authorization**: Supabase Auth token verification on the backend; backend is the strict security boundary enforcing RBAC and geographic scope (`cityId`, `zoneId`, `wardId`, `societyId`).
   - **Frontend**: React + Vite + TypeScript + Tailwind CSS, with Lucide icons and Axios/Zustand.

3. **Development Loop Baseline (Phase 00)**:
   - Git branching strategy: `main` (production-ready releases) and `develop` (integration baseline).
   - Backend provides `GET /api/health` returning system status, timestamp, uptime, and service identity.
   - Dashboard provides a connection verification shell displaying backend connectivity and diagnostic details.
   - No real credentials or secrets committed; `.env.example` templates define all required configuration keys.

## Consequences
- Single development environment enables running backend and dashboard together or independently.
- Strict TypeScript configuration ensures type safety across client and server boundaries.
- Adheres strictly to the GeoSweep phased implementation roadmap (`PHASES.md`).
