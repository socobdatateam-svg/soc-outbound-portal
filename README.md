# SOC Outbound Dispatch Portal — Project Plan

**Created:** 2025-10-25
**Owner / Display name:** SOC Outbound Dispatch Monitoring

---

## Executive Summary

A web portal for SOC Backroom staff (OPS Coordinators / PIC) to encode outbound dispatch records using a modern spreadsheet-like Submit Report UI. The SOC Data Team verifies encoded details; upon verification a chatbot posts CSV-formatted messages to Seatalk hub group chats. Roles include Backroom (submitters), SOC Data Team (verifiers/editors), and SOC Supervisors/Managers (FTE). The system emphasizes speed, accuracy, local-draft autosave, clear status tracking, and strict role-based access.

### Objectives & Success Metrics
- **Fast encoding:** average time to submit a single row & batch under target (e.g., < 90s/row). Measured by telemetry on Submit events.
- **Accuracy:** validation failures per 1,000 rows < 5. Tracked by server-side rejection reasons.
- **Verification throughput:** Data Team verifies X batches / day (target configurable).
- **Status transparency:** percentage of dispatches with correct status transitions and seatalk message ID populated after send.
- **Mapping correctness:** outbound map match rate (cluster→hubs) ≥ 99%.

---

## High-Level Feature Breakdown (Top-level)
1. Authentication & Role Management (Backroom Ops login; FTE Gmail OAuth)
2. Submit Report (spreadsheet-like editable grid, autosave drafts, max 10 rows/session)
3. Lookup & Mapping Services (clusters → hubs, docks, processors, OPS lookup)
4. Pre Alert Database + Dashboard (sortable, filterable, groupable table + details)
5. Data Verification Flow (checkbox verify, per-row results, CSV batch generation)
6. LM Hub Email List CRUD (Data Team / FTE only)
7. Seatalk Integration (send CSVs / messages via chatbot — deferred / queued)
8. Audit & Metadata (created_by, updated_by, verified_by, timestamps)
9. Admin Tools (allowlist management, user reset, system logs)
10. Client-side polish (keyboard nav, tooltips, inline validation, keyboard shortcuts)

---

## Suggested Tech Stack (per request)
- Frontend: **Next.js (React)** + Bootstrap for UI components. Consider using a spreadsheet grid library (eg. AG Grid free version or react-data-grid) for better UX.
- Backend: **Node.js (Express or Next.js API routes)** or NestJS for structured APIs.
- Database: **MySQL** (or MariaDB) with knex / TypeORM / Prisma for migrations and models.
- Auth: Gmail OAuth 2.0 (FTE) + local ops_id/password with bcrypt hashed passwords.
- Storage: generated CSVs saved to internal file storage or Google Drive (signed link) depending on enterprise preference.
- Queue: lightweight job queue (BullMQ/Redis) for seatalk send & CSV generation (deferred tasks).
- Deploy: Vercel for Next.js frontend + API or separate Node service; MySQL hosted on managed DB.

---

## Implementation Phases (overview)
- **Phase 0 — Preparation & Onboarding**
  - Finalize data sources (Google Sheet mapping), sample data, and allowlist.
  - Provision dev DB and dev environment. Create repo skeleton.
  - Create initial ERD and API contract.

- **Phase 1 — Core Authentication, Lookups & Submit Report UI (MVP)**
  - Implement auth (Backroom ops login + FTE OAuth stub / allowlist).
  - Implement lookup endpoints (user ops lookup, clusters, hubs, processors).
  - Implement DB schema for `dispatches`, `users`, `hubs`, `clusters`, `hub_emails`.
  - Implement frontend Login page, Dashboard shell, Submit Report UI (editable grid), autosave drafts to localStorage and POST /api/submit-rows (server-side validation).
  - Unit & integration tests for auth, lookups, submit flow.

- **Phase 2 — Verification Flow & Prealert Database**
  - Verification endpoints and UI for Data Team.
  - CSV generation service and per-row result responses.
  - Dashboard detailed views and filters.
  - LM Hub Email List CRUD pages and endpoints.

- **Phase 3 — Seatalk Integration & Queueing**
  - Implement job queue for CSV generation and seatalk send (send-seatalk endpoint triggers worker job)
  - Messaging id capture and status updates.
  - Retry policies and alerting.

- **Phase 4 — Hardening & Admin Tools**
  - Admin panel, allowlist management, encryption at rest, audit logs, rate limiting.
  - E2E tests, security penetration testing, load testing.

- **Phase 5 — Polish & Ops**
  - Monitoring, runbooks, documentation, onboarding guides, training session deliverables.

---

## Phase 1 — Detailed Plan (MUST be completed & tested before Phase 2)

### Goals (Phase 1)
1. Working authentication for Backroom and FTE (FTE allowlist stubbed if necessary).
2. Lookup endpoints returning mapping from Google Sheet (imported into DB or stored as JSON initially).
3. Submit Report UI (editable grid) with client validations, autosave, hide/unhide columns, multi-hub auto-split behavior.
4. Server endpoint `/api/submit-rows` with definitive validation and persisting to `dispatches` table.
5. Local draft autosave, export draft, and restore.
6. Tests: unit tests for core validation, integration tests for POST `/api/submit-rows`.

### Phase 1 Deliverables
- DB migrations for `users`, `clusters`, `hubs`, `dispatches`, `hub_emails`.
- Backend API: auth, lookups, submit-rows.
- Frontend pages: Login, Dashboard shell, Submit Report UI with editable table.
- Test suite and test run report.
- `.env.example` and README with setup steps.

### Phase 1 Timeline & Milestones (suggested)
- Day 0 — Repo skeleton, env, DB setup
- Day 1 — Auth endpoints & user model
- Day 2 — Lookup endpoints & import mapping data
- Day 3 — Submit Report frontend UI (grid + autosave)
- Day 4 — `/api/submit-rows` backend validation & persistence
- Day 5 — Testing, bugfix, and demo to stakeholders

> Note: adjust timeline to actual team capacity. The plan here assumes a small dedicated team or 1–2 engineers.

---

## Phase 1 — Technical Requirements (per component)

### Database schema (DDL sketch)

```sql
-- users
CREATE TABLE users (
  ops_id VARCHAR(32) PRIMARY KEY,
  name VARCHAR(128) NOT NULL,
  email VARCHAR(256),
  role ENUM('BACKROOM','DATA','FTE','ADMIN') NOT NULL,
  password_hash VARCHAR(256),
  must_change_password BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- clusters
CREATE TABLE clusters (
  id INT AUTO_INCREMENT PRIMARY KEY,
  cluster_name VARCHAR(128) UNIQUE,
  region VARCHAR(64),
  hubs_json JSON, -- array of hub objects {hub_id, hub_name, dock_numbers:[]}
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- hubs
CREATE TABLE hubs (
  id INT AUTO_INCREMENT PRIMARY KEY,
  hub_name VARCHAR(128) UNIQUE,
  region VARCHAR(64),
  default_dock VARCHAR(32),
  contact_email VARCHAR(256),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- dispatches (Prealerts / rows)
CREATE TABLE dispatches (
  dispatch_id CHAR(36) PRIMARY KEY,
  created_by_ops_id VARCHAR(32),
  created_by_name VARCHAR(128),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  timestamp_client TIMESTAMP NULL,
  batch_label VARCHAR(32),
  batch_sequence INT,
  region VARCHAR(64),
  cluster_name VARCHAR(128),
  station_name VARCHAR(128),
  count_of_to INT DEFAULT 0,
  total_oid_loaded INT DEFAULT 0,
  dock_number VARCHAR(64),
  dock_confirmed BOOLEAN DEFAULT FALSE,
  actual_docked_time DATETIME,
  actual_depart_time DATETIME,
  processor_name VARCHAR(128),
  lh_trip VARCHAR(64),
  plate_number VARCHAR(64),
  fleet_size VARCHAR(16),
  assigned_ops_id VARCHAR(32),
  assigned_ops_name VARCHAR(128),
  verified_flag BOOLEAN DEFAULT FALSE,
  verified_by VARCHAR(32),
  verified_at DATETIME,
  status ENUM('Pending','Ongoing','Done') DEFAULT 'Pending',
  seatalk_message_id VARCHAR(256),
  csv_file_link TEXT,
  notes TEXT,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- hub_emails (LM Hub email list)
CREATE TABLE hub_emails (
  id INT AUTO_INCREMENT PRIMARY KEY,
  hub_id INT,
  email VARCHAR(256),
  name VARCHAR(128),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Backend API Contract (selected endpoints for Phase 1)

#### 1) GET `/api/lookups/user?ops_id={ops_id}`
Response 200:
```json
{ "ops_id": "OPS123", "name": "Juan Dela Cruz", "role": "BACKROOM", "must_change_password": true }
```
404 when not found.

#### 2) POST `/api/auth/login` (Backroom)
Request:
```json
{ "ops_id": "OPS123", "password": "SOC5-Outbound" }
```
Response 200:
```json
{ "token": "jwt...", "ops_id": "OPS123", "must_change_password": true }
```

#### 3) POST `/api/auth/change-password`
Request:
```json
{ "ops_id": "OPS123", "old_password": "...", "new_password": "..." }
```

#### 4) GET `/api/lookups/clusters?q={q}&region={region}`
Returns matching clusters with mapped hubs.

#### 5) GET `/api/lookups/hubs?cluster={cluster_name}`
Return list of hubs for the cluster.

#### 6) POST `/api/submit-rows`
Request body: `{"rows": [ {row model} ], "ops_id":"OPS123

