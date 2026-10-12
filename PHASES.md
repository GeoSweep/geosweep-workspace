# GeoSweep — Phased Development Plan

This document is the implementation source of truth for developers and AI coding agents (including Antigravity). Read `README.md` and this file before making repository changes.

The committed stack is **Node.js + Express + TypeScript + Supabase PostgreSQL/PostGIS + React/Vite/TypeScript**. Do not mix in the earlier Java/Spring Boot architecture. Hardware is delayed by design; build and test the platform using simulated telemetry first.

---

# A. Universal rules for Antigravity and all coding agents

## A1. Inspect before editing

Before writing code:

1. Read `README.md` and `PHASES.md`.
2. Inspect the repository tree and package manifests.
3. Identify the current phase and its acceptance criteria.
4. Read relevant modules, migrations, API contracts, and tests.
5. Check whether a similar component, API, table, or utility already exists.
6. Propose a short plan listing files to create/change and tests to run.

Do not assume a file exists because it is listed in the target structure. Inspect the actual repository.

## A2. Do not hallucinate requirements

Never silently invent roles, permissions, columns, API routes, MQTT topics, message fields, screens, integrations, business rules, secrets, or hardware behavior.

If a detail is unspecified:

- Search the existing code and docs first.
- Follow the smallest consistent implementation for a low-impact detail.
- Record the decision in the appropriate documentation if it affects future work.
- Ask the user before making a decision that changes security, data model, architecture, billing, or externally visible behavior.

Do not present assumptions as established requirements.

## A3. Work only on the requested phase

Do not implement later-phase features early. In particular, do not add real MQTT ingestion, physical firmware, FCM, citizen mobile apps, or advanced geofencing while working on basic auth/RBAC unless explicitly requested.

## A4. Preserve working code

- Do not rewrite the whole repository for a small change.
- Do not delete working features or migrations.
- Do not duplicate the dashboard or backend under a second folder.
- Follow the repository's existing naming and patterns unless they conflict with this document; report material conflicts.
- Keep changes small enough to review.

## A5. Backend is the security boundary

React visibility checks are UX only. Every protected API, record query, Socket.io room, and notification target must be authorized on the server.

Never trust client-provided `role`, `cityId`, `zoneId`, `wardId`, `societyId`, `truckId`, or device identity as proof of access. Resolve the authenticated user and scope from trusted server-side data.

## A6. Supabase rules

- Supabase is the managed PostgreSQL/PostGIS platform and may provide phone OTP authentication.
- Browser/mobile clients must never connect directly to PostgreSQL.
- The backend must verify access tokens before trusting identity claims.
- Load GeoSweep roles, status, and geographic scope from authoritative data.
- Never expose the service-role key, database password, or other server credentials to a client.
- Use RLS as defense in depth where appropriate; do not treat frontend filters as security.
- Use migrations as the schema source of truth.

## A7. Secrets and configuration

- Commit `.env.example`, never real `.env` files.
- Never hard-code API keys, Supabase service-role keys, database passwords, MQTT credentials, JWT secrets, or Firebase private keys.
- Do not place secrets in frontend `VITE_*` variables.
- Do not invent credentials or claim external services are configured without checking.
- If a paid service is proposed, explain why and ask before introducing it.

## A8. No fake integration

Mocks are allowed only when the current phase calls for them. Label mock data clearly and keep it isolated. Do not pretend an API, broker, notification, or database integration is live when it is not. Remove or replace mock paths when real integration is implemented.

## A9. Dependencies and versions

Before adding a dependency, check whether the repository already has a suitable package. Add only what the current phase needs. Avoid unnecessary major-version upgrades during a phase. Keep lockfiles in sync with package manifests.

## A10. Test and report honestly

After a meaningful change, run the available checks, such as:

- backend type-check/build
- backend unit/integration tests
- frontend type-check/build
- migration validation
- relevant API test
- authorization negative test
- UI flow check

Never report a check as passed if it was not run. If blocked by missing credentials/services, report exactly what could and could not be verified.

## A11. Required agent response format

For each implementation task, finish with:

```text
PHASE:
<phase name/number>

REQUIREMENT:
<what was implemented>

FILES CREATED:
- ...

FILES CHANGED:
- ...

DATABASE / API CHANGES:
- ...

VALIDATION:
- Backend checks: PASS / FAIL / NOT RUN
- Frontend checks: PASS / FAIL / NOT RUN
- API/manual checks: PASS / FAIL / NOT RUN
- Authorization checks: PASS / FAIL / NOT RUN

LIMITATIONS / ASSUMPTIONS:
- ...

NEXT ACCEPTANCE CRITERION:
- ...
```

---

# B. Global architecture contract

## Stack

**Backend:** Node.js, Express, TypeScript. Organize code by domain/module.

**Database:** Supabase PostgreSQL + PostGIS. Use SQL migrations, constraints, and indexes.

**Auth:** Supabase Auth phone OTP where appropriate; backend verifies tokens and loads GeoSweep authorization profile.

**Web:** React, Vite, TypeScript, Tailwind CSS, React Router, Axios, Zustand, React-Leaflet, Leaflet, Lucide React.

**Realtime:** Socket.io, with server-authorized room membership and payload scope.

**IoT later:** HiveMQ Cloud, MQTT client for Node.js, ESP32 C++ firmware.

**Mobile later:** React Native, Expo, `react-native-maps`, Firebase Cloud Messaging.

Do not switch this stack without explicit user approval.

## Domain hierarchy

```text
Platform Super Admin
  └── City Admin
       └── Zonal Officer
            └── Sanitary Inspector
                 ├── Route / Truck / Stop assignments
                 └── Society Secretary (society scope)
                      └── Citizen (restricted society access)

Driver → only assigned truck/route/stop workflow
```

This is a permission and operational relationship, not necessarily a single parent-child table. Model actual user scopes and assignments explicitly. Do not assume every role has exactly one scope unless requirements establish that.

## Target repository shape

```text
GeoSweep/
├── README.md
├── PHASES.md
├── .env.example
├── backend/
├── dashboard/
├── public-website/
├── mobile/          # later
├── firmware/        # later
├── simulator/
├── supabase/
│   ├── migrations/
│   └── seed.sql
└── docs/
```

Do not create all future folders as empty placeholders. Add them when their phase begins. If the repository currently uses `admin-web/` rather than `dashboard/`, do not create a duplicate; either keep the existing agreed name or rename deliberately with all references updated.

## Domain-oriented backend organization

```text
backend/src/
├── app.ts
├── config/
├── middleware/
├── modules/
│   ├── auth/
│   ├── users/
│   ├── roles/
│   ├── cities/
│   ├── zones/
│   ├── wards/
│   ├── societies/
│   ├── trucks/
│   ├── routes/
│   ├── stops/
│   ├── telemetry/
│   ├── alerts/
│   └── audit/
├── mqtt/            # later
├── realtime/
└── utils/
```

Within a module, separate routes/controller handlers, validation schemas, service/business logic, and repository/data access when useful. Avoid both giant monolithic files and needless layers for trivial code.

---

# PHASE 00 — Repository and developer foundation

## Goal

Create a clean, reproducible monorepo and verify that the local development loop works.

## Tasks

- Initialize Git with `main` and `develop` branches.
- Create or verify `backend/` and the React dashboard directory.
- Add `.gitignore`, `.env.example`, root documentation, and package scripts.
- Configure TypeScript strict mode, formatting/linting appropriate to the selected tools, and consistent naming.
- Add a backend health endpoint, for example `GET /api/health`.
- Add a minimal frontend shell that calls the health endpoint only if the API is running; display loading/error states.
- Document local setup commands and required environment variables.
- Create `docs/decisions/` for decisions that affect architecture.

## Acceptance criteria

- Backend starts locally.
- Frontend starts and builds.
- TypeScript checks pass.
- Health endpoint returns a documented response.
- No real secrets are tracked by Git.
- A new developer can follow README setup steps without guessing.

Do not implement business modules or MQTT in this phase.

---

# PHASE 01 — Supabase database and schema foundation

## Goal

Set up PostgreSQL/PostGIS, migrations, minimal seed data, and a documented relational model.

## Tasks

- Create/connect a Supabase project or document local development configuration.
- Enable PostGIS through a migration or documented database setup step.
- Establish consistent primary-key, timestamp, naming, and foreign-key conventions.
- Create only the initial tables required for authentication/profile and municipal hierarchy.
- Proposed initial domains: profiles, cities, zones, wards, societies, roles/permissions or role assignments, and user scopes. Finalize table shape based on the auth and RBAC design before creating migrations.
- Add appropriate unique constraints and indexes.
- Add development seed data without real personal information.
- Document how migrations are applied and how a clean database is initialized.

## Rules

- Do not use an ORM or schema generator as a substitute for reviewed migrations.
- Never edit an already-applied migration to change deployed schema; add a new migration.
- Do not store secrets in `seed.sql`.
- Avoid creating telemetry tables before the telemetry model and retention policy are defined.

## Acceptance criteria

- A clean database can be initialized from migrations.
- Foreign keys and uniqueness rules work.
- PostGIS is enabled.
- The schema is documented.
- Seed data is deterministic and clearly development-only.

---

# PHASE 02 — Authentication, profiles, RBAC, and geographic scope

## Goal

Implement phone OTP login and enforce authorization on the backend.

## Auth contract

- Use Supabase Auth phone OTP if the chosen project configuration supports it.
- Verify tokens server-side using a supported verification mechanism.
- Resolve the GeoSweep profile, active status, role, permissions, and scope from trusted records.
- Users cannot choose their role at login.
- If OTP provider credentials or phone delivery are not configured, document the blocker and use a clearly labelled development-only mock flow; never claim real SMS works.

## Roles

- `PLATFORM_SUPER_ADMIN`
- `CITY_ADMIN`
- `ZONAL_OFFICER`
- `SANITARY_INSPECTOR`
- `SOCIETY_SECRETARY`
- `DRIVER`
- `CITIZEN`

## Tasks

- Implement authentication middleware.
- Implement a centralized permission/scope authorization helper.
- Implement `GET /api/me` or the equivalent current-profile endpoint.
- Protect API routes by permission and record scope.
- Define how roles and scopes are assigned and audited.
- Add negative tests for invalid/expired token, inactive user, missing permission, and out-of-scope access.
- Decide and document token/session handling; do not invent a custom token format.

## Acceptance criteria

- Invalid or expired credentials are rejected.
- User role is never accepted from a login form as authoritative.
- Backend resolves roles and scopes from trusted data.
- Out-of-scope API requests are rejected even when manually constructed.
- Service-role and other server secrets are never exposed to clients.

---

# PHASE 03 — Super Admin and municipal onboarding

## Goal

Deliver the first complete administrative slice.

## Screens

- Login/OTP
- Dashboard shell
- Cities
- Users
- Roles/assignments as required by the approved data model

## APIs

Design endpoint names after inspecting existing routes. Expected capabilities include:

- list/create city
- list/create/update/deactivate user/profile
- assign role and geographic scope
- retrieve current user's profile and permitted navigation data

Do not create duplicate endpoints if equivalent routes already exist.

## First end-to-end flow

```text
Platform Super Admin signs in
  ↓
Creates a city / ULB
  ↓
Creates a City Admin profile
  ↓
Assigns the City Admin to that city
  ↓
City Admin signs in
  ↓
City Admin can access only that city's data
```

## Acceptance criteria

- Data comes from the database, not permanent frontend mock arrays.
- Form loading, validation, success, empty, and error states work.
- A City Admin cannot access another city by changing a URL or API identifier.
- Sensitive user/role/scope changes are audit logged where the audit module is available.

---

# PHASE 04 — Zones, wards, societies, and scope enforcement

## Goal

Build the municipal hierarchy and constrain data access at each level.

## Tasks

- CRUD for zones, wards, and societies as authorized by role.
- Validate parent-child relationships (a ward belongs to a valid zone/city according to the approved schema).
- Implement scope-filtered queries in backend services.
- Add tests for cross-city, cross-zone, cross-ward, and cross-society access.
- Add admin UI for the permitted operations only.

## Acceptance criteria

- Parent-child relationships are validated by the backend and database.
- A user cannot reassign a record outside their authority.
- Scope is not enforced only by frontend filtering.

---

# PHASE 05 — Fleet and device registry

## Goal

Manage trucks and their device identities before physical telemetry is required.

## Tasks

- Add truck CRUD and controlled status values.
- Store registration number and appropriate municipal scope.
- Add device registry/pairing only after deciding device identity and authorization rules.
- Ensure a device maps to an authorized truck server-side.
- Seed clearly fake development trucks.

## Acceptance criteria

- Duplicate registration/device identity is rejected according to the chosen constraints.
- Users can manage only trucks in their scope.
- Device identity is not trusted merely because it appears in a payload.

---

# PHASE 06 — Routes and collection stops

## Goal

Create routes, stop ordering, truck/driver assignments, and spatial records.

## Tasks

- Define route and stop data contracts.
- Use appropriate PostGIS geometry/geography types and SRIDs.
- Add migrations and spatial indexes for actual query patterns.
- Implement route draft/publish behavior if confirmed by the user requirements.
- Add stop coordinates, sequence, and only the operational fields currently required.
- Validate route, truck, driver, stop, and municipal-scope assignments.

## Acceptance criteria

- Routes and stops persist and reload correctly.
- Invalid coordinates and invalid parent/assignment relationships are rejected.
- Out-of-scope route/stop modifications fail.
- API contracts are documented.

---

# PHASE 07 — Admin dashboard and route-planning map

## Goal

Provide the first useful map and operational web interface.

## Tasks

- Build shared dashboard layout, navigation, role-aware menu, and reusable form/table patterns.
- Add React-Leaflet map.
- Display authorized trucks, routes, and stops from real APIs.
- Support route drawing/editing and stop placement only using a library and interaction model verified in the repository.
- Save edits through backend APIs; do not treat client map state as persisted data.
- Show loading, error, empty, and stale-data states.
- Attribute and comply with the chosen map tile provider's policy. Do not assume public OpenStreetMap tiles are unlimited production infrastructure.

## Acceptance criteria

- Map shows only authorized records.
- Route/stop edits survive refresh.
- Backend validates the saved geometry and assignments.
- Existing CRUD screens remain functional.

---

# PHASE 08 — Telemetry simulator and normalized data contract

## Goal

Exercise the end-to-end telemetry path before hardware arrives.

## Tasks

- Finalize and document a versioned telemetry payload.
- Implement a simulator for two or three fake trucks.
- Support deterministic route replay, pauses, missing updates, delayed/duplicate messages, and a test device-offline condition.
- Send data through the same ingestion interface planned for MQTT devices.
- Validate coordinates, timestamp, device identity, sequence/duplicate behavior, and payload size.
- Persist latest location separately from history.

## Acceptance criteria

- Simulator can run without physical hardware.
- Invalid telemetry is rejected and logged safely.
- Latest truck state updates correctly.
- Duplicate and out-of-order messages have defined behavior.
- Dashboard can show moving and stale trucks from simulated data.

---

# PHASE 09 — Realtime updates with Socket.io

## Goal

Push authorized truck and event updates to connected clients.

## Tasks

- Authenticate Socket.io connections.
- Derive room membership from server-side user scope; never accept arbitrary room names from clients as authorization.
- Emit normalized location/event payloads only to eligible rooms/clients.
- Handle reconnect and stale state by refetching authoritative data.
- Add tests proving cross-city/society clients cannot subscribe to restricted information.

## Acceptance criteria

- Authorized clients receive updates.
- Unauthorized room joins are rejected.
- Disconnect/reconnect does not leak or duplicate access.
- Realtime messages contain only fields required by the client.

---

# PHASE 10 — Geofencing, approach events, and alerts

## Goal

Turn location data into useful operational events.

## Tasks

- Implement spatial queries with PostGIS.
- Support configurable approach radius (initial target: 500 m) where authorized.
- Define entry/exit and event state transitions.
- Add event deduplication and notification cooldown rules.
- Add stale-truck, missed-stop, route-deviation, dwell-time, and power-loss events only when their triggering data and business rules are specified.
- Record event timestamps, source, relevant entity IDs, and acknowledgement status as appropriate.
- Ensure proximity is not automatically treated as proof of collection completion.

## Acceptance criteria

- Repeated GPS updates do not create repeated identical alerts.
- Events are sent only to authorized roles/societies.
- Boundary and radius behavior is tested.
- Missing/stale telemetry is distinguished from a known stationary truck.

---

# PHASE 11 — Physical MQTT and ESP32 integration

Start only after the simulator-to-dashboard path is reliable.

## Tasks

- Finalize broker host, TLS settings, topic format, ACLs, and device credential provisioning.
- Implement Node.js MQTT subscription and reconnect/error handling.
- Validate each device's topic and map device ID to an assigned truck server-side.
- Normalize MQTT messages to the same schema used by the simulator.
- Implement firmware reading GPS and any approved sensor inputs.
- Define power-cut behavior and test safe voltage sensing; never connect vehicle voltage directly to ESP32 GPIO.
- Document offline buffering, sequence numbers, time source, retry behavior, and credential rotation.

## Acceptance criteria

- Valid device messages enter the same processing pipeline as simulator messages.
- Unknown devices and unauthorized topics are rejected.
- Broker credentials are not committed.
- Device/network interruption produces a visible stale/offline status rather than fabricated location.

---

# PHASE 12 — Push notifications

## Goal

Send relevant mobile alerts through Firebase Cloud Messaging after the event and recipient authorization logic is stable.

## Tasks

- Define device-token registration and revocation.
- Store notification preferences and eligible society/user relationships as required.
- Send FCM messages server-side only.
- Handle invalid tokens, retry policy, and notification deduplication.
- Avoid putting sensitive location details in push payloads unless explicitly required and authorized.

## Acceptance criteria

- Only eligible recipients are targeted.
- FCM server credentials remain private.
- Failures are logged and do not break telemetry ingestion.

---

# PHASE 13 — Mobile applications

## Goal

Build citizen and driver experiences after API and authorization contracts stabilize.

## Citizen app tasks

- Expo foundation and authentication.
- Authorized society onboarding/verification.
- Relevant truck approach/status view.
- Push notifications and preferences.
- Restricted map/radar view; no unrestricted city-wide fleet feed.

## Driver app tasks

- Assigned route/truck view.
- Stop checklist and current/next stop.
- Navigation integration only after requirements and map-provider implications are reviewed.
- Dwell-time workflow and connectivity states.

## Acceptance criteria

- Mobile apps reuse backend business logic and APIs.
- Users cannot access other societies/routes by modifying client state.
- Offline and permission states are explicit.

---

# PHASE 14 — Reports, audit, and operational accountability

## Goal

Provide traceable operations and useful reports.

## Tasks

- Audit sensitive user, role, scope, route, truck, and alert actions.
- Build authorized CSV exports and basic operational reports.
- Record alert acknowledgement and relevant history.
- Define retention and access rules for telemetry.
- Add pagination and limits to large result sets.

## Acceptance criteria

- Reports respect the caller's scope.
- Exports cannot be used to bypass authorization.
- Audit records are not editable by ordinary users.
- Telemetry retention is documented and tested.

---

# PHASE 15 — Public website

## Goal

Create the public GeoSweep product website separately from the protected municipal dashboard.

## Pages/sections

- Product overview and problem statement.
- How the system works.
- Feature overview.
- Security/privacy explanation that matches implemented controls.
- Pilot-request/contact form.
- Accessibility, metadata, and basic SEO.

## Rules

- Do not display real citizen data, private locations, credentials, or internal dashboards.
- Do not claim unimplemented features are live; label roadmap items honestly.
- Keep marketing content consistent with the actual product.

## Acceptance criteria

- Responsive page builds and deploys.
- Form validates input and has success/error states.
- No secret keys are included in client code.

---

# PHASE 16 — Production hardening and deployment

## Goal

Prepare the tested system for a controlled pilot.

## Tasks

- Configure environment-specific settings and restrictive CORS.
- Add structured logging, health checks, and useful operational metrics.
- Review rate limiting, request validation, token expiry, and dependency vulnerabilities.
- Review database backups, migrations, indexes, connection limits, and retention.
- Test failure/recovery for backend, database, broker, realtime, and push services.
- Review secrets rotation and deployment access.
- Verify provider free-tier restrictions and estimated resource use.

## Acceptance criteria

- Deployment uses environment variables and no committed secrets.
- Recovery and backup strategy is documented.
- Authorization regression tests pass.
- Free-tier quotas and production limitations are understood.

---

# PHASE 17 — Controlled real-world pilot

## Goal

Validate a small deployment before scaling.

## Pilot outline

- Select one ward and a small number of trucks, with municipal authorization.
- Test simulated and physical telemetry paths.
- Verify route/stop configuration and location accuracy.
- Measure message delivery, stale-location detection, alert accuracy, and duplicate rates.
- Train assigned staff and collect feedback.
- Document issues and remediation before expansion.

Do not claim a pilot is complete until real measurements and stakeholder acceptance exist.

---

# C. API and database rules

## API rules

- Use a consistent `/api/...` prefix.
- Inspect existing routes before creating new ones.
- Validate request parameters, body, and query strings with a consistent schema-validation approach.
- Use DTOs/request schemas; do not return unrestricted database rows by default.
- Centralize error handling and return safe messages without stack traces or secrets.
- Paginate potentially large collections.
- Enforce authorization at service/data-query boundaries.
- Document API changes.

## Database rules

- Use migrations under `supabase/migrations/` or the repository's established migration location.
- Never casually edit a migration already applied to a shared database; create a new migration.
- Use foreign keys, unique constraints, check constraints, and indexes where appropriate.
- Use PostGIS spatial types and spatial indexes for geographic queries.
- Avoid indefinitely storing high-frequency telemetry without a retention plan.
- Keep latest-position data separate from history when it improves access patterns.
- Keep seed data fake and reproducible.

## Realtime rules

- Authenticate the connection.
- Determine permissions and rooms on the server.
- Validate subscription requests.
- Send the minimum data necessary.
- Re-check scope when assignments or permissions change; do not rely forever on stale room membership.

## MQTT rules

- Use TLS and unique credentials per device where supported.
- Configure broker ACLs for least privilege.
- Validate payloads and device-to-truck assignment.
- Handle reconnect, duplicates, delayed messages, malformed data, and unknown devices.
- Never treat topic text alone as proof of identity.

---

# D. Free-tier policy

GeoSweep should use free-tier services during development wherever feasible. Before choosing or changing a provider:

1. Check current official free-tier limits and usage terms.
2. Identify sleeping/pausing behavior, storage, egress, connection, message, and build limits.
3. Explain limitations before depending on the service for a pilot.
4. Ask before introducing paid infrastructure.
5. Keep domain logic portable so providers can be upgraded later.

Do not guarantee that free tiers are sufficient for production. GPS updates every few seconds can create substantial write volume; establish sampling and retention before scale testing.

---

# E. Phase completion checklist

A phase is complete only when all applicable items are true:

- [ ] Acceptance criteria are met.
- [ ] Architecture follows this document.
- [ ] Database migration exists for schema changes.
- [ ] Input validation exists.
- [ ] Server-side authorization exists for protected operations.
- [ ] API contracts and docs are updated.
- [ ] Frontend uses real APIs unless mocks are explicitly required.
- [ ] Loading, empty, error, and success states are handled where relevant.
- [ ] Unit/integration tests are added where useful.
- [ ] Relevant build/type-check/test commands were run and reported honestly.
- [ ] Existing features still work.
- [ ] No secrets or private data were committed.
- [ ] README/PHASES/docs are updated if decisions changed.

## Immediate next step

Start with **Phase 00**, inspect the actual repository, and produce a short plan before making edits. The first major product milestone is Super Admin city onboarding + City Admin scope enforcement. Hardware, live MQTT, mobile apps, and production push notifications come later.
