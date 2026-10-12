# GeoSweep

> IoT-powered municipal solid-waste collection, route planning, fleet tracking, geofencing, and accountability platform.

GeoSweep is designed to help municipal authorities monitor garbage-truck operations, plan collection routes, verify stop activity, notify eligible residents when a truck approaches, and investigate operational exceptions. It connects municipal administrators, zonal officers, sanitary inspectors, society secretaries, drivers, and citizens through role-based and geographic-scope-based access control.

**Build strategy:** web and backend first, using simulated trucks. Physical ESP32/GPS hardware is an additional telemetry source and must not be required for development of the core platform.

## 1. Product goals

- Manage municipalities and their city/zone/ward/society hierarchy.
- Securely manage users, roles, permissions, and geographic scopes.
- Manage trucks, devices, routes, stops, assignments, and route geometry.
- Receive and validate GPS telemetry through MQTT.
- Store current truck state and historical telemetry.
- Use PostGIS for proximity, route, boundary, and geofence queries.
- Show authorized fleet information in a responsive municipal dashboard.
- Notify eligible residents when an assigned truck approaches their society (initial target: 500 metres, configurable by policy).
- Detect operational exceptions such as missed stops, route deviations, prolonged idling, stale telemetry, and possible device power loss.
- Support future citizen and driver mobile apps.

These are product goals, not claims that every feature is already implemented. Use `PHASES.md` for implementation order and acceptance criteria.

## 2. Architecture

```text
ESP32 + GPS + cellular (future physical source)
             │ MQTT over TLS
             ▼
        HiveMQ Cloud
             │ MQTT subscriber
             ▼
 Node.js + Express + TypeScript
 ├── Authentication verification
 ├── RBAC and geographic-scope authorization
 ├── Telemetry validation and normalization
 ├── Geofence/event processing
 ├── REST APIs
 └── Socket.io authorized updates
             │
             ▼
 Supabase PostgreSQL + PostGIS
 ├── users/profiles and authorization data
 ├── cities, zones, wards, societies
 ├── trucks, routes, stops, assignments
 ├── latest truck state and telemetry history
 ├── alerts and audit records
 └── spatial queries and indexes
             │
      ┌──────┴─────────┐
      ▼                ▼
React admin web   React Native / Expo
+ Leaflet         citizen and driver apps (later)
                       │
                       ▼
                Firebase Cloud Messaging
```

The public product website is a separate presentation surface and may be deployed on Vercel. The admin dashboard, backend API, database, MQTT broker, realtime layer, and mobile apps have distinct responsibilities.

### Architectural boundaries

- **React clients:** presentation, form validation for usability, navigation, and API/realtime consumption. They are not security boundaries.
- **Express backend:** application business logic, validation, authorization, REST endpoints, MQTT ingestion, event processing, and realtime authorization.
- **Supabase:** managed PostgreSQL and PostGIS; Supabase Auth may be used for phone OTP identity if the implementation follows the documented authentication decision below.
- **HiveMQ Cloud:** secure MQTT message broker, not the application database.
- **Socket.io:** authorized live updates to connected web/mobile clients.
- **FCM:** mobile push delivery; notification eligibility is decided by the backend.
- **Simulator:** emits the same normalized telemetry contract as the eventual physical device.

## 3. Committed technology stack

| Layer | Technology | Responsibility |
|---|---|---|
| Firmware | ESP32, Arduino C++ | Read sensors and publish device telemetry |
| GPS | NEO-6M or compatible module | Location readings |
| Cellular | SIM800L or compatible module | Cellular connectivity where supported |
| Device MQTT | PubSubClient or compatible MQTT client | Publish telemetry securely |
| Broker | HiveMQ Cloud | MQTT transport and topic access control |
| Backend | Node.js, Express, TypeScript | APIs and business logic |
| Database | Supabase PostgreSQL + PostGIS | Relational and geospatial persistence |
| Authentication | Supabase Auth phone OTP, verified by backend | Identity proofing and session validation |
| Realtime | Socket.io | Authorized live client updates |
| Web | React, Vite, TypeScript | Municipal admin dashboard and public site |
| UI | Tailwind CSS | Responsive styling |
| Maps | Leaflet, React-Leaflet, OpenStreetMap tiles | Fleet map, route planning, and stops |
| Mobile | React Native, Expo, `react-native-maps` | Citizen and driver apps, later |
| Push | Firebase Cloud Messaging | Mobile notifications |
| Hosting | Free-tier services initially; provider selected per deployment needs | Hosting and delivery |

Do not reintroduce Java/Spring Boot into this project unless the user explicitly changes the stack. Keep versions compatible and record important version choices in package manifests and docs.

## 4. Authentication and authorization

GeoSweep uses a hierarchy. Users do not select their own role on the login screen; their verified identity is mapped to a profile and permissions on the server.

### Roles

1. `PLATFORM_SUPER_ADMIN` — platform-wide onboarding and municipal tenant management.
2. `CITY_ADMIN` — authorized municipality administration.
3. `ZONAL_OFFICER` — assigned zones/wards and operational oversight.
4. `SANITARY_INSPECTOR` — assigned routes/stops, device-to-truck pairing, and collection review.
5. `SOCIETY_SECRETARY` — assigned society activity and exception reporting.
6. `DRIVER` — assigned route and stop workflow.
7. `CITIZEN` — only information and notifications permitted for their verified/assigned society.

Role labels are centralized in the backend/database. The frontend may mirror labels for navigation, but the backend remains authoritative.

### Authorization layers

Every protected operation must validate:

1. The session/token is valid.
2. The user is active.
3. The user's role has the required permission.
4. The requested record is within the user's geographic scope.
5. Any ownership or assignment constraints are satisfied.

Use scope identifiers such as `city_id`, `zone_id`, `ward_id`, and `society_id` where relevant. Enforce checks in backend services and queries. Apply Supabase Row Level Security (RLS) as a defense-in-depth control where applicable; never assume frontend filtering or RLS alone replaces application authorization.

### Supabase Auth integration rule

If Supabase Auth is used for phone OTP, the client may complete the provider's supported OTP flow, but the backend must verify the supplied access token using the supported verification method before trusting identity claims. The backend then loads the authoritative GeoSweep profile, role, scope, and status. Do not accept role/scope claims directly from client-submitted JSON. Never expose the Supabase service-role key to browser or mobile clients.

## 5. Geospatial and telemetry design

PostGIS is required for geographic features. Use appropriate `geometry`/`geography` types, SRIDs, and spatial indexes based on query needs. Do not store route geometry as arbitrary JSON when a proper spatial column is appropriate.

Core geospatial use cases:

- Stops within a configured distance of a truck.
- Validate stops against city/ward boundaries.
- Determine whether a truck enters or exits a geofence.
- Find trucks near an authorized society.
- Evaluate route deviation and stop arrival.

A 500-metre approach event must not be generated on every GPS update. Implement event deduplication, entry/exit state, timestamps, configurable radius, and notification cooldowns. Proximity alone is not proof that a collection occurred.

### Normalized telemetry contract (initial proposal)

```json
{
  "deviceId": "TEST_DEVICE_001",
  "latitude": 18.65,
  "longitude": 73.77,
  "speedKph": 22.5,
  "headingDegrees": 120,
  "recordedAt": "2026-10-12T10:30:00Z",
  "sequence": 123
}
```

This is an illustrative development contract. Finalize and version the schema before simulator/hardware implementation. The server must map an authenticated device to its assigned truck; it must not trust a client-supplied city or truck ID as proof of authorization.

Telemetry processing should validate device identity, coordinates, timestamps, duplicate/out-of-order messages, and device status. Maintain a fast latest-position record separately from history. Establish a retention policy before storing high-frequency location data indefinitely.

### MQTT topic concept

```text
geosweep/{cityId}/{deviceId}/telemetry
geosweep/{cityId}/{deviceId}/alert
```

Treat this as a proposed topic pattern until the broker ACL and device identity model are finalized. Use TLS, unique device credentials, least-privilege topic permissions, payload validation, and credential rotation. Do not embed shared production credentials in firmware repositories.

## 6. Suggested repository structure

```text
GeoSweep/
├── README.md
├── PHASES.md
├── .gitignore
├── .env.example
├── backend/
│   ├── src/
│   │   ├── app.ts
│   │   ├── config/
│   │   ├── middleware/
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   ├── users/
│   │   │   ├── roles/
│   │   │   ├── cities/
│   │   │   ├── zones/
│   │   │   ├── wards/
│   │   │   ├── societies/
│   │   │   ├── trucks/
│   │   │   ├── routes/
│   │   │   ├── stops/
│   │   │   ├── telemetry/
│   │   │   ├── alerts/
│   │   │   └── audit/
│   │   ├── mqtt/
│   │   ├── realtime/
│   │   ├── services/
│   │   └── utils/
│   └── tests/
├── dashboard/                 # React admin dashboard
├── public-website/             # Public landing/pilot-request website
├── mobile/                     # Expo apps, later
├── firmware/                   # ESP32 firmware, later
├── simulator/                  # Fake truck telemetry publisher
├── supabase/
│   ├── migrations/
│   ├── seed.sql
│   └── config.toml             # only if using Supabase CLI
└── docs/
    ├── architecture/
    ├── api/
    ├── database/
    ├── mqtt/
    └── decisions/
```

This is the target organization, not a mandate to create empty folders for every future feature immediately. Create a folder when its phase begins. If an existing repository uses a different agreed folder name, inspect it and update docs deliberately instead of duplicating applications.

## 7. Free-first infrastructure strategy

The project should remain on free tiers during learning and prototype development wherever feasible. Do not introduce a paid service without explaining the need and obtaining approval.

Potential initial setup:

- GitHub for source control.
- Supabase Free for PostgreSQL/PostGIS and optional Auth.
- Local Node.js backend and Vite frontend during development.
- HiveMQ Cloud free offering if its current limits and account eligibility fit testing.
- Vercel or another suitable free-tier static host for the web frontend.
- Free/local map libraries; use public map tile services in accordance with their usage policy.
- Firebase FCM for push notifications, subject to its current product terms.
- Docker locally where useful.

Free tiers change and may include pausing, quotas, project limits, egress restrictions, or production-use constraints. Check provider pricing/limits before committing to a deployment. Do not promise uninterrupted production service on a free tier.

Supabase's commonly documented Free plan limits have included 500 MB database size per project and 1 GB file storage, along with limits on egress, active users, and realtime usage. **Verify current official pricing before relying on these numbers**, because limits can change. GeoSweep's high-frequency telemetry may consume database capacity faster than profile and configuration records. Keep retention, sampling, and archival in mind from the start.

Use environment variables and separate development/production projects when possible. Keep business logic provider-independent so database hosting or backend hosting can be upgraded later without rewriting domain logic.

## 8. Environment configuration

Maintain `.env.example` with variable names and safe placeholders only. Keep `.env` out of Git.

Expected configuration categories may include:

```dotenv
NODE_ENV=development
PORT=4000
APP_ORIGIN=http://localhost:5173
DATABASE_URL=
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
SUPABASE_JWT_SECRET=
HIVEMQ_HOST=
HIVEMQ_PORT=
HIVEMQ_USERNAME=
HIVEMQ_PASSWORD=
FIREBASE_PROJECT_ID=
FIREBASE_CLIENT_EMAIL=
FIREBASE_PRIVATE_KEY=
```

These are configuration examples, not a requirement to use every variable in every phase. Document exactly which variables each service needs. The service-role key, database credentials, MQTT secrets, and Firebase private credentials are server-only. Do not put them in `VITE_*` variables or ship them to clients.

## 9. Development principles

- Build small, verifiable vertical slices.
- Use TypeScript strict mode and shared request/response types where useful.
- Keep routes/controllers thin; put business logic in domain services.
- Validate all external input (HTTP, MQTT, realtime subscriptions).
- Use centralized error handling and structured logging.
- Use database migrations as the schema source of truth.
- Add indexes based on actual access patterns; spatial queries need appropriate spatial indexes.
- Keep latest truck state separate from telemetry history.
- Make simulator messages match the normalized device contract.
- Do not put authorization rules only in React.
- Do not claim a feature works until it has been tested.
- Do not commit secrets, real citizen data, or production telemetry in seed files.
- Update README, API docs, migrations, and `PHASES.md` when architecture decisions change.

## 10. First milestone

The first milestone is a complete working web/backend slice without physical hardware:

1. Super Admin signs in.
2. Super Admin creates a city/municipality.
3. Super Admin creates a City Admin and assigns the city scope.
4. City Admin signs in and sees only that city's records.
5. A simulated truck sends test telemetry through the planned ingestion path.
6. The backend validates and persists the position.
7. The dashboard displays the truck and indicates stale telemetry correctly.
8. A nearby-stop event is deduplicated and sent only to eligible authorized clients.
9. Cross-city/scope access attempts are rejected by the backend.

This milestone proves the architecture before hardware integration. The eventual ESP32 device becomes another source of messages, not a reason to redesign the application.

## 11. Related documents

- [`PHASES.md`](./PHASES.md) — phased implementation plan, agent instructions, and acceptance criteria.
- `docs/architecture/` — architecture diagrams and decisions.
- `docs/api/` — API contracts.
- `docs/database/` — schema and retention decisions.
- `docs/mqtt/` — topic, device identity, and message contracts.

## 12. Project status

Treat features as planned until code and tests demonstrate they work. Update this section as real milestones are completed; do not mark features complete based only on generated files or mock UI.
