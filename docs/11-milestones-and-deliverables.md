# Milestones and deliverables

Purpose: This document gives the 6-week ClinicQ work plan, effort budget, demo rhythm, final checklist, and demo script.

## Effort budget

Must work stays near 120 hours. The full six-week project remains about 180 hours. Removed future-client/public-deployment effort and grading now assess Python/SQL/API regressions, fixture quality, and reproducible local operations, with no interface or cloud bonus task.

| Feature area | Priority | Hours | Included FR IDs |
|---|---|---:|---|
| Guided learning and setup | Must | 30 | FR-OPS-01, FR-CLIENT-01 |
| Auth, tokens, lockout, RBAC | Must | 18 | FR-AUTH-01, FR-AUTH-02, FR-AUTH-03, FR-AUTH-04 |
| Doctor profiles, schedules, leaves | Must | 14 | FR-DOC-01, FR-DOC-02 |
| Availability and search | Must | 14 | FR-AVAIL-01, FR-SEARCH-01 |
| Booking rules and concurrency | Must | 16 | FR-BOOK-01, FR-BOOK-02 |
| Appointment state workflows | Must | 10 | FR-APPT-01, FR-APPT-02, FR-APPT-03, FR-APPT-04 |
| Audit, reports, logs, API standards | Must | 10 | FR-AUDIT-01, FR-REPORT-01, FR-API-01 |
| Must test hardening, local acceptance, and API demonstration | Must | 8 | FR-CLIENT-01, FR-OPS-01, all Must FRs |
| Should notifications and extras | Should | 28 | FR-NOTIF-01, FR-NOTIF-02, FR-DOC-03, FR-PAT-01, FR-CLIN-01 |
| Performance, contract tests, documentation | Should/Could | 18 | FR-PERF-01, FR-TEST-01, FR-REPORT-02, FR-API-02 |
| Interview prep and contingency | Project | 14 | Viva, README, ADR review |
| **Total** |  | **180** |  |

## Week-by-week plan

| Week | Goals | Tasks with IDs | Deliverables | Friday demo checkpoint |
|---|---|---|---|---|
| 1 | Setup and first vertical slice | Learn FastAPI/uv/Docker/PostgreSQL; add CI; begin FR-OPS-01's canonical start/stop, migration, idempotent seed, health, and fixed ports; start FR-CLIENT-01's Python/client health requests. | Clean local start/stop and persisted seed IDs; first migration; skeleton CI and API smoke. | Show exact health JSON, seeded doctors, loopback bindings, and normal stop preserving data. |
| 2 | Authentication and staff access | Implement FR-AUTH-01, FR-AUTH-02, FR-AUTH-03, FR-AUTH-04; add hashing, 15-minute access/7-day refresh, lockout; extend FR-CLIENT-01's selected API demonstration. | Auth/RBAC tests and local collection or Python request assertions. | Show isolated registration, seeded login, lockout, refresh rotation, and patient denial on an admin endpoint. |
| 3 | Clinic structure, availability, and reports | Implement FR-DOC-01, FR-DOC-02, FR-AVAIL-01, FR-SEARCH-01, FR-REPORT-01; add schedule conflicts/query plans and FR-CLIENT-01's report/availability requests. | Doctor endpoints, availability/search/report tests, SQL plan, updated Python/client demo. | Show schedule conflict `409`, IST slots, revenue rank, and selected API smoke. |
| 4 | Booking, appointment, and local workflows | Implement FR-BOOK-01, FR-BOOK-02, FR-APPT-01, FR-APPT-02, FR-APPT-03, FR-APPT-04, FR-AUDIT-01, FR-CLIENT-01; complete FR-OPS-01 and all Must local cases. | Concurrency/state/audit tests; Python HTTP demo or local collection; 50+ meaningful cases; TC-LOCAL-001 through TC-LOCAL-006 evidence. | Show 1 success/9 conflicts, state/audit history, exact offline availability, and persisted stop/start. |
| 5 | Should features and fixes | Add local FR-NOTIF-01/02 or FR-API-02 if time allows; close review, security, and coverage gaps. | Local Mailpit/outbox evidence if selected, expanded error regressions, stable CI, updated README. | Show selected backend Should work and tests. |
| 6 | Hardening, documentation, viva | Close coverage gaps. Run Locust for FR-PERF-01 if selected. Prepare README, ADRs, demo video, and viva answers. | Final tag, coverage report, EXPLAIN ANALYZE notes, 5-minute video. | Full 10-minute final rehearsal and viva practice. |

## Gantt chart

```mermaid
gantt
  title ClinicQ six-week plan
  dateFormat  YYYY-MM-DD
  section Setup
  Environment and CI           :a1, 2026-10-05, 5d
  section Must features
  Authentication and RBAC       :a2, 2026-10-12, 5d
  Doctors schedules availability:a3, 2026-10-19, 5d
  Booking audit API demo        :a4, 2026-10-26, 5d
  API demonstration increments  :a5, 2026-10-05, 20d
  Offline and persistence tests :a9, 2026-10-05, 20d
  Reports in week 3             :a8, 2026-10-22, 2d
  section Hardening
  Should features and performance:a6, 2026-11-02, 5d
  Final docs demo viva          :a7, 2026-11-09, 5d
```

## Final deliverables checklist

- Public GitHub repository named `clinicq-api`.
- Trainer `@sukurcf` added as collaborator.
- README with clean-clone setup in 10 steps or fewer.
- Architecture diagram and at least 5 ADRs.
- Alembic migrations and seed command for 5 doctors and 20 patients.
- Docker Compose lite profile with API and PostgreSQL; Mailpit, Redis, and Celery are optional LOCAL services only.
- Local Bruno/Postman collection or Python HTTP demonstration covering auth, search, availability, booking, state changes, conflict, report, and health.
- Recorded fixtures with SHA-256; demo dates anchored to first seed's actual date and retained on restart. Fixed business clocks are used only in automated tests; manual bookings use current future slots.
- TC-LOCAL-001 through TC-LOCAL-006 results for clean clone, exact offline availability, persisted appointment/audit/fee/token state, errors, loopback, and confirmed reset.
- Test suite with at least 50 meaningful test cases.
- Coverage report proving total 85% line and 75% branch, service 95% line and 85% branch, auth 90% line and 80% branch, and report 85% line and 75% branch.
- Concurrency evidence for same-slot, patient-limit, and cancel-then-book cases.
- EXPLAIN ANALYZE notes for availability and doctor performance report.
- CI green on `main`.
- No secrets in history.
- Final tag `v1.0.0`.
- Demo video of maximum 5 minutes.

## Submission process

1. Freeze Must scope after week 4 unless a blocking bug remains.
2. Create a final PR named `release: prepare ClinicQ demo`.
3. Attach coverage, CI, API/local JUnit, sanitized persistence/hash, and query-plan evidence to the PR.
4. Merge only when CI is green.
5. Tag `v1.0.0` on `main`.
6. Submit repository URL, final tag, demo video, and local collection/Python demonstration plus local acceptance evidence paths.
7. Keep the repository public until evaluation is complete.

## Ten-minute final demo script

| Minute | What to show | Evidence |
|---:|---|---|
| 0 | State the problem: Pune clinic has double bookings and paper queues. | One-sentence overview. |
| 1 | Run canonical local start and show both exact health responses, seed counts, offline/persistence acceptance evidence. | FR-OPS-01, BR-19, BR-31, TC-LOCAL-001 through TC-LOCAL-003. |
| 2 | Show registration test on an isolated fixture, then log in seeded Asha; access token expiry is 900 seconds. | FR-AUTH-01 and FR-AUTH-02. |
| 3 | Show refresh rotation and a denied patient call to admin endpoint. | FR-AUTH-03 and FR-AUTH-04. |
| 4 | Search Dr. Rohan and request a free future slot within 30 days. | FR-SEARCH-01 and FR-AVAIL-01. |
| 5 | Book the selected future slot and show appointment status `BOOKED`. | FR-BOOK-01. |
| 6 | Run same-slot conflict request or test output: 1 success and 9 conflicts. | FR-BOOK-02 and NFR-REL-01. |
| 7 | Check in appointment and mark completed as doctor. | FR-APPT-01 and FR-APPT-04. |
| 8 | Show schedule-change conflict with affected appointment IDs. | FR-DOC-02 and BR-08. |
| 9 | Show doctor performance report, audit row, CI, and coverage report. | FR-REPORT-01, FR-AUDIT-01, and quality gates. |

If time remains, show Mailpit reminder or Locust report as optional Should evidence.

[Back to README](../README.md)
