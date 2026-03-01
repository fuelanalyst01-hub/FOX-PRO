# FoxPRO Suite — Master Technical Specification (Production-Ready)

## 1) Product Vision & Scope
FoxPRO Suite is a secure, mobile-first Progressive Web Application (PWA) for:
- Daily fuel operations management by supervisors.
- Attendance management by employees using short video + GPS verification.
- Centralized administration, auditing, monitoring, and reporting by administrators.

### 1.1 Primary Outcomes
- Improve fuel data accuracy and operational accountability.
- Reduce attendance fraud with video + geolocation validation.
- Provide actionable dashboards and exports for management.
- Ensure reliable operation across desktop/tablet/mobile with installable PWA support.

### 1.2 Out of Scope (Phase 1)
- Native Android/iOS apps (web PWA only, with future API readiness).
- Third-party biometric hardware integration.

---

## 2) Technology Stack & Standards
- **Backend:** Django 6.0, Python 3.10
- **Frontend:** Django templates + HTML5 + Bootstrap 5 + modern CSS tokens/utilities
- **Database:** SQLite (phase-1 default; schema and ORM kept PostgreSQL-compatible)
- **Media:** Local media storage (videos/images) with secure serving rules
- **Time:** UTC in DB, Asia/Karachi in UI/reports
- **PWA:** Manifest + Service Worker (offline shell + queued sync where possible)
- **API:** REST API layer (versioned: `/api/v1/`)
- **Task Scheduling:** Cron/Celery beat equivalent for cleanup, backup, report jobs
- **Logging/Monitoring:** Structured logs + health dashboard

### 2.1 Non-Functional Targets
- **Availability target:** 99.5% monthly (single-host baseline)
- **Response targets:** p95 page/API response < 800ms under normal load
- **Security baseline:** OWASP-aligned controls for auth, CSRF, sessions, file handling
- **Mobile UX:** usable and responsive at 360px width and above

---

## 3) Roles & Permission Model (RBAC)

## 3.1 Roles
1. **Administrator**
   - Full system access: users, roles, sites, offices, fuel, attendance, reports, backups, audit, settings.
2. **Supervisor**
   - Can submit/update fuel entries only for assigned sites.
   - Can view own-assignment dashboards and reports.
3. **Employee**
   - Can create attendance check-in/check-out with video + GPS.
   - Can view own attendance history.

### 3.2 Permission Enforcement
- Route-level guards (middleware/decorators).
- Queryset-level filtering (never rely only on UI hiding).
- Action-level checks in serializers/forms/services.
- Denied access attempts logged to security audit.

---

## 4) Core Functional Modules

## 4.1 Authentication & Access Control
- Secure login/logout, password reset/change.
- Mandatory password change on first login.
- Session timeout with configurable idle limit.
- Rate limiting for login attempts (IP + username key).
- Optional remember-me with shorter privileged session windows.

## 4.2 Fuel Entry Module
- Supervisors create daily fuel entries for assigned sites.
- One entry per site/date/shift (configurable uniqueness).
- Required fields include opening, received, consumed/sold, closing stock.
- Auto-validation formula checks:
  - `closing_expected = opening + received - consumed`
  - deviation beyond threshold requires reason note + flags record.
- Edit window policy (e.g., same day editable; late edit requires admin approval).
- Soft-delete support with restoration and full audit trail.

## 4.3 Attendance Module (Video + GPS)
- Employee attendance actions:
  - Check-in (start of duty)
  - Check-out (end of duty)
- Video capture required (short clip; e.g., 3–10 seconds, configurable).
- GPS coordinates required, with accuracy meters captured.
- Auto-close prior open attendance record for same office/site when new check-in occurs.
- Validation and anti-fraud controls:
  - Allowed file formats (mp4/webm), max file size, max duration.
  - Geofence validation (if site geofence configured).
  - Duplicate punch prevention within cooldown window.
  - IP logging + user-agent fingerprint snapshot.

## 4.4 Dashboards & Analytics
- Role-aware dashboards:
  - Admin: system-wide fuel trends, attendance compliance, anomalies, health widgets.
  - Supervisor: assigned-site fuel KPIs, entry completion rate.
  - Employee: attendance streak/history summary.
- Time filters: today, week, month, custom range.
- Visuals: cards + trend charts + exception tables.

## 4.5 Reporting & Exports
- Dynamic Excel exports for fuel and attendance.
- Monthly attendance report by employee/office/site.
- Attendance matrix (employee vs date with status codes).
- Supervisor performance report (submission timeliness, correction count).
- Export access controlled per role and assignment.

## 4.6 Soft Delete & Recovery
- Soft delete for business records (fuel, attendance, users where applicable).
- Recycle-bin style restore for admins.
- Hard delete restricted + logged (for legal/compliance exceptions only).

## 4.7 Audit & Activity Timeline
- Central immutable audit log for:
  - Create/update/delete/restore actions
  - Auth events (login success/failure, password changes)
  - Permission denials and admin-sensitive actions
- Activity timeline UI for administrators with advanced filters.

## 4.8 Health Monitoring Dashboard
- Database connectivity status.
- Storage usage (media/static/db).
- Last backup status/time.
- Task scheduler heartbeat.
- Error rate summary and recent critical exceptions.

## 4.9 Backup & Restore
- Scheduled automated backups (DB + media metadata references).
- Manual backup trigger for admins.
- Restore workflow with safety confirmation + pre-restore snapshot.
- Backup retention policy (e.g., daily 14, weekly 8, monthly 6).

## 4.10 REST API Layer (Future Mobile Readiness)
- Versioned endpoints under `/api/v1/`.
- Token/session auth strategy defined.
- OpenAPI schema generation.
- Permission parity with web app policies.

---

## 5) Data Model (Complete Baseline)

> Added to close missing definitions from initial prompt.

## 5.1 Master Tables
- `roles` (Admin, Supervisor, Employee)
- `users` (extends auth user; employee/supervisor profile fields)
- `offices`
- `sites` (belongs to office)
- `user_site_assignments` (supervisor-to-site mapping, effective dates)
- `employee_office_assignments` (employee-to-office/site mapping)

## 5.2 Fuel Domain
- `fuel_entries`
  - `site_id`, `entry_date`, `shift`, `opening_qty`, `received_qty`, `consumed_qty`, `closing_qty`
  - `closing_expected`, `variance_qty`, `variance_reason`
  - `status` (draft/submitted/approved/rejected)
  - `created_by`, `approved_by`, timestamps, soft-delete fields
- Unique constraint: `(site_id, entry_date, shift, is_deleted=false)`

## 5.3 Attendance Domain
- `attendance_records`
  - `employee_id`, `office_id`, `site_id`
  - `check_in_time_utc`, `check_out_time_utc`
  - `check_in_video_path`, `check_out_video_path`
  - `check_in_lat`, `check_in_lng`, `check_in_accuracy_m`
  - `check_out_lat`, `check_out_lng`, `check_out_accuracy_m`
  - `check_in_ip`, `check_out_ip`, `device_info`
  - `status` (open/closed/auto_closed/flagged)
  - `auto_closed_reason`, soft-delete fields
- Duplicate prevention index example:
  - no multiple open records per employee-office pair.

## 5.4 Security/Audit/Ops Tables
- `audit_logs` (actor, action, entity, before/after, ip, ua, timestamp)
- `auth_events` (login success/fail, lockout, password reset/change)
- `system_health_snapshots`
- `backup_jobs` and `restore_jobs`
- `failed_jobs` (for background task failures)

## 5.5 Reference/Configuration Tables
- `app_settings` (global config: limits, windows, thresholds)
- `geofences` (site polygon/radius)
- `holiday_calendar` (optional for attendance reports)
- `shift_definitions`

---

## 6) Validation Rules & Business Logic

## 6.1 Fuel Rules
- Reject negative quantities.
- Enforce numeric precision and unit consistency.
- Flag abnormal variance over configured threshold.
- Prevent duplicate entries by unique constraints + user-friendly form errors.

## 6.2 Attendance Rules
- Check-in required before check-out.
- Video must pass type/size/duration validation.
- GPS must be present; low-accuracy submissions can be flagged.
- Auto-close previous open records for same office/site policy.
- Cooldown to prevent rapid repeated punches.

## 6.3 Cross-Cutting Rules
- All writes create audit entries.
- All timestamps stored in UTC.
- Soft-deleted records excluded by default managers/querysets.

---

## 7) Security Specification
- CSRF protection on all state-changing web requests.
- Secure/HttpOnly session cookies; SameSite policy set.
- Password policy (length, complexity, reuse limits configurable).
- Brute-force controls (rate limits + temporary lockout).
- Strict file upload validation and media path sanitization.
- Direct media URL protection for private attendance videos.
- Principle of least privilege enforced at all layers.

### 7.1 Data Privacy & Compliance Enhancements (Added)
- PII classification for employee data and retention rules.
- Video retention and purge policy (e.g., configurable 90/180 days).
- Legal-hold flag capability for audit/legal investigations.

---

## 8) Frontend / UX Specification (Modern & Responsive)

## 8.1 Design System
- Bootstrap 5 + custom design tokens:
  - Color palette, spacing scale, typography scale, elevation, border radius.
- Consistent components: navbar, sidebar, cards, tables, forms, toasts, modals.
- Dark mode ready (optional toggle in phase 2).

## 8.2 Responsive Behavior
- Mobile-first layout (360px+).
- Breakpoints optimized for phone/tablet/desktop.
- Touch-friendly controls and large tap targets.
- Sticky action bars on mobile forms where useful.

## 8.3 Form UX & Accessibility
- Real-time field validation with clear error states.
- Loading/disabled states for submit actions.
- WCAG-minded contrast, labels, keyboard navigation basics.
- Multi-language readiness hooks (i18n-ready labels/messages).

## 8.4 PWA UX
- Install prompt support.
- Offline shell caching for essential UI assets.
- Graceful offline messaging; queue pending attendance/fuel actions when possible.

---

## 9) API Contract (High-Level)
- Auth endpoints: login/logout/profile/password change.
- Fuel endpoints: list/create/update/approve/export.
- Attendance endpoints: check-in/check-out/list/history.
- Audit/report endpoints (admin restricted).
- Health/backup endpoints (admin restricted).
- Standard response envelope with pagination and error schema.

### 9.1 API Security
- Auth required for all non-public endpoints.
- Rate limiting for sensitive endpoints.
- Request/response logging for critical actions (without leaking secrets).

---

## 10) Background Jobs & Automation
- Daily backup execution and retention pruning.
- Orphaned media cleanup and consistency checks.
- Monthly report materialization/summary caching.
- Health snapshot collector job.
- Alert dispatch job for failed backups or abnormal error spikes.

---

## 11) Deployment & Environment
- Production settings split (`dev/staging/prod`).
- Static/media mapping correctly configured.
- HTTPS enforced in production.
- Rotating logs and error monitoring integration.
- Secret management through environment variables.

### 11.1 Environment Variables (Minimum)
- `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS`
- DB path/URL
- Session/cookie security flags
- Media/static roots
- Backup storage path/credentials
- Rate-limit config values

---

## 12) QA & Testing Strategy (Added)
- Unit tests for validators, services, and permissions.
- Integration tests for fuel and attendance flows.
- API tests for auth and role boundaries.
- UI smoke tests for key responsive screens.
- Security tests: CSRF, upload validation, lockout behavior.
- Data migration tests for constraints/indexes.

### 12.1 Acceptance Criteria Snapshot
- Supervisor can submit valid fuel entry only for assigned site.
- Employee can check-in/check-out only with valid video + GPS.
- Duplicate and out-of-policy submissions are blocked/flagged.
- Admin can audit any critical action and restore soft-deleted records.
- Exports generate correctly and respect access permissions.

---

## 13) Suggested Project Structure (Django)
- `apps/accounts`
- `apps/fuel`
- `apps/attendance`
- `apps/reports`
- `apps/audit`
- `apps/ops` (health, backup, scheduler hooks)
- `apps/api`
- `templates/`, `static/`, `media/`

---

## 14) Roadmap & Future Extensions (Added)
- Face recognition confidence scoring (optional, policy-approved).
- PostgreSQL migration path for scale.
- Notification center (email/SMS/WhatsApp) for alerts.
- Advanced anomaly detection (fuel/attendance fraud patterns).
- SSO integration (OIDC/SAML) for enterprise deployments.

---

## 15) Final Build Directive
This specification is the authoritative implementation blueprint for FoxPRO Suite. 
All modules, validations, RBAC, PWA behavior, reporting, auditability, and operational controls
must be implemented as defined here. Any ambiguity should be resolved in favor of security,
traceability, and mobile usability.
