# Implementation Plan: Clinic Modernization Platform (CMP)

## Overview
The Clinic Modernization Platform (CMP) is a secure, cloud-hosted digital operations platform designed to transition a chain of private clinics from manual paper/WhatsApp workflows to a secure, digital operations model. Key goals include reducing receptionist manual scheduling time by 70%, reducing patient no-show rates by 25–30%, eliminating scheduling conflicts, and ensuring NDPR-compliant patient clinical records. The system utilizes a decoupled React PWA frontend (Vite) and a FastAPI backend (Python 3.12+) backed by PostgreSQL, Redis/Celery queue, and AWS KMS for envelope encryption.

---

## Architectural Decisions & Tech Stack
* **Frontend:** Vite + React + TypeScript + Tailwind CSS (responsive design, mobile/tablet/desktop friendly).
* **Backend:** FastAPI (Python 3.12+) async REST API.
* **Database:** PostgreSQL 16+ with SQLModel/SQLAlchemy ORM.
* **Task Queue:** Celery + Redis for asynchronous notifications and OTP verification tasks.
* **Security & KMS:** Application-level column encryption using AES-256-GCM via the Python `cryptography` library and AWS KMS (envelope encryption). Immutable audit logs are created in the same database transactions.
* **Notifications:** Pluggable Strategy Pattern supporting WhatsApp Business API, Termii SMS, and Infobip SMS with an automatic failover worker.

---

## Dependency Graph

```
[Task 1: Backend Scaffolding & DB Config] ──┐
[Task 3: Frontend PWA Scaffolding]         │
                                           ▼
                       [Task 2: Database Schema & Migration]
                                           │
                                           ▼
                       [Task 4: OTP Verification Engine]
                                           │
                                           ▼
                       [Task 5: User & Session Auth Endpoints]
                                           │
                                           ▼
                       [Task 6: Frontend Auth & Registration UI]
                                           │
                                           ▼
                       [Task 7: Scheduling Engine & Bookings API]
                                           │
                    ┌──────────────────────┴──────────────────────┐
                    ▼                                             ▼
       [Task 8: Rescheduling Engine]                 [Task 10: Notification Abstraction]
                    │                                             │
                    ▼                                             ▼
       [Task 9: Patient Portal Booking UI]           [Task 11: Failover Notifications]
                    │                                             │
                    └──────────────────────┬──────────────────────┘
                                           ▼
                       [Task 12: Envelope Encryption Helper]
                                           │
                                           ▼
                       [Task 13: Clinical Records API & Audit Logs]
                                           │
                                           ▼
                       [Task 14: Doctor Clinical Records UI]
                                           │
                    ┌──────────────────────┴──────────────────────┐
                    ▼                                             ▼
       [Task 15: Receptionist check-in UI]           [Task 16: Management Reports UI]
                    │                                             │
                    └──────────────────────┬──────────────────────┘
                                           ▼
                       [Task 17: Service Worker Offline Cache]
                                           │
                                           ▼
                       [Task 18: Performance Tuning & Prod Build]
```

---

## Task List

### Phase 1: Scaffolding & Database Setup (Foundations)

#### Task 1: Backend Scaffolding & DB Configuration
* **Description:** Bootstrap the FastAPI backend directory, setup configuration management (Pydantic settings), configure PostgreSQL database engine, and initialize Alembic for database migrations.
* **Acceptance criteria:**
  - [ ] FastAPI boilerplate running with health check endpoint (`/health`).
  - [ ] SQLAlchemy/SQLModel connection pool initialized with configurable environment variables.
  - [ ] Alembic initialized and connected to development database.
* **Verification:**
  - [ ] Checkpoint: Server runs locally and responds with `{"status": "ok"}` on `/health`.
  - [ ] Command run: `alembic current` successfully connects to DB and returns current revision status.
* **Dependencies:** None
* **Files likely touched:**
  - `backend/app/main.py`
  - `backend/app/core/config.py`
  - `backend/app/db/session.py`
  - `backend/alembic.ini`
  - `backend/requirements.txt`
* **Estimated scope:** Medium (3-5 files)

#### Task 2: Database Schema & Migration Script
* **Description:** Define SQLModel models and database tables including `users` (Auth), `patient_profiles` (Confidential), `doctor_availability` (Shifts), `appointments`, `clinical_records` (Encrypted), `verification_otps` (OTP), and `security_audit_logs` (Immutable logs) along with SQL Enums. Generate the initial Alembic migration.
* **Acceptance criteria:**
  - [ ] SQLModel models match specifications in `data-models.md` precisely, including enums for user roles, appointment status, and payment states.
  - [ ] Alembic auto-generates migrations including indexes for fields like `phone_number`, `email`, and availability start/end times.
  - [ ] Check constraints added (e.g. `start_datetime < end_datetime`).
* **Verification:**
  - [ ] Command run: `alembic upgrade head` executes without errors.
  - [ ] PostgreSQL inspection confirms tables, columns, foreign keys, and indexes are created.
* **Dependencies:** Task 1
* **Files likely touched:**
  - `backend/app/models/enums.py`
  - `backend/app/models/all_models.py`
  - `backend/alembic/versions/xxxx_initial.py`
* **Estimated scope:** Medium (3-5 files)

#### Task 3: Frontend PWA Scaffolding
* **Description:** Initialize Vite + React PWA client application with TypeScript, Tailwind CSS, and React Router. Create basic layouts, responsive sidebar/header navigation, and configure root styling variables matching modern design guidelines.
* **Acceptance criteria:**
  - [ ] Vite-based React-TS app successfully bootstrapped and running dev server.
  - [ ] Design system CSS custom properties (color palettes, shadows, animations) configured in `index.css` and Tailwind.
  - [ ] Navigation routing and layouts created for Guest, Patient, and Staff views.
* **Verification:**
  - [ ] Build succeeds: `npm run build` inside frontend.
  - [ ] Visual check: Landing layout loads with clean responsive header.
* **Dependencies:** None
* **Files likely touched:**
  - `frontend/package.json`
  - `frontend/vite.config.ts`
  - `frontend/src/index.css`
  - `frontend/src/App.tsx`
  - `frontend/src/components/Layout.tsx`
* **Estimated scope:** Medium (3-5 files)

---

### Checkpoint: Foundation
* [ ] Database migrations run and upgrade successfully to head.
* [ ] Backend health check responds correctly.
* [ ] Frontend builds cleanly and launches its development server.

---

### Phase 2: Authentication & Verification Engine (Core 1)

#### Task 4: OTP Verification Service (Backend)
* **Description:** Implement OTP generation and validation backend module. Generates 6-digit codes, hashes them for secure storage in `verification_otps`, invalidates past OTPs for the same phone, implements 10-minute TTL expiration, and rate-limits requests using Redis.
* **Acceptance criteria:**
  - [ ] OTPs generated, hashed (bcrypt/sha256), and saved with expiry timestamps.
  - [ ] Limit of max 5 verification attempts before invalidating active session.
  - [ ] Redis-based rate limiting restricts requests to max 3 per phone number per 15 minutes.
* **Verification:**
  - [ ] Tests pass: `pytest tests/test_otp.py` covering creation, validation, attempts exhaustion, rate limiting.
* **Dependencies:** Task 2
* **Files likely touched:**
  - `backend/app/services/otp_service.py`
  - `backend/app/core/redis_client.py`
  - `backend/tests/test_otp.py`
* **Estimated scope:** Medium (3-5 files)

#### Task 5: User & Session Auth Endpoints
* **Description:** Implement authentication endpoints: `/api/v1/auth/verify-request` (triggers OTP send), `/api/v1/auth/verify-code` (validates OTP and returns access token), and `/api/v1/auth/login` (password auth for staff). Define JWT scopes matching user roles.
* **Acceptance criteria:**
  - [ ] JWT tokens issued with specific security scopes mapping to `user_role` enums.
  - [ ] Dependency injection helpers for FastAPI route protection based on scopes (e.g. `SecurityScopes`).
  - [ ] Auth endpoints return correct validation codes (400 on invalid OTP, 401 on bad password, 429 on rate limit).
* **Verification:**
  - [ ] Tests pass: Auth API integration tests verify login & OTP verification endpoint results.
* **Dependencies:** Task 4
* **Files likely touched:**
  - `backend/app/api/v1/endpoints/auth.py`
  - `backend/app/core/security.py`
  - `backend/tests/test_auth_api.py`
* **Estimated scope:** Medium (3-5 files)

#### Task 6: Frontend Auth & Registration UI
* **Description:** Implement Login Page, OTP Verification modal/screen, and Patient Self-Registration form. Integrate auth state management (React Context/Redux) to store tokens and persist patient profile details.
* **Acceptance criteria:**
  - [ ] Interactive login form with responsive layouts supporting both mobile OTP (patients) and password login (staff).
  - [ ] Self-registration form collecting full name, date of birth, gender, and emergency contact.
  - [ ] Auth state persists across reloads via localStorage.
* **Verification:**
  - [ ] Manual check: Run frontend, request OTP, verify navigation shifts to profile setup or dashboard upon successful login.
* **Dependencies:** Task 3, Task 5
* **Files likely touched:**
  - `frontend/src/context/AuthContext.tsx`
  - `frontend/src/pages/Login.tsx`
  - `frontend/src/pages/Register.tsx`
  - `frontend/src/services/authApi.ts`
* **Estimated scope:** Medium (3-5 files)

---

### Checkpoint: Auth Flow
* [ ] End-to-end patient registration and OTP login works.
* [ ] Role-Based Access Control correctly restricts endpoints on the backend based on token scopes.

---

### Phase 3: Scheduling Engine & Patient Portal (Core 2)

#### Task 7: Scheduling Engine & Booking Endpoints
* **Description:** Build scheduling engine with doctor availability shift management. Implement `POST /api/v1/appointments` with PostgreSQL transaction-level pessimistic locking (`SELECT ... FOR UPDATE`) to prevent concurrent double-bookings.
* **Acceptance criteria:**
  - [ ] Staff can CRUD doctor availability shifts in `doctor_availability` table.
  - [ ] Appointment booking validates slots against doctor availability shifts and existing booked appointments.
  - [ ] Pessimistic locks prevent race conditions when two clients attempt to book the exact same slot concurrently.
* **Verification:**
  - [ ] Concurrency test: Run parallel scripts attempting to book the same slot; one succeeds (201) and the other fails with lock conflict (409) within 3 seconds.
  - [ ] Unit tests pass: `pytest tests/test_scheduler.py`
* **Dependencies:** Task 5
* **Files likely touched:**
  - `backend/app/api/v1/endpoints/appointments.py`
  - `backend/app/services/scheduler.py`
  - `backend/tests/test_scheduler.py`
* **Estimated scope:** Medium (3-5 files)

#### Task 8: Rescheduling Engine & Penalty Rules
* **Description:** Implement appointment updates/cancellations (`PATCH` / `DELETE` /api/v1/appointments/{id}). Apply business rules for late cancellations (<2 hours before shift): logging warnings, soft-flagging profile, and blocking self-service bookings.
* **Acceptance criteria:**
  - [ ] Validation blocks cancellations <2 hours before start, unless marked as emergency by staff/admin.
  - [ ] Late cancellations increment penalty metrics on the patient profile.
  - [ ] Automated self-service lock applied on patient booking profile if they reach $\ge 4$ penalties within 90 days.
* **Verification:**
  - [ ] Tests verify cancellation time check, audit increments, penalty tiers enforcement, and booking blocks.
* **Dependencies:** Task 7
* **Files likely touched:**
  - `backend/app/services/penalty_manager.py`
  - `backend/app/api/v1/endpoints/appointments.py`
  - `backend/tests/test_penalties.py`
* **Estimated scope:** Small (1-2 files)

#### Task 9: Patient Portal Booking UI
* **Description:** Build Patient Portal Dashboard. Allow patients to search doctors by availability, select branches, book slots, view upcoming schedules, and cancel appointments with penalty warning prompts. Use Dexie.js for caching.
* **Acceptance criteria:**
  - [ ] Calendar booking view showing available slots dynamically based on backend searches.
  - [ ] Dynamic modal displaying late-cancellation warnings if a cancellation is attempted <2 hours from start time.
  - [ ] IndexedDB caching via Dexie.js initialized to load current appointments offline.
* **Verification:**
  - [ ] Manual check: Book, cancel, and reschedule appointments via Patient Portal interface. Verify browser console local cache outputs.
* **Dependencies:** Task 6, Task 8
* **Files likely touched:**
  - `frontend/src/pages/patient/Dashboard.tsx`
  - `frontend/src/pages/patient/BookAppointment.tsx`
  - `frontend/src/services/dbCache.ts`
* **Estimated scope:** Medium (3-5 files)

---

### Checkpoint: Scheduling & Bookings
* [ ] Concurrency tests successfully prevent double-bookings.
* [ ] Late cancellation engine flags penalties and blocks bookings when threshold is hit.
* [ ] Patients can search, book, and view appointments via portal.

---

### Phase 4: Pluggable Notification Queue (Core 3)

#### Task 10: Notification Abstraction & Celery Worker Setup
* **Description:** Setup Celery task runner backed by Redis inside the FastAPI app. Build the pluggable `NotificationService` wrapper following the Strategy Pattern to define message payloads (confirmations, OTPs, reminders).
* **Acceptance criteria:**
  - [ ] Celery worker worker process starts and listens to queue tasks.
  - [ ] Base `NotificationStrategy` class defines abstract send interfaces.
  - [ ] Integration with `NotificationLog` table to audit outbound latency, channel, status.
* **Verification:**
  - [ ] Server log inspection validates Celery task processing of mock payloads.
* **Dependencies:** Task 2
* **Files likely touched:**
  - `backend/app/core/celery_app.py`
  - `backend/app/services/notifications/base.py`
  - `backend/app/models/all_models.py`
* **Estimated scope:** Medium (3-5 files)

#### Task 11: Notification Providers & Failover Chain
* **Description:** Implement adapters for WhatsApp Business API, Termii SMS, and Infobip SMS. Write the Celery task runner executing failover strategy: send via WhatsApp $\rightarrow$ check response/timeout (15s) $\rightarrow$ fallback to Termii SMS $\rightarrow$ fallback to Infobip SMS on failure.
* **Acceptance criteria:**
  - [ ] Individual adapter classes for WhatsApp, Termii, and Infobip.
  - [ ] Failover logic handles external HTTP exceptions and triggers fallback.
  - [ ] Transactional booking confirmations and OTP verification codes queued through this chain.
* **Verification:**
  - [ ] Integration tests mock API responses (success, slow timeouts, and hard failures) to verify correct fallback paths and final logs.
* **Dependencies:** Task 10
* **Files likely touched:**
  - `backend/app/services/notifications/providers.py`
  - `backend/app/tasks/notification_tasks.py`
  - `backend/tests/test_notifications.py`
* **Estimated scope:** Medium (3-5 files)

---

### Checkpoint: Notification Delivery
* [ ] Celery handles async background delivery.
* [ ] Failover chain operates correctly when WhatsApp or primary SMS returns errors/timeouts.

---

### Phase 5: Encrypted Clinical Records (Core 4)

#### Task 12: Application-Level Envelope Encryption Module
* **Description:** Create application memory cryptography layer. Use Python `cryptography` library's AES-256-GCM. Integrate with AWS KMS client (`boto3`) to implement envelope encryption: request Data Encryption Key (DEK), cache DEK temporarily in-memory, encrypt data, and format ciphertext output containing key version, IV, tag, and payload.
* **Acceptance criteria:**
  - [ ] Encrypted data structure safely formatted (e.g. JSON containing `ciphertext`, `iv`, `tag`, `kms_key_version`).
  - [ ] In-memory DEK caching to prevent redundant AWS KMS calls per column write/read.
  - [ ] Test suites mock KMS responses to ensure offline testing capability.
* **Verification:**
  - [ ] Unit tests pass: `pytest tests/test_crypto.py` verifies correct encrypt/decrypt operations, DEK caching, and key rotation compatibility.
* **Dependencies:** Task 5
* **Files likely touched:**
  - `backend/app/core/crypto.py`
  - `backend/tests/test_crypto.py`
* **Estimated scope:** Small (1-2 files)

#### Task 13: Clinical Records Endpoints & Immutable Audit Logging
* **Description:** Create endpoints `POST /api/v1/clinical-records` and `GET /api/v1/clinical-records/{patient_id}`. Enforce strict RBAC scopes permitting ONLY doctor roles to decrypt. Enforce immutable database inserts into `security_audit_logs` inside the same database transaction.
* **Acceptance criteria:**
  - [ ] Endpoint allows saving encrypted notes, diagnoses, and prescriptions.
  - [ ] Audit logs containing user details, action type, IP, and timestamp are inserted inside PostgreSQL transaction blocks.
  - [ ] System administrator and manager scopes explicitly fail to decrypt clinical notes (return HTTP 403 or filtered blank payload).
* **Verification:**
  - [ ] Tests verify role restriction permissions, encrypted storage in DB tables, and audit log creations.
* **Dependencies:** Task 12
* **Files likely touched:**
  - `backend/app/api/v1/endpoints/clinical_records.py`
  - `backend/app/services/audit.py`
  - `backend/tests/test_clinical_api.py`
* **Estimated scope:** Medium (3-5 files)

#### Task 14: Doctor Clinical Records Interface
* **Description:** Build clinic consultation dashboard for doctors. Create patient records search (log audit trails), patient history viewer (decrypt notes dynamically in memory), and encounter logger for entering clinical notes, diagnoses, and prescriptions.
* **Acceptance criteria:**
  - [ ] Doctor interface layout optimized for tablets and laptops.
  - [ ] Encounter logger form with validation that encrypts clinical notes before saving.
  - [ ] Reading historical notes calls backend API and triggers audit log tracking.
* **Verification:**
  - [ ] Manual check: Log in as doctor, search patient, open record (verify audit log populated), save notes, check DB tables to confirm text is stored as encrypted blocks.
* **Dependencies:** Task 9, Task 13
* **Files likely touched:**
  - `frontend/src/pages/doctor/Dashboard.tsx`
  - `frontend/src/pages/doctor/Encounter.tsx`
  - `frontend/src/pages/doctor/PatientHistory.tsx`
* **Estimated scope:** Medium (3-5 files)

---

### Checkpoint: Security & Encryption
* [ ] Database values are fully encrypted; direct SQL query reveals only ciphertext.
* [ ] Decryption is only possible through authenticated Doctor JWT credentials.
* [ ] All record reads generate audit logs.

---

### Phase 6: Front Desk, Reporting & PWA Offline Sync (Polish & Rollout)

#### Task 15: Receptionist & Check-In Portal
* **Description:** Build front desk receptionist dashboard. Allows walk-in patient registrations, scheduling new slots on behalf of patients, check-in arrivals (notifying Doctor queue), and managing clinic queues.
* **Acceptance criteria:**
  - [ ] Search patient UI, quick registration form.
  - [ ] Check-in button toggles appointment status to 'checked-in' and sends real-time updates to doctor panel.
  - [ ] Admin/staff booking overrides with reason inputs logged to audit files.
* **Verification:**
  - [ ] Walk-through verification of receptionist walk-in register, check-in, and doctor calendar state updates.
* **Dependencies:** Task 9, Task 14
* **Files likely touched:**
  - `frontend/src/pages/receptionist/Dashboard.tsx`
  - `frontend/src/pages/receptionist/WalkInRegistration.tsx`
* **Estimated scope:** Medium (3-5 files)

#### Task 16: Management Dashboard & Reporting
* **Description:** Build real-time analytics reports dashboard for Branch Managers and Executive Managers. Aggregate operational metrics: branch utilization, doctor shift schedules, no-show/cancellation percentages.
* **Acceptance criteria:**
  - [ ] Real-time aggregated statistics display with filters for date ranges and branches.
  - [ ] Graphs/charts visualizing no-show rates, active slots, and penalty counts.
  - [ ] Access control limits branch managers to their specific clinic branch data, while senior executives see cross-clinic reports.
* **Verification:**
  - [ ] Verify restricted manager credentials can only access statistics for their designated branch, while executive accounts view aggregated multi-branch stats.
* **Dependencies:** Task 15
* **Files likely touched:**
  - `frontend/src/pages/management/Dashboard.tsx`
  - `frontend/src/components/AnalyticsCharts.tsx`
* **Estimated scope:** Medium (3-5 files)

#### Task 17: Service Worker Offline Support
* **Description:** Fully configure Workbox Service Worker caching behavior. Store current-day schedules, branch/doctor lists, and patient profiles locally in IndexedDB (using Dexie.js). Prevent booking modifications while offline and display clear indicator.
* **Acceptance criteria:**
  - [ ] Service worker caches web assets and API responses for offline operations.
  - [ ] Schedule dashboards load successfully when network is disconnected (read-only for $\ge 2$ hours).
  - [ ] Write operations (booking, check-in) disabled with active user warning when offline.
* **Verification:**
  - [ ] Simulate network disconnect in browser DevTools: dashboard loads cached schedules, network warning banner displays, and booking submit button is disabled.
* **Dependencies:** Task 9, Task 15
* **Files likely touched:**
  - `frontend/src/sw.ts`
  - `frontend/src/components/OfflineBanner.tsx`
* **Estimated scope:** Small (1-2 files)

#### Task 18: Performance Tuning & Production Build
* **Description:** Add Database indexing, request logging with request correlation IDs, profiling performance to satisfy NFR-001 (search <2s under load) and NFR-002 (PWA load <3s on 3G/4G). Build production assets.
* **Acceptance criteria:**
  - [ ] Correlation IDs appended to structured JSON logs.
  - [ ] Database queries indexed on search terms (e.g. `patient_profiles.full_name`, `appointments.start_datetime`).
  - [ ] Production frontend bundles built, minified, and optimized.
* **Verification:**
  - [ ] Lighthouse audit scores for Performance and PWA exceed 90.
  - [ ] Network load tests under simulated 100 concurrent requests confirm endpoint response times remain <2.0s.
* **Dependencies:** Task 16, Task 17
* **Files likely touched:**
  - `backend/app/core/logging.py`
  - `backend/app/db/indexing.py`
  - `frontend/vite.config.ts`
* **Estimated scope:** Small (1-2 files)

---

### Checkpoint: Complete
* [ ] Lighthouse audit confirms sub-3s load on simulated mobile connections.
* [ ] Production frontend and backend builds compile cleanly.
* [ ] Full end-to-end integration tests verify auth, concurrency locks, encryption, and audit logs.

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **AWS KMS Downtime** | High | Implement local cache for Data Encryption Keys (DEKs) and fail gracefully (temporary lock on new encrypted write/read transactions, displaying clear error message rather than crash). |
| **Race Conditions in Booking** | High | Enforce database-level pessimistic locking (`SELECT ... FOR UPDATE`) in PostgreSQL. Fail fast and return a clean HTTP 409 Conflict. |
| **Notification Gateway Blockages** | Medium | Maintain an automated Strategy Pattern queue with fallback chain: WhatsApp $\rightarrow$ Termii (Nigerian DND bypass) $\rightarrow$ Infobip SMS failover. |
| **Data Privacy (NDPR) Breaches** | High | Cryptographically isolate clinical notes. Decryption only in memory, scoped to Doctor credentials. Block administrators from decryption keys. |

---

## Open Questions
- Do we have sandbox credentials available for WhatsApp Cloud API, Termii, and Infobip to test notification delivery, or should we initialize mocks for development?
- Will the AWS KMS key be managed in the same AWS account as the database, or will it be in a cross-account setup?
- What is the expected initial volume of doctors and patients per clinic to optimize search indices (e.g., pg_trgm indices)?
