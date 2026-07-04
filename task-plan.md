# CMP Phase 1 MVP — Task Plan (Vertical Slice)
**Based on**: cmp/implementation-plan.md | **Generated**: 2026-06-04

---

## Execution Strategy
**Vertical Slice**: DB → Backend → Frontend → Tests per feature.
**Dependencies**: Strictly ordered; parallel where noted.

---

## Task 1.1: Foundation — Alembic Migrations Setup
**Description:** Configure Alembic for async PostgreSQL 16+, base model with UUID/timestamps, env.py for RDS.
**Acceptance:**
- [ ] Alembic configured with async dialect
- [ ] Base model: id (UUID), created_at, updated_at
- [ ] alembic upgrade head succeeds on empty RDS; alembic downgrade base works
**Dependencies:** None
**Files:** cmp/alembic/env.py, cmp/src/backend/models/base.py
**Scope:** XS

---

## Task 1.2: Auth & RBAC — User Schema
**Description:** Create users, patient_profiles, verification_otps tables + enums via Alembic.
**Acceptance:**
- [ ] user_role enum: patient/receptionist/doctor/manager/admin/executive
- [ ] users table with phone/email unique, password_hash, role
- [ ] Indexes on phone_number, email
**Dependencies:** Task 1.1
**Files:** cmp/alembic/versions/0002_auth_schema.py, cmp/src/backend/models/user.py
**Scope:** Small

---

## Task 2.1: Backend Foundation — Project Setup
**Description:** Initialize FastAPI, async SQLAlchemy, pydantic-settings config, CORS, correlation_id middleware.
**Acceptance:**
- [ ] FastAPI app at main.py with async lifespan
- [ ] Config loaded from env (RDS, Redis, KMS)
- [ ] Health check at /health returns 200
**Dependencies:** None
**Files:** cmp/src/backend/main.py, cmp/src/backend/core/config.py, cmp/src/backend/db/session.py
**Scope:** XS

---

## Task 2.2: Authentication & RBAC Module
**Description:** Implement JWT auth, OTP flows, patient registration, staff login, RoleChecker dependency.
**Acceptance:**
- [ ] POST /api/v1/auth/register creates patient + JWT
- [ ] POST /api/v1/auth/verify-request enqueues OTP (rate limited 3/15min)
- [ ] POST /api/v1/auth/verify-code validates OTP, issues JWT
- [ ] POST /api/v1/auth/login staff email+password auth
- [ ] RoleChecker enforces roles per endpoint
**Dependencies:** Task 1.2, Task 2.1
**Files:** cmp/src/backend/api/v1/auth/router.py, cmp/src/backend/services/auth_service.py, cmp/tests/test_auth.py
**Scope:** Medium

---

## Task 2.3: NotificationService & Async Workers
**Description:** Strategy Pattern adapters (WhatsApp/Termii/Infobip), failover orchestrator, Celery tasks.
**Acceptance:**
- [ ] Abstract NotificationService + 3 provider adapters
- [ ] Failover: WhatsApp (15s) → Termii → Infobip
- [ ] Celery tasks: send_appointment_confirmation, send_otp
- [ ] notifications_log table migration
**Dependencies:** Task 1.1, Task 2.1
**Files:** cmp/src/backend/services/notification_service.py, cmp/src/backend/workers/tasks.py
**Scope:** Medium

---

## Task 3.1: Scheduling Schema
**Description:** Alembic migration for doctor_availability, appointments tables + enums.
**Acceptance:**
- [ ] appointment_status enum: booked/cancelled/completed/no-show
- [ ] payment_status enum (INT-005 placeholder)
- [ ] Indexes on doctor_id+start_datetime
**Dependencies:** Task 1.1
**Files:** cmp/alembic/versions/0004_scheduling_schema.py
**Scope:** XS

---

## Task 3.2: Booking Engine with Pessimistic Locking
**Description:** Appointment booking with SELECT FOR UPDATE, conflict detection, penalty engine.
**Acceptance:**
- [ ] POST /appointments: lock sequence → insert or HTTP 409
- [ ] DELETE /appointments/{id}: tiered penalty logic (Tier 1/2/3)
- [ ] PATCH /appointments/{id}: reschedule with re-lock
- [ ] Staff override for Tier 3 logs to audit
**Dependencies:** Task 3.1, Task 2.1, Task 2.3
**Files:** cmp/src/backend/api/v1/appointments/router.py, cmp/src/backend/services/scheduling_engine.py
**Scope:** Large

---

## Task 3.3: Clinical Records with KMS Encryption
**Description:** AWS KMS envelope encryption, AES-256-GCM utility, clinical record CRUD.
**Acceptance:**
- [ ] KMS data key generate/decrypt
- [ ] AES-256-GCM encrypt/decrypt with random IV
- [ ] POST /clinical-records: encrypt before DB write
- [ ] GET /clinical-records/{id}: decrypt in memory for doctors only
- [ ] Audit log written in same transaction
**Dependencies:** Task 1.4 (schema), Task 2.1, Task 1.2
**Files:** cmp/src/backend/services/clinical_record_service.py, cmp/src/backend/utils/encryption.py
**Scope:** Large

---

## Task 3.4: Reports & Dashboards
**Description:** Branch/organization aggregation endpoints.
**Acceptance:**
- [ ] GET /reports/branch/daily (manager)
- [ ] GET /reports/organization/summary (executive)
**Dependencies:** Task 3.2, Task 2.3
**Files:** cmp/src/backend/api/v1/reports/router.py
**Scope:** Medium

---

## Task 4.1: Frontend Foundation — PWA & Auth UI
**Description:** Vite+React PWA, Workbox, Dexie.js, Tailwind, Axios, auth pages.
**Acceptance:**
- [ ] npm run dev works on port 5173
- [ ] Service worker registered (Workbox)
- [ ] Dexie schema for offline appointment cache
- [ ] Auth pages: Register, VerifyOTP, Login
**Dependencies:** Task 2.2
**Files:** cmp/src/frontend/src/App.tsx, cmp/src/frontend/src/pages/
**Scope:** Medium

---

## Task 4.2: Patient Portal — Booking Flow
**Description:** Branch/doctor/slot selection, booking, list, cancel/reschedule with penalty UI.
**Acceptance:**
- [ ] New booking: branch → doctor → slot → confirm
- [ ] Appointment list with status badges
- [ ] Cancellation warns for <2h (Tier 1 penalty)
**Dependencies:** Task 4.1, Task 3.2
**Files:** cmp/src/frontend/src/pages/Appointments/
**Scope:** Medium

---

## Task 5.1: Receptionist Dashboard
**Description:** Daily schedule, check-in, walk-in, phone booking, offline mode.
**Acceptance:**
- [ ] Schedule view loads appointments by branch/date
- [ ] "Offline Mode — Read Only" banner on disconnect
- [ ] IndexedDB cache purged on logout
**Dependencies:** Task 4.1, Task 3.2
**Files:** cmp/src/frontend/src/pages/Staff/
**Scope:** Medium

---

## Task 5.2: Doctor Clinical Portal
**Description:** Schedule, clinical notes entry, history, lab results release.
**Acceptance:**
- [ ] Clinical note form encrypts before submit
- [ ] Lab results release toggle (FR-008)
**Dependencies:** Task 5.1, Task 3.3
**Files:** cmp/src/frontend/src/pages/Doctor/
**Scope:** Medium

---

## Task 5.3: Management Dashboard
**Description:** Manager/Executive KPIs, charts, notification metrics.
**Acceptance:**
- [ ] Daily ops metrics load
- [ ] 30-second auto-refresh
**Dependencies:** Task 5.1, Task 3.4
**Files:** cmp/src/frontend/src/pages/Manager/
**Scope:** S

---

## Task 5.4: Admin Console
**Description:** Branch CRUD, user management, availability management, system settings.
**Acceptance:**
- [ ] Create/edit branches
- [ ] Assign roles to staff users
**Dependencies:** Task 5.1, Task 3.2
**Files:** cmp/src/frontend/src/pages/Admin/
**Scope:** S

---

## Task 6.1: Backend Unit Tests
**Description:** pytest suite for auth, scheduling, clinical records, notifications, RBAC.
**Acceptance:**
- [ ] pytest tests pass
- [ ] Coverage > 80%
**Dependencies:** Task 2.2, Task 3.2, Task 3.3, Task 2.3
**Files:** cmp/tests/test_*.py
**Scope:** Medium

---

## Task 6.2: Integration Tests
**Description:** End-to-end API flow tests with test database.
**Acceptance:**
- [ ] Booking flow with concurrent conflict test
- [ ] Clinical encryption round-trip verified
**Dependencies:** Task 6.1, Task 3.2, Task 3.3
**Files:** cmp/tests/integration/
**Scope:** Medium

---

## Task 6.3: E2E Tests
**Description:** Playwright tests for critical user journeys.
**Acceptance:**
- [ ] Patient journey passes
- [ ] Offline mode verified
**Dependencies:** Task 4.1, Task 5.1
**Files:** cmp/src/frontend/tests/e2e/
**Scope:** Medium

---

## Task 6.4: Performance & Security Tests
**Description:** Load tests (NFR-001), Lighthouse (NFR-002), encryption audits.
**Acceptance:**
- [ ] /available-slots < 2.0s at 100 users
- [ ] PWA score >=90 on 3G/4G
**Dependencies:** Task 6.2
**Files:** cmp/tests/load/
**Scope:** S

---

## Task 7.1: AWS Infrastructure (IaC)
**Description:** CloudFormation/Terraform for RDS, Redis, S3, CloudFront, ECS, KMS, IAM.
**Acceptance:**
- [ ] All resources provisioned with least-privilege
- [ ] KMS policy denies access to admin/root
**Dependencies:** None (parallel)
**Files:** cmp/infrastructure/*.yaml
**Scope:** Large

---

## Task 7.2: CI/CD Pipelines (GitHub Actions)
**Description:** CI: lint → test → build; CD: deploy PWA + API + Alembic migrations.
**Acceptance:**
- [ ] CI runs on PR; CD on main merge
**Dependencies:** Task 7.1, Task 6.2
**Files:** cmp/.github/workflows/
**Scope:** Medium

---

## Task 7.3: Staging & Rollout
**Description:** Deploy staging, E2E validation, phased rollout plan.
**Acceptance:**
- [ ] Staging E2E green
- [ ] Rollback procedures documented
**Dependencies:** Task 7.1, Task 7.2, Task 6.3
**Files:** cmp/docs/rollout-plan.md
**Scope:** S

---

## Dependency Graph
```
1.1 → 1.2 → 2.2 ← 4.1 ← 4.2
2.1 → 2.2
2.1 → 2.3 → 3.2 → 3.4 → 5.3
3.1 → 3.2 → 5.1/5.4
3.2 → 3.3 → 5.2
7.1 (parallel) → 7.2 → 7.3
```

## Checkpoints
1. Schema Complete: Tasks 1.1, 1.2, 3.1
2. Auth Functional: Tasks 2.1, 2.2
3. Booking Verified: Tasks 3.1, 3.2
4. Encryption Verified: Task 3.3
5. Notifications: Task 2.3
6. PWA Offline: Tasks 4.1, 5.1
7. Performance: Tasks 6.2, 6.4
8. Deployed: Tasks 7.1, 7.2, 7.3

---

*End of Task Plan*