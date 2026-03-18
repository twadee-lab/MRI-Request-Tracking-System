# MRI Request & Tracking System — Developer Prompt

> **Purpose:** This prompt contains the complete system specification for building the MRI Request & Tracking System. Use it with AI coding assistants (Claude Code, Cursor, GitHub Copilot, etc.) to generate the full application.

---

## PROJECT OVERVIEW

Build a web-based **MRI Request & Tracking System** for a hospital. The system allows hospital staff to submit MRI result requests and track their status. MRI department staff manage the request workflow.

### Key Capabilities
- Submit MRI result requests via Google Form
- Track request status via unique Tracking Link (no login required)
- MRI Staff manage request queue and status transitions
- Email notifications on status changes
- Operational dashboard for hospital management

### Users
- **Requester** — Hospital department staff who submit and track requests
- **MRI Staff** — MRI department staff who process requests (authenticated)
- **Hospital Management** — View dashboard reports (authenticated)

---

## TECH STACK

| Layer | Technology |
|---|---|
| Frontend | React 18+, TypeScript 5+, Vite 5+, Tailwind CSS 3+, React Router 6+, Axios, Recharts |
| Backend | Node.js 20 LTS, Express.js 4.18+, TypeScript 5+, Prisma 5+ ORM |
| Database | PostgreSQL 15+ |
| Auth | express-session + bcrypt (session-based) |
| Validation | Zod 3+ |
| Email | Nodemailer 6+ (via SendGrid SMTP) |
| Queue | BullMQ + Redis (async notification queue) |
| Logging | Winston 3+ |

---

## DATABASE SCHEMA

### Enum Types

```sql
CREATE TYPE request_status AS ENUM ('Submitted','Review','On Hold','Processing','Ready','Complete','Rejected','Cancelled');
CREATE TYPE request_priority AS ENUM ('Red','Yellow','Green');
CREATE TYPE status_action AS ENUM ('Start Review','Put On Hold','Resume Review','Start Processing','Mark Ready','Complete Request','Reject Request','Reopen Request','Cancel Request');
CREATE TYPE actor_role AS ENUM ('Requester','MRI Staff');
CREATE TYPE notification_event AS ENUM ('New Request','Submitted','On Hold','Ready','Rejected','Cancelled','Complete');
CREATE TYPE notification_status AS ENUM ('Sent','Failed');
```

### Sequence

```sql
CREATE SEQUENCE request_id_seq START WITH 1 INCREMENT BY 1 CACHE 1;
```

### Tables

```sql
CREATE TABLE request (
    request_id          VARCHAR(8) PRIMARY KEY,          -- REQ-XXXX format
    patient_hn          VARCHAR(20) NOT NULL,            -- HN-YYnnnnn format
    requester_email     VARCHAR(255) NOT NULL,
    requester_department VARCHAR(100) NOT NULL,
    mri_reason          VARCHAR(500) NOT NULL,
    priority            request_priority NOT NULL,
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status              request_status NOT NULL DEFAULT 'Submitted',
    version             INTEGER NOT NULL DEFAULT 1,       -- optimistic locking
    duplicate_flag      BOOLEAN NOT NULL DEFAULT FALSE,
    duplicate_ref_id    VARCHAR(8) NULL REFERENCES request(request_id),
    tracking_token      VARCHAR(64) NOT NULL UNIQUE,
    CONSTRAINT chk_version CHECK (version >= 1),
    CONSTRAINT chk_duplicate CHECK (
        (duplicate_flag = FALSE AND duplicate_ref_id IS NULL) OR
        (duplicate_flag = TRUE AND duplicate_ref_id IS NOT NULL)
    )
);

CREATE TABLE status_history (
    log_id       VARCHAR(36) PRIMARY KEY DEFAULT gen_random_uuid()::TEXT,
    request_id   VARCHAR(8) NOT NULL REFERENCES request(request_id),
    from_status  request_status NULL,        -- NULL for initial Submitted entry
    to_status    request_status NOT NULL,
    action       status_action NOT NULL,
    performed_by VARCHAR(255) NOT NULL,       -- organizational email
    role         actor_role NOT NULL,
    reason       VARCHAR(500) NULL,           -- required for On Hold, Reject, Cancel
    performed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE notification_log (
    notification_id VARCHAR(36) PRIMARY KEY DEFAULT gen_random_uuid()::TEXT,
    request_id      VARCHAR(8) NOT NULL REFERENCES request(request_id),
    event           notification_event NOT NULL,
    recipient_email VARCHAR(255) NOT NULL,
    status          notification_status NOT NULL DEFAULT 'Sent',
    retry_count     INTEGER NOT NULL DEFAULT 0 CHECK (retry_count BETWEEN 0 AND 3),
    sent_at         TIMESTAMPTZ NULL,
    failed_at       TIMESTAMPTZ NULL,
    error_message   VARCHAR(1000) NULL
);

CREATE TABLE mri_staff (
    staff_id      VARCHAR(36) PRIMARY KEY DEFAULT gen_random_uuid()::TEXT,
    email         VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    display_name  VARCHAR(100) NULL,
    is_active     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login_at TIMESTAMPTZ NULL
);
```

### Indexes

```sql
CREATE INDEX idx_request_status_priority ON request (status, priority, submitted_at);
CREATE INDEX idx_request_patient_hn_status ON request (patient_hn, status);
CREATE UNIQUE INDEX idx_request_tracking_token ON request (tracking_token);
CREATE INDEX idx_request_duplicate_ref ON request (duplicate_ref_id) WHERE duplicate_ref_id IS NOT NULL;
CREATE INDEX idx_status_history_request ON status_history (request_id, performed_at);
CREATE INDEX idx_notification_request ON notification_log (request_id);
CREATE INDEX idx_notification_retry ON notification_log (status, retry_count) WHERE status = 'Failed' AND retry_count < 3;
```

### Helper Functions

```sql
CREATE OR REPLACE FUNCTION generate_request_id() RETURNS VARCHAR(8) AS $$
BEGIN RETURN 'REQ-' || LPAD(nextval('request_id_seq')::TEXT, 4, '0'); END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION generate_tracking_token() RETURNS VARCHAR(64) AS $$
BEGIN RETURN encode(gen_random_bytes(16), 'hex'); END;
$$ LANGUAGE plpgsql;
```

---

## API ENDPOINTS

### Authentication (F09)

| Method | Endpoint | Body | Response |
|---|---|---|---|
| POST | /api/auth/login | {email, password} | 200: {user, token} · 401: generic error |
| POST | /api/auth/logout | — | 200: OK |
| GET | /api/auth/me | — | 200: {email, role} · 401: unauthorized |

**Rules:** Generic error on failed login — never reveal if email exists. Session associated with org email. Email recorded as performed_by in all audit logs.

### Request Submission (F01, F02, F07)

| Method | Endpoint | Body | Response |
|---|---|---|---|
| POST | /api/requests/check-duplicate | {patient_hn} | 200: {has_active, active_request?} |
| POST | /api/requests | {patient_hn, requester_email, requester_department, mri_reason, priority, duplicate_flag?, duplicate_ref_id?} | 201: {request_id, tracking_token, tracking_url} · 400: validation · 500: error |

**Side effects on create:**
1. Generate Request ID via DB sequence (atomic)
2. Generate random tracking_token
3. Set version = 1
4. INSERT status_history (null → Submitted)
5. Enqueue email T01 (New Request → MRI Dept) and T02 (Submitted → Requester)

**CRITICAL:** Request creation is atomic. If any step fails, rollback everything.

### Queue & Request Detail (F03, F04)

| Method | Endpoint | Params/Body | Response |
|---|---|---|---|
| GET | /api/queue | ?status=active(default)/all/completed/rejected/cancelled | 200: sorted request list |
| GET | /api/queue/stats | — | 200: count per active status |
| GET | /api/requests/:request_id | — | 200: full request + history + available_actions |
| GET | /api/requests/:request_id/history | — | 200: audit log entries |
| POST | /api/requests/:request_id/transition | {action, version, reason?} | 200: updated · 400: reason missing · 403: flag blocks · 409: version conflict |

**Queue sorting:** Red priority first → Yellow → Green. Within same priority: oldest submitted_at first.

**Default queue filter:** Only active statuses (Submitted, Review, On Hold, Processing, Ready).

### Status Transitions — Valid Actions

| action | From | To | Reason | Notes |
|---|---|---|---|---|
| start_review | Submitted | Review | No | |
| put_on_hold | Review | On Hold | YES | |
| resume_review | On Hold | Review | No | NO notification sent |
| start_processing | Review | Processing | No | BLOCKED if duplicate_flag=true → 403 |
| mark_ready | Processing | Ready | No | |
| complete | Ready | Complete | No | |
| reject | Review | Rejected | YES | |
| reopen | Rejected | Review | No | Retains original Request ID |
| cancel | Submitted/Review/Processing | Cancelled | YES | Requester: only from Submitted/Review. Staff: also from Processing |

**Server-side validation order:**
1. Auth check (401)
2. Request exists (404)
3. Action valid for current status (400)
4. Duplicate flag check for start_processing (403)
5. Version match — optimistic locking (409)
6. Reason provided if required (400)
7. Re-check duplicate flag at COMMIT time, not page load time
8. Commit: UPDATE status + increment version + INSERT status_history
9. Enqueue notification (async, non-blocking)
10. Auto-clear duplicate flags if closing a request referenced by others

### Tracking (F05, F10) — PUBLIC, no auth

| Method | Endpoint | Body | Response |
|---|---|---|---|
| GET | /api/track/:token | — | 200: {request_id, status, priority, submitted_at, version, can_cancel, reason?, history[]} · 404: generic error |
| POST | /api/track/:token/cancel | {reason, version} | 200: cancelled · 400: reason missing · 403: not cancellable · 404: invalid token · 409: conflict |

**CRITICAL security rules:**
- NEVER return patient_hn, requester_email, requester_department in public tracking response
- Filter out status_history entries where to_status = 'Complete' (Complete hidden from requester)
- 404 response must be generic — never confirm if request exists
- can_cancel = true only when status IN ('Submitted', 'Review')

### Dashboard (F08) — Authenticated

| Method | Endpoint | Response |
|---|---|---|
| GET | /api/dashboard/stats | 200: {total, completed, rejected, cancelled, avg_processing_time_hours, by_status, by_priority, volume_over_time[]} |

---

## EMAIL NOTIFICATIONS (F06)

7 templates. All emails include: Request ID, Current Status, Request Summary, Tracking Link.

| Template | Trigger | Recipient | Includes Reason |
|---|---|---|---|
| T01 New Request | Request created | MRI Department | No |
| T02 Submitted | Request submitted | Requester | No |
| T03 On Hold | Staff puts on hold | Requester | YES |
| T04 Ready | Staff marks ready | Requester | No |
| T05 Rejected | Staff rejects | Requester | YES |
| T06 Cancelled | Either party cancels | Requester | YES |
| T07 Complete | Staff completes | Requester | No |

**Rules:**
- Resume Review sends NO notification
- If reason is required but empty, do NOT send
- Email delivery failure is NON-BLOCKING — status transition already committed
- Retry policy: 3 attempts at 1min, 3min, 5min intervals via BullMQ

Subject format: `[MRI Request System] {Status} — {Request ID}`

---

## STATUS WORKFLOW

Main flow: `Submitted → Review → Processing → Ready → Complete`

Alternative flows:
- `Review → On Hold → Review` (resume)
- `Review → Rejected → Review` (reopen by MRI Staff)
- `Submitted/Review → Cancelled` (by Requester)
- `Submitted/Review/Processing → Cancelled` (by MRI Staff)

---

## DUPLICATE DETECTION (F07)

**At submission:** Check if active request exists for same patient_hn. Active = Submitted, Review, On Hold, Processing, Ready. Show warning but do NOT block (MVP).

**After submission (if proceeded):** Set duplicate_flag=true, duplicate_ref_id=active request ID.

**Flag rules:**
- Visible to MRI Staff only (warning icon in queue + banner on detail page)
- Start Processing is BLOCKED while flag is present
- Start Review, Put On Hold, Reject, Cancel are ALLOWED
- Flag auto-cleared when referenced active request transitions to Complete/Rejected/Cancelled

**Auto-clear query:**
```sql
UPDATE request SET duplicate_flag=FALSE, duplicate_ref_id=NULL
WHERE duplicate_ref_id='REQ-XXXX' AND duplicate_flag=TRUE;
```

---

## OPTIMISTIC LOCKING

Every request has a `version` field (starts at 1, increments on each transition).

**How it works:**
1. Frontend reads version from GET response
2. Frontend sends version with every POST /transition
3. Backend: `UPDATE request SET status=X, version=version+1 WHERE request_id=Y AND version=Z`
4. If 0 rows affected → 409 Conflict: "This request has already been updated. Please refresh."
5. Frontend shows toast notification (SCR-09) with Refresh button

---

## FRONTEND SCREENS

### SCR-01: Login Page (/login)
- Email + password form
- Generic error on failure (no data exposure)
- Redirect to /queue on success

### SCR-02: Google Form (external)
- Fields: Email (auto-filled), Patient HN, Department, Reason (500 char), Priority (Red/Yellow/Green radio)
- All fields required
- On submit: check duplicate → if found show SCR-02a modal → create request

### SCR-02a: Duplicate Warning Modal
- Shows active request info (ID, status, requester)
- Buttons: "Cancel Submission" / "Proceed Anyway"

### SCR-03: MRI Work Queue (/queue)
- Auth required → redirect to login if not
- Top nav: logo, user email, logout, tabs (Queue / Dashboard)
- Summary stat cards: total active, count per status
- Filter tabs: Active (default), All, Completed, Rejected, Cancelled
- Data table: Priority dot, Request ID (link), Patient HN, Department, Status badge, Submitted At, Duplicate flag icon
- Click row → SCR-04

### SCR-04: Request Detail (/queue/:request_id)
- Auth required
- Back link to queue
- Header: Request ID (large), status badge, priority badge
- Duplicate flag banner (if flagged): shows active request info
- Request info card: Patient HN, Email, Department, Reason, Submitted At
- Action buttons: dynamic based on current status (see Action Button Matrix)
- Status timeline: chronological audit log

**Action Button Matrix:**

| Status | Buttons |
|---|---|
| Submitted | Start Review, Cancel |
| Review | Start Processing, Put On Hold, Reject, Cancel |
| Review + Flag | Start Processing (DISABLED), Put On Hold, Reject, Cancel |
| On Hold | Resume Review |
| Processing | Mark Ready, Cancel (staff only) |
| Ready | Complete |
| Rejected | Reopen |
| Complete/Cancelled | (none) |

### SCR-04a: Reason Modal
- Dynamic title: "Put On Hold" / "Reject Request" / "Cancel Request"
- Textarea (required, max 500 chars, character count)
- Confirm button disabled until text entered
- Color matches action type (amber/red/gray)

### SCR-05: Tracking Page (/track?token=xxx)
- NO auth required — public via token
- Header: system title
- Request ID, status badge (large), priority tag
- Status stepper: Submitted → Review → Processing → Ready (Complete NEVER shown)
- Reason card: shown for On Hold (amber), Rejected (red), Cancelled (gray)
- Status timeline (Complete entries hidden)
- Cancel button: visible ONLY for Submitted/Review
- Mobile responsive (NFR-U02)

### SCR-06: 404 Page (/track?token=invalid)
- Generic "Page Not Found" — NO request data exposed
- Contact MRI Department link

### SCR-07: Dashboard (/dashboard)
- Auth required
- KPI cards: Total, Completed, Rejected, Cancelled, Avg Processing Time
- Charts: Requests by Status (bar), by Priority (donut), Volume over Time (line)

### SCR-08: Email Templates
- 7 variants (T01-T07) with base template
- Header bar (blue), greeting, notification message, request summary card, reason block (conditional), CTA button "View Request Status", footer

### SCR-09: Version Conflict Toast
- Amber bar at top of SCR-04
- Message: "This request has already been updated. Please refresh."
- Refresh button + close button
- Auto-dismiss after 10 seconds

---

## DESIGN SYSTEM

### Colors
| Role | Hex |
|---|---|
| Primary Blue | #1565C0 |
| Success Green | #2E7D32 |
| Warning Amber | #F57F17 |
| Danger Red | #C62828 |
| Processing Purple | #6A1B9A |
| Background | #F5F7FA |
| Text Primary | #212121 |
| Text Secondary | #757575 |

### Priority Colors
| Priority | Dot | Badge BG | Badge Text |
|---|---|---|---|
| Red (Urgent) | #D32F2F | #FFEBEE | #B71C1C |
| Yellow (Normal) | #FBC02D | #FFF8E1 | #F57F17 |
| Green (Low) | #388E3C | #E8F5E9 | #1B5E20 |

### Status Badge Colors
| Status | BG | Text |
|---|---|---|
| Submitted | #E0E0E0 | #424242 |
| Review | #BBDEFB | #0D47A1 |
| On Hold | #FFF3E0 | #E65100 |
| Processing | #E1BEE7 | #4A148C |
| Ready | #C8E6C9 | #1B5E20 |
| Complete | #A5D6A7 | #1B5E20 |
| Rejected | #FFCDD2 | #B71C1C |
| Cancelled | #CFD8DC | #37474F |

---

## PROJECT STRUCTURE

```
mri-request-system/
├── frontend/
│   └── src/
│       ├── components/    (StatusBadge, PriorityDot, ReasonModal, StatusStepper, Timeline)
│       ├── pages/         (LoginPage, WorkQueue, RequestDetail, TrackingPage, NotFoundPage, Dashboard)
│       ├── api/           (API client functions)
│       ├── types/         (shared TypeScript types)
│       ├── hooks/
│       ├── utils/
│       └── App.tsx        (router)
├── backend/
│   └── src/
│       ├── routes/        (auth, requests, queue, track, dashboard)
│       ├── services/      (RequestService, StatusService, DuplicateService, NotificationService, TrackingService, AuthService, ReportService)
│       ├── middleware/     (auth, errorHandler)
│       ├── validators/    (Zod schemas)
│       ├── jobs/          (emailWorker — BullMQ consumer)
│       └── app.ts
│   └── prisma/
│       ├── schema.prisma
│       └── migrations/
├── shared/types/
├── google-apps-script/Code.gs
├── docker-compose.yml
└── .env.example
```

---

## ENVIRONMENT VARIABLES

```env
DATABASE_URL=postgresql://mri_user:password@localhost:5432/mri_system
PORT=3001
NODE_ENV=development
FRONTEND_URL=http://localhost:5173
SESSION_SECRET=change-in-production
SESSION_MAX_AGE=86400000
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
SMTP_USER=apikey
SMTP_PASSWORD=your-sendgrid-key
EMAIL_FROM=mri-system@hospital.org
MRI_DEPARTMENT_EMAIL=mri.department@hospital.org
REDIS_URL=redis://localhost:6379
TRACKING_BASE_URL=https://mri-system.hospital.org/track
GOOGLE_WEBHOOK_SECRET=your-webhook-secret
```

---

## DOCKER COMPOSE (dev)

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: mri_user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mri_system
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
volumes:
  pgdata:
```

---

## NON-FUNCTIONAL REQUIREMENTS

| ID | Requirement | Implementation |
|---|---|---|
| NFR-P01 | Tracking Page load < 3 seconds | Unique index on tracking_token |
| NFR-P02 | Request ID assigned < 5 seconds | DB sequence atomic increment |
| NFR-P03 | Status updates reflected < 5 seconds | Direct DB update, no cache |
| NFR-P04 | Email delivered < 2 minutes | BullMQ immediate processing |
| NFR-A01 | 24/7 availability | Cloud hosting + managed DB |
| NFR-S01 | Org email only for submission | Validate at Form + API level |
| NFR-S02 | Tracking via non-guessable link | 32-char random hex token |
| NFR-S03 | Staff actions require auth | Express middleware on /api/queue, /api/requests |
| NFR-S04 | No patient data in public URLs | Token-based URL, exclude patient_hn from public API response |
| NFR-D01 | 3-year data retention | No hard deletes, Cancelled status only |
| NFR-U01 | Web browser only | React SPA, no plugins |
| NFR-U02 | Tracking Page mobile-friendly | Responsive Tailwind CSS |

---

## CRITICAL IMPLEMENTATION NOTES

1. **Request creation is ATOMIC** — generate ID + insert request + insert history + add to queue must all succeed or all rollback
2. **Email is ASYNC and NON-BLOCKING** — failed email NEVER rolls back status transition
3. **Optimistic locking** — every transition sends version, server checks before commit
4. **Duplicate flag re-check at COMMIT time** — not at page load time
5. **Complete status HIDDEN from Tracking Page** — filter from history, show Ready as last visible
6. **Public tracking endpoint EXCLUDES** patient_hn, requester_email, requester_department
7. **Generic 404** for invalid tracking tokens — never confirm if request exists
8. **Generic login error** — never reveal if email exists
9. **Reason required** for: Put On Hold, Reject, Cancel — block action if empty
10. **Resume Review** is the ONLY transition that sends NO notification
