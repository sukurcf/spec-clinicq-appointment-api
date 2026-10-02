# Testing strategy and test cases

Purpose: This document defines the test approach, coverage gates, concrete test catalog, traceability, and performance conditions for ClinicQ.

## Test strategy

ClinicQ uses Python unit/service tests and real PostgreSQL/API integration tests. The implementation MUST retain at least 50 meaningful cases, and all Must catalog rows need evidence. The existing catalog exceeds that floor; local lifecycle cases below are additional, not substitutes for concurrency, JWT, privacy, or SQL checks.

| Level | Approximate count | Scope | Tools |
|---|---:|---|---|
| Unit | 90 | Password policy, token expiry math, slot generation, state transitions, Problem Details mapping | pytest 9.1.x |
| Service | 70 | Booking rules, login lockout, refresh rotation, patient limits, report calculations | pytest 9.1.x, factory_boy 3.3.x or polyfactory 3.3.x |
| API integration | 80 | FastAPI routes, RBAC, validation, OpenAPI, JSON logs | FastAPI 0.142.x, httpx/TestClient, pytest |
| Database integration | 35 | PostgreSQL constraints, migrations, indexes, raw SQL report, Testcontainers | PostgreSQL 18.x, SQLAlchemy 2.1.x, Alembic 1.20.x, Testcontainers 4.15.x |
| Concurrency | 6 | Same-slot race, patient upcoming limit race, cancelled slot reuse | pytest with thread pool or async client |
| Security | 35 | JWT expiry, tampering, lockout, authorization matrix, no password or token leakage | pytest, PyJWT 2.15.x, pip-audit 2.10.x |
| Performance | 4 | Availability and booking latency targets | Locust 2.46.x |
| Contract and demo | 10 | OpenAPI shape, Bruno/Postman collection, health endpoints, named query-plan checks | Schemathesis 4.29.x if chosen; named checks: OpenAPI document, availability EXPLAIN ANALYZE, report EXPLAIN ANALYZE |
| Local lifecycle | 6 | Clean clone, deterministic offline operation, persisted stop/start, clear failures, loopback, explicit reset | Host-side Python/pytest driver |

## Test tools and versions

| Tool | Reference line | Use |
|---|---|---|
| Python | 3.12.x | Runtime and tests |
| uv | 0.12.x | Dependency install and lockfile |
| FastAPI | 0.142.x | API under test |
| Pydantic | 2.13.x | Request and response validation |
| pydantic-settings | 2.15.x | Test configuration |
| SQLAlchemy | 2.1.x | ORM and SQL execution |
| Alembic | 1.20.x | Migration tests |
| PostgreSQL | 18.x | Constraint and transaction tests |
| psycopg | 3.3.x | PostgreSQL driver |
| Redis | 8.x | Should-scope worker tests |
| Celery | 5.6.x | Should-scope reminder tests |
| Mailpit | 1.31.x | E-mail assertions |
| pytest | 9.1.x | Main test runner |
| pytest-cov / coverage | 7.1.x / 7.16.x | Coverage gates |
| Ruff | 0.16.x | Lint and format gate |
| mypy | 2.4.x | Type gate |
| Testcontainers | 4.15.x | PostgreSQL integration tests |
| Locust | 2.46.x | Performance tests |
| Schemathesis | 4.29.x | Could-scope contract tests |

## Test environments

| Environment | Services | Required use |
|---|---|---|
| Local unit | No Docker services | Unit and service tests on every edit. |
| Local lite integration | API test process and PostgreSQL 18.x | Mandatory CI path for Must features on 8 GB hardware. |
| Local standard integration | API, PostgreSQL, Redis, Celery, Mailpit | Should e-mail tests and full demo rehearsal. |
| CI | PostgreSQL, cached uv/images, isolated local test volumes, coverage/JUnit | Pull requests and pushes to main; offline runtime phase after initial downloads. |
| Performance workstation | Standard 16 GB profile with seeded database | Locust and booking latency evidence. |

## Test data strategy

- Use fictional Indian names and example.in e-mail addresses only.
- Seed command creates exactly 5 doctors, 20 patients, 2 weeks of schedules, and at least 100 appointments for local performance tests.
- Factories create isolated users, doctors, sessions, leaves, and appointments for test cases.
- Freeze time in tests at named instants such as 2026-10-02T10:00:00+05:30.
- Store Mailpit responses only from local Mailpit, not real e-mail services.
- Reset database state per test module using migrations plus transaction rollback or clean schema recreation.
- Recorded local fixture templates and SHA-256 are stored in the student repository; live APIs and real SMTP are not test inputs.
- Seed anchors demo dates at first initialization and preserves them on restart. The exact local acceptance fixture uses `APP_ENV=test` with document 06's restricted business clock; token/login expiry is not globally frozen.
- Pre-cache pinned PostgreSQL and Testcontainers helper images. Offline tests may neither pull images nor silently fall back to SQLite.

## Coverage thresholds

| Scope | Line threshold | Branch threshold | Measured packages | Exclusions |
|---|---:|---:|---|---|
| Total project | 85% | 75% | app | Alembic generated revisions, __main__ blocks, local seed data constants |
| Service layer | 95% | 85% | app/services/booking_service.py, app/services/availability_service.py, app/services/appointment_state_service.py | Framework wiring only |
| Auth and security services | 90% | 80% | app/services/auth_service.py, app/core/security.py | Test-only key fixtures |
| Report service | 85% | 75% | app/services/report_service.py | SQL formatting branches with no behavior |

CI MUST fail when line or branch coverage drops below its separate threshold.

## Test case catalog

| ID | Title | Type | Priority | Linked requirement IDs | Preconditions | Steps | Test data | Expected result |
|---|---|---|---|---|---|---|---|---|
| TC-Q-001 | Register patient account | API positive | Must | FR-AUTH-01 | No user exists for Asha. | Patient sends POST /api/v1/auth/register with asha.kulkarni@example.in, <password>, Asha Kulkarni, 9876543210. | asha.kulkarni@example.in, <password>, Asha Kulkarni, 9876543210. | HTTP 201; response role PATIENT; one users row with hashed password and one linked patients row. |
| TC-Q-002 | Reject duplicate e-mail | API negative | Must | FR-AUTH-01 | Asha already registered. | Patient sends POST /api/v1/auth/register with Same as TC-Q-001. | Same as TC-Q-001. | HTTP 409; Problem Details type email-already-registered; user count for e-mail remains 1. |
| TC-Q-003 | Reject weak password | API negative | Must | FR-AUTH-01 | No user exists for weak password e-mail. | Patient sends POST /api/v1/auth/register with weak@example.in, password short. | weak@example.in, password short. | HTTP 422; Problem Details type password-policy-failed; no user row is created. |
| TC-Q-004 | Successful login returns tokens | API positive | Must | FR-AUTH-02 | Asha exists with password hash. | Patient sends POST /api/v1/auth/login with Asha credentials. | Asha credentials. | HTTP 200; token_type is bearer; expires_in is 900; refresh token row expires in 7 days. |
| TC-Q-005 | Login lockout after five failures | Security negative | Must | FR-AUTH-02 | Asha exists and time is 2026-10-02 10:00 IST. | Patient uses Submit five wrong passwords, then a sixth login before 10:15. Data: Password <wrong_password>. | Password <wrong_password>. | Sixth response HTTP 429; type account-temporarily-locked; lockout_until is 15 minutes after fifth failure. |
| TC-Q-006 | Login succeeds after lockout expires | Security positive | Must | FR-AUTH-02 | Asha locked until 10:30 IST. | Patient uses Freeze time at 10:31 IST and POST valid login. Data: Asha credentials. | Asha credentials. | HTTP 200; failed-attempt counter no longer blocks login; new refresh token is stored. |
| TC-Q-007 | Refresh rotates token | Security positive | Must | FR-AUTH-03 | Login issued refresh token <refresh_token_1>. | Patient uses POST /api/v1/auth/refresh with <refresh_token_1>. Data: <refresh_token_1> before its 7-day expiry. | <refresh_token_1> before its 7-day expiry. | HTTP 200; response has new refresh token <new_refresh_token>; database sets old token rotated_at; new token has same family ID. |
| TC-Q-008 | Reused refresh revokes family | Security negative | Must | FR-AUTH-03 | TC-Q-007 completed. | Patient uses POST /api/v1/auth/refresh with <refresh_token_1> again. Data: Rotated token <refresh_token_1>. | Rotated token <refresh_token_1>. | HTTP 401; type refresh-token-reused; every token in the family has revoked_at set. |
| TC-Q-009 | Revoked refresh rejected | Security negative | Must | FR-AUTH-03 | User logged out with token <refresh_token_3>. | Patient uses POST /api/v1/auth/refresh with <refresh_token_3>. Data: Revoked token <refresh_token_3>. | Revoked token <refresh_token_3>. | HTTP 401; type refresh-token-revoked; no new token row is inserted. |
| TC-Q-010 | Logout revokes presented token | API positive | Must | FR-AUTH-03 | User has valid bearer token and refresh token <refresh_token_4>. | Patient sends POST /api/v1/auth/logout with Body {"refresh_token":"<refresh_token>"}. | Body {"refresh_token":"<refresh_token>"}. | HTTP 204; presented token revoked_at is set; later refresh with <refresh_token_4> returns refresh-token-revoked. |
| TC-Q-011 | Admin creates receptionist | API positive | Must | FR-AUTH-04 | Admin token exists. | Admin sends POST /api/v1/admin/staff-users with Neha receptionist payload. | Neha receptionist payload. | HTTP 201; role RECEPTIONIST; password stored as hash. |
| TC-Q-012 | Patient cannot create staff | Authorization negative | Must | FR-AUTH-04 | Patient token exists. | Patient sends POST /api/v1/admin/staff-users with Neha receptionist payload. | Neha receptionist payload. | HTTP 403; type permission-denied; no staff user is inserted. |
| TC-Q-066 | Admin deactivates staff user | API positive | Must | FR-AUTH-04 | Neha receptionist user is active and admin token exists. | Admin sends POST /api/v1/admin/staff-users/{user_id}/deactivate with Neha user ID and reason Left clinic. | Neha user ID and reason Left clinic. | HTTP 200; response is_active is false; user row is inactive. |
| TC-Q-067 | Non-admin cannot deactivate staff | Authorization negative | Must | FR-AUTH-04 | Neha receptionist user is active and patient token exists. | Patient sends POST /api/v1/admin/staff-users/{user_id}/deactivate with Neha user ID and patient bearer token. | Neha user ID and patient bearer token. | HTTP 403; type permission-denied; user remains active. |
| TC-Q-068 | Deactivated access token rejected | Security negative | Must | FR-AUTH-04 | Neha has an unexpired access token and admin deactivated her account. | Patient uses Use Neha token on GET /api/v1/reception/schedule?date=2026-10-05. Data: Neha bearer token. | Neha bearer token. | HTTP 401; type account-deactivated; schedule body is not returned. |
| TC-Q-013 | Admin creates doctor profile | API positive | Must | FR-DOC-01 | Admin token and General Medicine specialization exist. | Admin sends POST /api/v1/admin/doctors with Dr. Rohan, fee 600.00, slot length 15. | Dr. Rohan, fee 600.00, slot length 15. | HTTP 201; active true; consultation_fee_inr equals 600.00. |
| TC-Q-014 | Negative doctor fee rejected | API negative | Must | FR-DOC-01 | Admin token exists. | Admin sends PATCH /api/v1/admin/doctors/{id} with consultation_fee_inr -10.00. | consultation_fee_inr -10.00. | HTTP 422; response errors include code fee-must-be-non-negative; stored fee unchanged. |
| TC-Q-015 | Inactive doctor hidden from availability | API negative | Must | FR-DOC-01 | Dr. Rohan has no future active appointments and is deactivated. | Patient sends GET /api/v1/doctors/{id}/availability?date=2026-10-05 with Doctor ID for inactive Rohan. | Doctor ID for inactive Rohan. | HTTP 404; type doctor-not-active; no slots are returned. |
| TC-Q-016 | Save two Monday sessions | API positive | Must | FR-DOC-02 | Active Dr. Rohan and admin token exist. | Admin sends PUT /api/v1/admin/doctors/{doctor_id}/weekly-sessions with Monday 09:00-13:00, 17:00-20:00 IST. | Monday 09:00-13:00, 17:00-20:00 IST. | HTTP 200; sessions_saved is 2; two active rows exist for Monday. |
| TC-Q-017 | Reject third weekday session | API negative | Must | FR-DOC-02 | Admin token exists. | Admin uses PUT /api/v1/admin/doctors/{doctor_id}/weekly-sessions with three Tuesday rows. Data: Tuesday sessions 08:00-10:00, 12:00-14:00, 17:00-19:00. | Tuesday sessions 08:00-10:00, 12:00-14:00, 17:00-19:00. | HTTP 422; no Tuesday rows are replaced. |
| TC-Q-018 | Leave conflict returns affected appointment | API negative | Must | FR-DOC-02 | Appointment 77777777-7777-4777-8777-777777777777 is BOOKED at 2026-10-05 10:00 IST. | Patient uses POST doctor leave covering 10:00-10:15 IST. Data: UTC 2026-10-05T04:30:00Z to 04:45:00Z. | UTC 2026-10-05T04:30:00Z to 04:45:00Z. | HTTP 409; type schedule-change-conflicts; affected_appointment_ids contains 77777777-7777-4777-8777-777777777777; leave not stored. |
| TC-Q-019 | Availability subtracts booked slot | Service positive | Must | FR-AVAIL-01 | Rohan works 09:00-10:00 IST and has BOOKED 09:15. | Patient uses GET /api/v1/doctors/{doctor_id}/availability for 2026-10-05. Data: Slot length 15. | Slot length 15. | HTTP 200; returned starts are exactly 09:00, 09:30, 09:45 IST. |
| TC-Q-020 | Availability omits past slots | Service boundary | Must | FR-AVAIL-01 | Time frozen at 2026-10-05 09:20 IST. | Patient sends GET /api/v1/doctors/{doctor_id}/availability with Rohan schedule 09:00-10:00 IST. | Rohan schedule 09:00-10:00 IST. | HTTP 200; no slot before 09:20 IST appears; first possible start is 09:30 IST. |
| TC-Q-021 | Availability rejects date after window | API boundary | Must | FR-AVAIL-01 | Time frozen at 2026-10-02. | Patient uses GET /api/v1/doctors/{doctor_id}/availability for 2026-11-02. Data: Date 31 days ahead. | Date 31 days ahead. | HTTP 422; type booking-window-exceeded. |
| TC-Q-022 | Search by specialization and partial name | API positive | Must | FR-SEARCH-01 | Rohan active in General Medicine. | Admin sends GET /api/v1/doctors?specialization=General%20Medicine&name=roh with Default pagination. | Default pagination. | HTTP 200; one item includes Dr. Rohan Mehta; limit is 20 and offset is 0. |
| TC-Q-023 | Search filters available doctor date | API positive | Must | FR-SEARCH-01 | Rohan has free slot on 2026-10-05; another doctor has none. | Patient sends GET /api/v1/doctors?available_on=2026-10-05 with Seed schedules. | Seed schedules. | HTTP 200; Rohan appears and the fully booked doctor does not. |
| TC-Q-024 | Search limit over 100 rejected | API boundary | Must | FR-SEARCH-01 | Any authenticated token exists. | Patient sends GET /api/v1/doctors?limit=101 with limit=101. | limit=101. | HTTP 422; type pagination-limit-exceeded. |
| TC-Q-025 | Patient books own slot | API positive | Must | FR-BOOK-01 | Asha has 0 upcoming active appointments; slot is free. | Patient uses Asha submits POST /api/v1/appointments without patient_id. Data: Rohan at 2026-10-05T10:00:00+05:30. | Rohan at 2026-10-05T10:00:00+05:30. | HTTP 201; status BOOKED; appointment end is 10:15:00+05:30; one audit row has to_status BOOKED. |
| TC-Q-026 | Active booked slot blocks second booking | API negative | Must | FR-BOOK-01 | TC-Q-025 appointment is BOOKED. | Patient uses Another patient POSTs same doctor and start. Data: Patient vivek.iyer@example.in. | Patient vivek.iyer@example.in. | HTTP 409; type slot-already-booked; appointment count for slot remains 1. |
| TC-Q-027 | Same patient same doctor date blocked | API negative | Must | FR-BOOK-01 | Asha has BOOKED Rohan at 10:00 on 2026-10-05. | Patient uses Asha books Rohan at 11:00 same IST date. Data: 2026-10-05T11:00:00+05:30. | 2026-10-05T11:00:00+05:30. | HTTP 409; type patient-doctor-date-limit-reached; no new row created. |
| TC-Q-028 | Fourth upcoming appointment blocked | API negative | Must | FR-BOOK-01 | Asha has 3 upcoming active appointments with different doctors or dates. | Patient uses Asha books another valid future slot. Data: Fourth slot 2026-10-06T10:00:00+05:30. | Fourth slot 2026-10-06T10:00:00+05:30. | HTTP 409; type booking-limit-reached; active upcoming count remains 3. |
| TC-Q-029 | Same-slot concurrency exact counts | Concurrency | Must | FR-BOOK-02 | Slot is free; 10 patients exist. | Patient uses Send 10 parallel POST /api/v1/appointments for same doctor and start. Data: Rohan 2026-10-07T10:00:00+05:30. | Rohan 2026-10-07T10:00:00+05:30. | Exactly 1 response is HTTP 201 and exactly 9 are HTTP 409 slot-already-booked; database has 1 active row. |
| TC-Q-030 | Patient limit concurrency exact cap | Concurrency | Must | FR-BOOK-02 | Asha has 0 upcoming active appointments. | Patient uses Send 5 parallel bookings for 5 valid different slots. Data: Five starts from 2026-10-08 to 2026-10-12. | Five starts from 2026-10-08 to 2026-10-12. | At most 3 responses are HTTP 201; remaining responses are HTTP 409 booking-limit-reached; active upcoming count is 3. |
| TC-Q-031 | Cancelled slot can be rebooked | Concurrency/recovery | Must | FR-BOOK-02 | Appointment 77777777-7777-4777-8777-777777777777 is CANCELLED. | Patient uses Another patient books same doctor and start. Data: Same slot as 77777777-7777-4777-8777-777777777777. | Same slot as 77777777-7777-4777-8777-777777777777. | HTTP 201; new appointment ID differs from 77777777-7777-4777-8777-777777777777; active slot count is 1. |
| TC-Q-032 | Patient list separates upcoming and past | API positive | Must | FR-APPT-02 | Asha has appointments on 2026-09-20 and 2026-10-05. | Patient uses GET /api/v1/patients/me/appointments?view=upcoming, then view=past. Data: Time frozen 2026-10-02. | Time frozen 2026-10-02. | Upcoming response contains 2026-10-05 only; past response contains 2026-09-20 only. |
| TC-Q-033 | Patient cancellation before window succeeds | API positive | Must | FR-APPT-02 | Asha owns BOOKED appointment at 18:00 IST. | Patient uses Freeze 15:59 IST and POST cancel. Data: Patient cancellation reason Patient recovered. | Patient cancellation reason Patient recovered. | HTTP 200; status CANCELLED; audit row stores from BOOKED to CANCELLED. |
| TC-Q-034 | Patient late cancellation rejected | API negative | Must | FR-APPT-02 | Asha owns BOOKED appointment at 18:00 IST. | Patient uses Freeze 16:30 IST and POST cancel. Data: Reason Traffic delay. | Reason Traffic delay. | HTTP 409; type cancellation-window-closed; appointment remains BOOKED. |
| TC-Q-035 | Receptionist sees all-doctor day schedule | API positive | Must | FR-APPT-03 | Neha receptionist token exists; two doctors have appointments. | Receptionist sends GET /api/v1/reception/schedule?date=2026-10-05 with Seed day schedule. | Seed day schedule. | HTTP 200; response has two doctor groups sorted by doctor name and appointments sorted by start time. |
| TC-Q-036 | Receptionist check-in succeeds | API positive | Must | FR-APPT-01 | Appointment 77777777-7777-4777-8777-777777777777 is BOOKED. | Receptionist sends POST /api/v1/appointments/77777777-7777-4777-8777-777777777777/check-in with arrival reason and observes status. Data: Check-in reason Patient arrived at front desk. | Check-in reason Patient arrived at front desk. | HTTP 200; status CHECKED_IN; one audit row records actor role RECEPTIONIST. |
| TC-Q-037 | Cannot check in cancelled appointment | API negative | Must | FR-APPT-01 | Appointment 88888888-8888-4888-8888-888888888888 is CANCELLED. | Receptionist sends POST /api/v1/appointments/88888888-8888-4888-8888-888888888888/check-in and observes the error. Data: Receptionist token. | Receptionist token. | HTTP 409; type invalid-status-transition; status remains CANCELLED. |
| TC-Q-038 | Receptionist no-show after 15 minutes succeeds | API positive | Must | FR-APPT-03 | BOOKED appointment starts 10:00 IST. | Receptionist uses Freeze 10:16 IST and POST no-show. Data: No-show reason Patient did not arrive. | No-show reason Patient did not arrive. | HTTP 200; status NO_SHOW; audit row reason is stored. |
| TC-Q-039 | Receptionist no-show too early rejected | API negative | Must | FR-APPT-03 | BOOKED appointment starts 10:00 IST. | Receptionist uses Freeze 10:10 IST and POST no-show. Data: Reason Not present. | Reason Not present. | HTTP 409; type no-show-too-early; status remains BOOKED. |
| TC-Q-040 | Doctor sees own schedule only | Authorization positive | Must | FR-APPT-04 | Rohan and Nair both have appointments on 2026-10-05. | Doctor uses Rohan GETs /api/v1/doctors/me/schedule?date=2026-10-05. Data: Rohan doctor token. | Rohan doctor token. | HTTP 200; top-level doctor_id equals 55555555-5555-4555-8555-555555555555; no appointment for doctor 99999999-9999-4999-8999-999999999999 appears. |
| TC-Q-041 | Doctor completes checked-in own appointment | API positive | Must | FR-APPT-04 | Rohan appointment is CHECKED_IN. | Doctor uses POST /api/v1/appointments/{appointment_id}/complete as Rohan. Data: Completion reason Consultation completed. | Completion reason Consultation completed. | HTTP 200; status COMPLETED; audit row stores from CHECKED_IN to COMPLETED. |
| TC-Q-042 | Doctor cannot complete another doctor's appointment | Authorization negative | Must | FR-APPT-04 | Nair appointment is CHECKED_IN. | Doctor uses Rohan POSTs complete for Nair appointment. Data: Rohan token, Nair appointment ID. | Rohan token, Nair appointment ID. | HTTP 403; type permission-denied; Nair appointment remains CHECKED_IN. |
| TC-Q-043 | Audit list shows status history | API positive | Must | FR-AUDIT-01 | Appointment moved BOOKED to CHECKED_IN. | Patient uses GET /api/v1/appointments/{id}/audit as admin. Data: Appointment ID from TC-Q-036. | Appointment ID from TC-Q-036. | HTTP 200; first row has from_status null and to_status BOOKED; second row has from_status BOOKED and to_status CHECKED_IN. |
| TC-Q-044 | No audit update endpoint exists | API negative | Must | FR-AUDIT-01 | Audit row exists. | Patient sends PATCH /api/v1/appointments/{id}/audit/{audit_id} with Any admin token. | Any admin token. | HTTP 404; type not-found; audit row is unchanged. |
| TC-Q-045 | Doctor performance report returns rank | Integration positive | Must | FR-REPORT-01 | October data has Rohan revenue 22800 and Nair revenue 12000. | Admin uses GET /api/v1/admin/reports/doctor-performance for 2026-10-01 to 2026-10-31. Data: Admin token. | Admin token. | HTTP 200; Rohan row has completed 38, no_shows 3, revenue_inr 22800.00, revenue_rank 1. |
| TC-Q-046 | Receptionist cannot view report | Authorization negative | Must | FR-REPORT-01 | Receptionist token exists. | Receptionist sends GET /api/v1/admin/reports/doctor-performance with Date range 2026-10-01 to 2026-10-31. | Date range 2026-10-01 to 2026-10-31. | HTTP 403; type permission-denied; no report rows are returned. |
| TC-Q-047 | Problem Details shape on validation error | API negative | Must | FR-API-01 | Patient token exists. | Patient uses POST appointment with malformed starts_at. Data: starts_at value Monday morning. | starts_at value Monday morning. | HTTP 422; body has type, title, status, detail, instance, errors, and request_id. |
| TC-Q-048 | Unknown API route returns not found | API negative | Must | FR-API-01 | API is running. | API client sends GET /api/v1/unknown-clinicq-route with No body. | No body. | HTTP 404; type not-found; response is Problem Details. |
| TC-Q-049 | Appointment response uses IST offset | API positive | Must | FR-API-01 | Appointment starts at 2026-10-05T04:30:00Z in DB. | Patient sends GET /api/v1/patients/me/appointments with Asha token. | Asha token. | HTTP 200; response starts_at is exactly 2026-10-05T10:00:00+05:30. |
| TC-Q-050 | Liveness does not query database | API positive | Must | FR-OPS-01 | API process runs with database stopped. | API client sends GET /api/v1/health/live with No auth. | No auth. | HTTP 200; body status is live; database query counter remains 0. |
| TC-Q-051 | Readiness fails when database down | API negative | Must | FR-OPS-01 | API process runs and PostgreSQL is stopped. | API client sends GET /api/v1/health/ready with No auth. | No auth. | HTTP 503; type database-not-ready. |
| TC-Q-052 | Seed creates exact demo counts | Integration positive | Must | FR-OPS-01 | Empty local database after migrations. | Test runner uses Run seed command once. Data: Demo clinic seed. | Demo clinic seed. | Database has exactly 5 doctors, 20 patients, 2 weeks of schedules, and at least 100 appointments. |
| TC-Q-053 | JSON request log contains request ID | Observability | Must | FR-OPS-01 | API logging captures one health request. | Patient uses GET readiness with DB up. Data: Header X-Request-ID: req-test-53. | Header X-Request-ID: req-test-53. | One JSON log has request_id req-test-53, method GET, path /api/v1/health/ready, status 200, and duration_ms. |
| TC-Q-054 | Local API demonstration covers core flow | Demo | Must | FR-CLIENT-01; BR-22 | API seeded; selected collection or Python HTTP driver uses `127.0.0.1:8001`. | Run auth, search, availability, booking, state, conflict, report, and health requests. | Generated local credentials and isolated booking fixture. | Exact success assertions pass; schedule conflict returns `409 schedule-change-conflicts`; saved evidence omits raw tokens/secrets. |
| TC-Q-055 | Patient blocked from admin doctor endpoint | Authorization negative | Must | FR-AUTH-04 | Patient token exists. | Patient sends POST /api/v1/admin/doctors with Valid Rohan-like doctor payload. | Valid Rohan-like doctor payload. | HTTP 403; type permission-denied; doctor table unchanged. |
| TC-Q-056 | Doctor blocked from patient profile update | Authorization negative | Must | FR-AUTH-04 | Doctor token exists. | Doctor sends PATCH /api/v1/patients/me/profile with full_name Bad Actor. | full_name Bad Actor. | HTTP 403; type permission-denied; no patient profile changes. |
| TC-Q-057 | Receptionist cannot complete appointment | Authorization negative | Must | FR-AUTH-04 | Appointment is CHECKED_IN. | Receptionist uses Receptionist POSTs complete. Data: Appointment ID 77777777-7777-4777-8777-777777777777. | Appointment ID 77777777-7777-4777-8777-777777777777. | HTTP 403; type permission-denied; status remains CHECKED_IN. |
| TC-Q-058 | Admin can perform receptionist schedule action | Authorization positive | Must | FR-APPT-03 | Admin token exists and schedule has appointments. | Admin sends GET /api/v1/reception/schedule?date=2026-10-05 with Admin token. | Admin token. | HTTP 200; grouped schedule is returned with all doctors. |
| TC-Q-059 | Expired access token rejected | Security negative | Must | FR-AUTH-02 | Access token issued at 10:00 IST. | Patient uses Freeze 10:16 IST and GET patient appointments. Data: Expired bearer token. | Expired bearer token. | HTTP 401; type token-expired; endpoint body is not returned. |
| TC-Q-060 | Tampered JWT rejected | Security negative | Must | FR-AUTH-02 | Valid token exists. | Patient uses Change one character in token signature and GET patient appointments. Data: Tampered bearer token. | Tampered bearer token. | HTTP 401; type invalid-token; no database data is returned. |
| TC-Q-061 | Password and raw refresh token are not logged | Security negative | Must | NFR-SEC-01 | Logging captures register and login requests. | Patient uses Register and login as Asha, then inspect captured logs. Data: Password <password>, refresh token from login. | Password <password>, refresh token from login. | No log entry contains <password>, raw refresh token, or Authorization header value. |
| TC-Q-062 | Availability Must performance target | Performance | Must | NFR-PERF-01 | Seed data loaded on standard profile. | Test runner uses Run Locust or equivalent with 20 concurrent users for availability endpoint after warm-up. Data: 80% Rohan availability, 20% other doctors. | 80% Rohan availability, 20% other doctors. | p95 availability latency is below 500 ms; error rate is 0%. |
| TC-Q-063 | Booking performance target | Performance | Must | NFR-PERF-03 | Seed data has enough free future slots. | Test runner uses Run 100 sequential normal booking requests with unique patients or cleanup between requests. Data: Free slots across 5 doctors. | Free slots across 5 doctors. | Booking p95 is below 800 ms; no duplicate active slot is created. |
| TC-Q-064 | Locust Should target | Performance | Should | FR-PERF-01 | Standard profile, seeded database, API warmed. | Test runner uses Run Locust for 50 concurrent users against availability for 5 minutes. Data: Seed 5 doctors and 20 patients. | Seed 5 doctors and 20 patients. | Availability p95 is below 300 ms; failed requests are below 1%. |
| TC-Q-065 | Reminder ambiguous crash allows duplicate e-mail | Worker | Should | FR-NOTIF-02 | Celery, Redis, Mailpit, and outbox row exist. | Worker sends SMTP message, crashes before setting sent_at, then retries the same outbox row. Data: Idempotency key reminder:77777777-7777-4777-8777-777777777777:24h. | Idempotency key reminder:77777777-7777-4777-8777-777777777777:24h. | There is still exactly one outbox row; duplicate Mailpit e-mail is allowed; retry records sent_at. |
| TC-Q-069 | Patient profile update succeeds | API positive | Must | FR-APPT-02 | Asha patient token exists. | Patient sends PATCH /api/v1/patients/me/profile with full_name Asha S. Kulkarni, phone 9876543210. | full_name Asha S. Kulkarni, phone 9876543210. | HTTP 200; response has Asha patient ID and updated name; no other patient row changes. |
| TC-Q-070 | Booking exactly now rejected | API boundary | Must | FR-BOOK-01 | Time is frozen at 2026-10-05 10:00 IST and slot starts at the same instant. | Patient uses Asha submits the booking request at the frozen current time. Data: Rohan start 2026-10-05T10:00:00+05:30. | Rohan start 2026-10-05T10:00:00+05:30. | HTTP 422; type booking-start-in-past; no appointment row is created. |
| TC-Q-071 | Booking at thirty days succeeds | API boundary | Must | FR-BOOK-01 | Time is frozen at 2026-10-02 10:00 IST and a scheduled free slot exists exactly 30 days later. | Patient uses Asha submits the booking request exactly 30 days ahead. Data: Start 2026-11-01T10:00:00+05:30. | Booking at thirty days succeeds data: Start 2026-11-01T10:00:00+05:30. | HTTP 201; status BOOKED; stored start matches the requested instant. |
| TC-Q-072 | Booking after thirty days rejected | API boundary | Must | FR-BOOK-01 | Time is frozen at 2026-10-02 10:00 IST. | Patient uses Asha submits the booking request 30 days and 1 minute ahead. Data: Start 2026-11-01T10:01:00+05:30. | Booking after thirty days rejected data: Start 2026-11-01T10:01:00+05:30. | HTTP 422; type booking-window-exceeded; no appointment row is created. |
| TC-Q-073 | Off-schedule booking rejected | API negative | Must | FR-BOOK-01 | Rohan works 09:00-10:00 IST only. | Patient uses POST /api/v1/appointments for 10:15 IST. Data: Asha token and Rohan doctor ID. | Asha token and Rohan doctor ID. | HTTP 422; type slot-not-on-schedule; no appointment row is created. |
| TC-Q-074 | On-leave booking rejected | API negative | Must | FR-BOOK-01 | Rohan has leave [10:00, 10:15) IST. | Patient uses POST /api/v1/appointments for 10:00 IST. Data: Asha token and Rohan doctor ID. | Asha token and Rohan doctor ID. | HTTP 409; type slot-not-available; no appointment row is created. |
| TC-Q-075 | Weekly schedule conflict with active bookings | API negative | Must | FR-DOC-02 | Appointment 77777777-7777-4777-8777-777777777777 is BOOKED at 10:00 IST. | Admin uses PUT /api/v1/admin/doctors/{doctor_id}/weekly-sessions that remove the 10:00 slot. Data: Admin token and shortened Monday sessions. | Admin token and shortened Monday sessions. | HTTP 409; type schedule-change-conflicts; affected_appointment_ids contains the UUID for 77777777-7777-4777-8777-777777777777. |
| TC-Q-076 | Concurrent patient and doctor limits hold | Concurrency | Must | FR-BOOK-02 | Asha has no active appointment with Rohan on 2026-10-09. | Patient uses Send two parallel bookings for Asha and Rohan on the same IST date with different starts. Data: Starts 10:00 and 10:30 IST. | Starts 10:00 and 10:30 IST. | At most one response is HTTP 201; any rejected response is HTTP 409 patient-doctor-date-limit-reached; active count for Asha, Rohan, and date is 1. |
| TC-Q-077 | Report uses fee at booking | Integration positive | Must | FR-REPORT-01 | Rohan appointment was booked at INR 600, completed, and Rohan current fee is INR 700. | Admin uses GET admin doctor performance report for the appointment date. Data: Admin token and inclusive date range. | Admin token and inclusive date range. | HTTP 200; Rohan revenue_inr includes 600.00, not 700.00. |
| TC-Q-078 | Reminder completed redelivery is skipped | Worker | Should | FR-NOTIF-02 | Reminder outbox row has sent_at set. | Worker uses Run due reminder worker again. Data: Idempotency key for that appointment. | Idempotency key for that appointment. | Worker sends no new Mailpit message and leaves sent_at unchanged. |
| TC-Q-079 | OpenAPI document validates | API contract | Must | FR-API-01 | API process is running in local/dev. | Test runner uses GET /api/v1/openapi.json and validate it with the chosen OpenAPI checker. Data: No auth. | No auth. | HTTP 200; document contains all Must paths; checker reports no schema errors. |
| TC-Q-080 | Admin lists staff users | Authorization positive | Must | FR-AUTH-04 | Admin token exists and Neha staff user exists. | Admin sends GET /api/v1/admin/staff-users with limit 20 and observes the list. Data: Admin bearer token. | Admin bearer token. | HTTP 200; response contains Neha and excludes password hashes. |
| TC-Q-081 | Deactivated account login rejected | Security negative | Must | FR-AUTH-02 | Neha account has is_active false. | Patient uses Neha sends POST /api/v1/auth/login with generated local test password and observes the error. Data: neha.patil@example.in and generated password placeholder. | neha.patil@example.in and generated password placeholder. | HTTP 401; type account-deactivated; no token is issued. |
| TC-Q-082 | Deactivated account refresh rejected | Security negative | Must | FR-AUTH-03 | Neha has a refresh token and account is deactivated. | Patient uses Neha sends POST /api/v1/auth/refresh with <refresh_token> and observes the error. Data: <refresh_token> for Neha. | <refresh_token> for Neha. | HTTP 401; type account-deactivated; no new token row is inserted. |
| TC-Q-083 | Admin creates specialization | Authorization positive | Must | FR-DOC-01 | Admin token exists. | Admin sends POST /api/v1/admin/specializations with name Dermatology and observes creation. | Name Dermatology. | HTTP 201; active is true and name is stored once. |
| TC-Q-084 | Patient cannot create specialization | Authorization negative | Must | FR-DOC-01 | Patient token exists. | Patient sends POST /api/v1/admin/specializations with name Cardiology and observes denial. | Name Cardiology. | HTTP 403; type permission-denied; specialization is not stored. |
| TC-Q-085 | User lists specializations | Authorization positive | Must | FR-DOC-01 | Asha patient token exists and General Medicine is active. | Patient sends GET /api/v1/specializations and observes catalog output. Data: Patient bearer token. | Patient bearer token. | HTTP 200; General Medicine appears without inactive setup-only fields. |
| TC-Q-086 | User gets active doctor | API positive | Must | FR-DOC-01 | D-100 doctor is active. | Patient sends GET /api/v1/doctors/55555555-5555-4555-8555-555555555555 and observes profile. Data: Patient bearer token. | Patient bearer token. | HTTP 200; doctor_id equals D-100 UUID and active is true. |
| TC-Q-087 | Admin deactivates doctor profile | API positive | Must | FR-DOC-01 | D-100 has no future active appointments. | Admin sends POST /api/v1/admin/doctors/55555555-5555-4555-8555-555555555555/deactivate with reason and observes status. Data: Reason Doctor left clinic. | Reason Doctor left clinic. | HTTP 200; active is false and historical appointments remain. |
| TC-Q-088 | Overlapping weekly sessions rejected | API negative | Must | FR-DOC-02 | Admin token exists and D-100 is active. | Admin sends PUT /api/v1/admin/doctors/55555555-5555-4555-8555-555555555555/weekly-sessions with overlapping Tuesday sessions and observes validation. Data: Tuesday 09:00-12:00 and 11:30-14:00. | Tuesday 09:00-12:00 and 11:30-14:00. | HTTP 422; type sessions-overlap; previous schedule remains active. |
| TC-Q-089 | Past-start booking rejected | API boundary | Must | FR-BOOK-01 | Time is frozen at 2026-10-05 10:00 IST. | Patient uses Asha sends POST /api/v1/appointments for 09:45 IST with D-100 and observes validation. Data: Start 2026-10-05T09:45:00+05:30. | Past-start booking rejected data: Start 2026-10-05T09:45:00+05:30. | HTTP 422; type booking-start-in-past; no appointment row is created. |
| TC-Q-090 | Slot ending at leave start remains free | Service boundary | Must | FR-AVAIL-01 | D-100 works 09:30-10:30 IST and leave is [10:00, 10:15). | Patient sends GET /api/v1/doctors/55555555-5555-4555-8555-555555555555/availability for the date and checks the 09:45-10:00 slot. Data: Date with leave boundary. | Date with leave boundary. | HTTP 200; 09:45-10:00 appears and 10:00-10:15 is absent. |
| TC-Q-091 | Slot starting at leave end remains free | Service boundary | Must | FR-AVAIL-01 | D-100 works 09:30-10:30 IST and leave is [10:00, 10:15). | Patient sends GET /api/v1/doctors/55555555-5555-4555-8555-555555555555/availability for the date and checks the 10:15-10:30 slot. Data: Date with leave boundary. | Date with leave boundary. | HTTP 200; 10:15-10:30 appears and 10:00-10:15 is absent. |
| TC-Q-092 | Receptionist books registered patient | Authorization positive | Must | FR-BOOK-01 | Neha token exists and Asha patient_id exists. | Receptionist sends POST /api/v1/appointments with patient_id, D-100, future start, and reason. Data: Asha patient_id and free slot. | Asha patient_id and free slot. | HTTP 201; booked_by_user_id is Neha and patient_id is Asha. |
| TC-Q-093 | Patient cannot book for another patient | Authorization negative | Must | FR-BOOK-01 | Asha and Vivek patient profiles exist. | Patient uses Asha sends POST /api/v1/appointments with Vivek patient_id and observes denial. Data: Vivek patient_id in request body. | Vivek patient_id in request body. | HTTP 403; type permission-denied; no Vivek appointment is created. |
| TC-Q-094 | Patient reads own profile | Authorization positive | Must | FR-APPT-02 | Asha token exists. | Patient uses Asha sends GET /api/v1/patients/me and observes profile fields. Data: Asha bearer token. | Asha bearer token. | HTTP 200; patient_id, full_name, phone, and email are returned. |
| TC-Q-095 | Doctor cannot read patient profile | Authorization negative | Must | FR-APPT-02 | Doctor token exists. | Doctor sends GET /api/v1/patients/me and observes denial. Data: Doctor bearer token. | Doctor bearer token. | HTTP 403; type permission-denied; no patient profile body is returned. |
| TC-Q-096 | Patient cannot cancel another appointment | Authorization negative | Must | FR-APPT-02 | Vivek owns a BOOKED appointment. | Patient uses Asha sends POST /api/v1/appointments/{vivek_appointment_id}/cancel and observes denial. Data: Vivek appointment ID and reason. | Vivek appointment ID and reason. | HTTP 403; type permission-denied; appointment remains BOOKED. |
| TC-Q-097 | Patient cannot read audit log | Authorization negative | Must | FR-AUDIT-01 | Appointment audit rows exist. | Patient uses Asha sends GET /api/v1/appointments/77777777-7777-4777-8777-777777777777/audit and observes denial. Data: Asha bearer token. | Asha bearer token. | HTTP 403; type permission-denied; audit rows are not returned. |
| TC-Q-098 | Receptionist reads audit log | Authorization positive | Must | FR-AUDIT-01 | Appointment audit rows exist. | Receptionist uses Neha sends GET /api/v1/appointments/77777777-7777-4777-8777-777777777777/audit and observes history. Data: Receptionist bearer token. | Receptionist bearer token. | HTTP 200; status history is returned in changed_at order. |
| TC-Q-099 | Expanded error-contract matrix | API contract | Should | FR-API-02; BR-18 | Selected Should/Could endpoints and error examples exist | Replay documented expired-token, wrong-role, conflict, and invalid-slot requests | Local optional-feature fixtures; slot length `25` | Expected `401/403/409/422` type/status/request-ID contracts pass; no stack traces, tokens, or cross-patient records |

## Required local acceptance cases

Implement these as a host-side Python/pytest driver in `tests/local/`, invoking document 06's canonical entry points and loopback HTTP endpoints. A container that the stop test terminates cannot be the test driver. Use `LOCAL_INSTANCE=clinicq_acceptance` with isolated test volumes/configuration, not the student's normal demo data; stop/reset must never affect an unrelated project.

Before the offline phase, cache locked packages and pinned application/DB/test-helper images. Disable external egress while keeping loopback/Compose networking. Save JUnit plus exact health/availability JSON, sanitized appointment/audit IDs and fee snapshots, and configuration/fixture hashes. Never publish raw tokens or generated signing secrets.

| ID | Title | Type | Priority | Linked requirement IDs | Preconditions | Steps | Test data | Expected result |
|---|---|---|---|---|---|---|---|---|
| TC-LOCAL-001 | Clean clone initializes local state | INFRA | Must | FR-OPS-01; BR-19; BR-31; NFR-OPS-01 | Clean implementation clone, cached downloads, empty isolated volumes | Run lite start; GET live/ready; login Asha; run start again | Recorded seed: five doctors, twenty patients, two-week context, at least 100 appointments | Exact ready message and health JSON; login `200 expires_in=900`; second start preserves IDs/counts/context and signing configuration |
| TC-LOCAL-002 | Deterministic core operation offline | API | Must | Local fixtures FR-OPS-01; slots FR-AVAIL-01; offline runtime NFR-OPS-03 | External egress disabled; local networks retained; fresh acceptance seed with test business clock | Start cached stack; login Asha; GET Rohan availability for `2026-10-05`; run runtime tests | Fixed Rohan UUID, Monday `09:00-10:00`, no leave/bookings in reserved slots | `200`; exact four-slot JSON from document 06; no image pull, external source fetch, SMTP provider, or test skips |
| TC-LOCAL-003 | Appointment, audit, tokens persist | IT | Must | Restart FR-OPS-01; retained local state BR-31; durability NFR-REL-03 | Seeded Asha has no upcoming appointment; local credentials exist | Book reserved `09:00` slot; capture IDs/fee/audit count and refresh family; stop; start; reauthenticate and inspect | New BOOKED appointment; fee `600.00`; signing-config hash | Same appointment ID/status/start/fee, audit count `1`, and token-family records remain; previous unexpired refresh can rotate; secrets were not regenerated |
| TC-LOCAL-004 | Invalid payload and DB/port failures are clear | API | Must | Dependency checks FR-OPS-01; Problem Details FR-API-01; clear diagnostics NFR-OPS-04 | Local stack has existing appointment/audit state | Send malformed `starts_at`; stop DB and call live/ready/protected API; restore DB; start with API port occupied | Invalid string `Monday morning`; request ID `req-local-ready` | Payload `422` Problem Details; live `200`; ready/protected call `503 database-not-ready`; occupied-port start nonzero with document 06 diagnostic; no partial writes/deletes |
| TC-LOCAL-005 | Local services publish only loopback | INFRA | Must | FR-OPS-01; BR-31 | Lite profile running; optional local services tested if selected | Inspect resolved Compose bindings and connection targets | Host ports `8001,25432,16379,21025,28025` | All enabled bindings use `127.0.0.1`, never public interfaces; internal DB `5432`; no Celery host port |
| TC-LOCAL-006 | Explicit reset stays project-local | INFRA | Must | Reset command FR-OPS-01; project boundary BR-31; retention safety NFR-REL-03 | Disposable acceptance records/config and a separate sentinel volume exist | Reset without flag; verify records; reset with exact confirmation | Acceptance appointment/audit/signing config only | Unconfirmed reset fails with exact confirmation message, retaining records; confirmed reset removes only acceptance data/config; sentinel and other projects survive |

Local trainer pre-check `LOCAL-PRECHECK` is the checklist in document 06. Record actual resource observations and OS; the proposed 8 GB budgets are not claimed test results.

## Fixture aliases

Every alias used in the catalog is replaced by its mapped UUID in paths, request bodies, and assertions.

| Alias | Fixed UUID | Purpose |
|---|---|---|
| A-300 | 77777777-7777-4777-8777-777777777777 for main appointment | Main Rohan appointment in state tests. |
| A-301 | 88888888-8888-4888-8888-888888888888 for cancelled appointment | Cancelled appointment in negative state test. |
| D-100 | 55555555-5555-4555-8555-555555555555 for Rohan doctor | Dr. Rohan profile. |
| D-200 | 99999999-9999-4999-8999-999999999999 for Nair doctor | Dr. Nair profile. |


## Authorization matrix tests

The catalog includes representative role checks. The implementation MUST also keep a generated or hand-maintained authorization matrix test that covers every endpoint group below against Patient, Doctor, Receptionist, and Admin.

| Endpoint group | Patient | Doctor | Receptionist | Admin | Required evidence |
|---|---|---|---|---|---|
| Patient self endpoints | Allow own profile and appointments | Deny | Deny | Deny | TC-Q-032, TC-Q-056, TC-Q-069 for patient self endpoints |
| Doctor own schedule and complete | Deny | Allow own doctor only | Deny | Deny for doctor-only route | TC-Q-040, TC-Q-041, TC-Q-057 for doctor own schedule and complete |
| Reception schedule and state operations | Deny | Deny | Allow | Allow | TC-Q-035, TC-Q-036, TC-Q-058 for reception schedule and state operations |
| Admin doctor, schedule, staff, reports | Deny | Deny | Deny except no admin reports | Allow | TC-Q-012, TC-Q-046, TC-Q-055 for admin doctor, schedule, staff, reports |
| Search and availability | Allow | Allow | Allow | Allow | TC-Q-019, TC-Q-022 for search and availability |
| Booking | Allow own patient | Deny | Allow registered patients | Allow registered patients | TC-Q-025, TC-Q-026, TC-Q-030 for booking |
| Audit read | Deny | Deny | Allow | Allow | TC-Q-043 for audit read |
| Health and OpenAPI | Public | Public | Public | Public | TC-Q-050, TC-Q-051 for health and OpenAPI |

## Authorization route matrix

Allowed means the role can call the route. Owner rules are enforced in addition to role rules.

| Method and path | Patient | Doctor | Receptionist | Admin | Owner or scope rule | Test IDs |
|---|---|---|---|---|---|---|
| POST /api/v1/auth/register | Public | Public | Public | Public | Creates only a Patient account. | TC-Q-001 |
| POST /api/v1/auth/login | Public | Public | Public | Public | Deactivated users receive account-deactivated. | TC-Q-004, TC-Q-081 |
| POST /api/v1/auth/refresh | Public | Public | Public | Public | Valid refresh token owner only; deactivated owner is rejected. | TC-Q-007 rotation, TC-Q-008 reuse, TC-Q-082 deactivated |
| POST /api/v1/auth/logout | Allow | Allow | Allow | Allow | Presented refresh token owner only. | TC-Q-010 |
| POST /api/v1/admin/staff-users | Deny | Deny | Deny | Allow | Admin creates staff accounts. | TC-Q-011, TC-Q-012 |
| GET /api/v1/admin/staff-users | Deny | Deny | Deny | Allow | Admin lists staff accounts. | TC-Q-080 |
| POST /api/v1/admin/staff-users/{user_id}/deactivate | Deny | Deny | Deny | Allow | Admin deactivates staff users. | TC-Q-066, TC-Q-067 |
| POST /api/v1/admin/specializations | Deny | Deny | Deny | Allow | Admin manages specialization setup. | TC-Q-083, TC-Q-084 |
| GET /api/v1/specializations | Allow | Allow | Allow | Allow | Authenticated catalog read. | TC-Q-085 |
| POST /api/v1/admin/doctors | Deny | Deny | Deny | Allow | Admin creates doctor profile. | TC-Q-013, TC-Q-055 |
| GET /api/v1/doctors/{doctor_id} | Allow | Allow | Allow | Allow | Active profile is readable; inactive public access returns doctor-not-active. | TC-Q-086, TC-Q-015 |
| PATCH /api/v1/admin/doctors/{doctor_id} | Deny | Deny | Deny | Allow | Admin updates profile. | TC-Q-014 |
| POST /api/v1/admin/doctors/{doctor_id}/deactivate | Deny | Deny | Deny | Allow | Admin deactivates doctor profile. | TC-Q-087 |
| PUT /api/v1/admin/doctors/{doctor_id}/weekly-sessions | Deny | Deny | Deny | Allow | Conflicts and overlapping sessions are rejected. | TC-Q-016 save, TC-Q-075 conflict, TC-Q-088 overlap |
| POST /api/v1/admin/doctors/{doctor_id}/leaves | Deny | Deny | Deny | Allow | Conflicts return affected appointment IDs. | TC-Q-018 |
| GET /api/v1/doctors/{doctor_id}/availability | Allow | Allow | Allow | Allow | Public clinic data after authentication. | TC-Q-019 |
| GET /api/v1/doctors | Allow | Allow | Allow | Allow | Public clinic data after authentication. | TC-Q-022 |
| POST /api/v1/appointments | Own only | Deny | Allow | Allow | Patient omits patient_id; staff must send it; non-owner patient is denied. | TC-Q-025 self, TC-Q-092 staff, TC-Q-093 non-owner |
| GET /api/v1/patients/me | Own only | Deny | Deny | Deny | Patient profile owner only. | TC-Q-094, TC-Q-095 |
| GET /api/v1/patients/me/appointments | Own only | Deny | Deny | Deny | Patient appointment owner only. | TC-Q-032 |
| PATCH /api/v1/patients/me/profile | Own only | Deny | Deny | Deny | Patient profile owner only. | TC-Q-056, TC-Q-069 |
| POST /api/v1/appointments/{appointment_id}/cancel | Own only | Deny | Allow | Allow | Patient owner has 2-hour guard; non-owner patient is denied; staff before start. | TC-Q-033 early, TC-Q-034 late, TC-Q-096 non-owner |
| GET /api/v1/reception/schedule | Deny | Deny | Allow | Allow | All-doctor day schedule. | TC-Q-035, TC-Q-058 |
| POST /api/v1/appointments/{appointment_id}/check-in | Deny | Deny | Allow | Allow | Reception operation. | TC-Q-036 |
| POST /api/v1/appointments/{appointment_id}/no-show | Deny | Deny | Allow | Allow | Reception operation after 15 minutes. | TC-Q-038, TC-Q-039 |
| GET /api/v1/doctors/me/schedule | Deny | Own doctor | Deny | Deny | Non-owner doctor appointments are excluded. | TC-Q-040 |
| POST /api/v1/appointments/{appointment_id}/complete | Deny | Own doctor | Deny | Deny | Non-owner doctor receives 403. | TC-Q-041, TC-Q-042 |
| GET /api/v1/appointments/{appointment_id}/audit | Deny | Deny | Allow | Allow | Receptionist and Admin only. | TC-Q-097, TC-Q-098 |
| GET /api/v1/admin/reports/doctor-performance | Deny | Deny | Deny | Allow | Admin report only. | TC-Q-045, TC-Q-046 |
| GET /api/v1/health/live | Public | Public | Public | Public | No database query. | TC-Q-050 |
| GET /api/v1/health/ready | Public | Public | Public | Public | Checks database connectivity. | TC-Q-051 |
| GET /api/v1/openapi.json and GET /docs | Public | Public | Public | Public | Local/dev documentation only. | TC-Q-079 |

## Requirement traceability matrix

| Requirement IDs | Test cases or demo step |
|---|---|
| FR-AUTH-01 registration rules BR-01 and BR-02 | TC-Q-001, TC-Q-002, TC-Q-003 for patient registration |
| FR-AUTH-02, BR-03 | TC-Q-004, TC-Q-005, TC-Q-006, TC-Q-059, TC-Q-060 for login expiry lockout |
| FR-AUTH-03, BR-04 | TC-Q-007, TC-Q-008, TC-Q-009, TC-Q-010 for refresh logout |
| FR-AUTH-04, BR-05 | TC-Q-011, TC-Q-012, TC-Q-055, TC-Q-056, TC-Q-057, TC-Q-058, TC-Q-066, TC-Q-067, TC-Q-068 for staff RBAC |
| FR-DOC-01, BR-06 | TC-Q-013, TC-Q-014, TC-Q-015 for doctor profile |
| FR-DOC-02 schedule rules BR-07 and BR-08 | TC-Q-016, TC-Q-017, TC-Q-018, TC-Q-075 for schedule leave conflict |
| FR-AVAIL-01 availability rules BR-09 and BR-10 | TC-Q-019, TC-Q-020, TC-Q-021, TC-Q-070, TC-Q-071, TC-Q-072, TC-Q-074, TC-Q-062 for availability windows |
| FR-BOOK-01, BR-09, BR-10, BR-11, BR-12, BR-13 | TC-Q-025, TC-Q-026, TC-Q-027, TC-Q-028, TC-Q-070, TC-Q-071, TC-Q-072, TC-Q-073, TC-Q-074, TC-Q-063 for booking rules |
| FR-BOOK-02, BR-10, BR-11, BR-12, BR-13 | TC-Q-029, TC-Q-030, TC-Q-031, TC-Q-076 for booking races |
| FR-APPT-01, BR-14, BR-15, BR-16 | TC-Q-033, TC-Q-034, TC-Q-036, TC-Q-037, TC-Q-038, TC-Q-039, TC-Q-041 for state machine |
| FR-APPT-02 | TC-Q-032, TC-Q-033, TC-Q-034, TC-Q-069 for patient appointments |
| FR-APPT-03 | TC-Q-035, TC-Q-036, TC-Q-038, TC-Q-039, TC-Q-058 for reception operations |
| FR-APPT-04 | TC-Q-040, TC-Q-041, TC-Q-042 for doctor operations |
| FR-SEARCH-01, BR-17 | TC-Q-022, TC-Q-023, TC-Q-024 for doctor search |
| FR-API-01, BR-18 | TC-Q-047, TC-Q-048, TC-Q-049, TC-Q-079 for api standards |
| FR-OPS-01, BR-19, BR-31 | TC-Q-050, TC-Q-051, TC-Q-052, TC-Q-053 and TC-LOCAL-001 through TC-LOCAL-006 for health, fixtures, local lifecycle, persistence, and failures |
| FR-AUDIT-01, BR-20 | TC-Q-025, TC-Q-036, TC-Q-041, TC-Q-043, TC-Q-044 for audit history |
| FR-REPORT-01, BR-21 | TC-Q-045, TC-Q-046, TC-Q-077 for admin report |
| FR-CLIENT-01, BR-22 | TC-Q-054 for local API collection or Python HTTP demonstration |
| FR-NOTIF-01, BR-23 | Demo step: book and cancel appointment, then show one booking e-mail and one cancellation e-mail in Mailpit. for booking mailpit mail |
| FR-NOTIF-02, BR-24 | TC-Q-065 and TC-Q-078 for reminder worker |
| FR-DOC-03, BR-25 | Demo step: save 20-minute slot doctor, then reject 25-minute slot length. for variable slot length |
| FR-PAT-01, BR-26 | Demo step: receptionist creates a walk-in patient with user_id null and books one slot. for walk-in patient |
| FR-CLIN-01, BR-27 | Demo step: doctor writes visit notes after check-in; patient reads notes after completion. for visit notes |
| FR-REPORT-02 | Demo step: admin downloads appointments-per-day CSV with header row. for csv extra reports |
| FR-API-02 | TC-Q-099 for expanded local API error contracts |
| FR-PERF-01, BR-28 | TC-Q-064 for locust target |
| FR-APPT-05 | Demo step: reschedule changes old and new slots atomically or rolls both back. for atomic reschedule |
| FR-WAIT-01, BR-29 | Demo step: first waitlisted patient receives notification after cancellation. for waitlist fifo |
| FR-TECH-01, BR-30 | Demo step: show ADR and test proving cache invalidates on booking or schedule change. for cache or async choice |
| FR-TEST-01 | Demo step: Schemathesis run covers /api/v1 OpenAPI without undocumented 500 errors. for schemathesis contract |
| NFR-PERF-01 | TC-Q-062 for locust target |
| NFR-PERF-02 | TC-Q-064 for should availability latency |
| NFR-PERF-03 | TC-Q-063 for booking latency |
| NFR-SCALE-01 | TC-Q-052 for seed scale |
| NFR-REL-01 | TC-Q-029, TC-Q-030, TC-Q-031 for race safety |
| NFR-REL-02 | TC-Q-065 and TC-Q-078 for reminder reliability |
| NFR-REL-03 | TC-LOCAL-003 and TC-LOCAL-006 for persisted data/configuration and confirmed reset |
| NFR-SEC-01 | TC-Q-001, TC-Q-061 for password secrecy |
| NFR-SEC-02 | TC-Q-004, TC-Q-007, TC-Q-008, TC-Q-009, TC-Q-059, TC-Q-060, TC-Q-068 for token safety |
| NFR-SEC-03 | TC-Q-005, TC-Q-006 for lockout security |
| NFR-SEC-04 | TC-Q-012, TC-Q-055, TC-Q-056, TC-Q-057, TC-Q-058, TC-Q-067 for authorization coverage |
| NFR-SEC-05 | TC-Q-047, TC-Q-060, TC-Q-061 for owasp checks |
| NFR-PRIV-01 | TC-Q-001 and privacy review demo of stored patient fields only for minimum data |
| NFR-PRIV-02 | README privacy section review demo for dpdp awareness |
| NFR-MAINT-01 | CI step: Ruff 0.16.x and mypy 2.4.x pass for lint type |
| NFR-MAINT-02 | Coverage report enforcing 85% line and 75% branch for total coverage |
| NFR-MAINT-03 | Service coverage report enforcing 95% line and 85% branch on booking, slots, and state machine for service coverage |
| NFR-MAINT-04 | CI pytest collection/pass summary and catalog mapping: at least 50 meaningful cases, including all Must local cases |
| NFR-OBS-01 | TC-Q-053, TC-Q-061 for request logs |
| NFR-OPS-01 | TC-LOCAL-001 plus trainer OS clean-clone start/stop check |
| NFR-OPS-02 | LOCAL-PRECHECK from document 06 on 8 GB/four-core hardware |
| NFR-OPS-03 | TC-LOCAL-002 for deterministic offline fixtures/runtime |
| NFR-OPS-04 | TC-LOCAL-004 plus TC-Q-047 for dependency and input diagnostics |
| NFR-LIC-01 | Dependency review of pyproject.toml and image tags for licence compliance |
| NFR-API-01 | TC-Q-047, TC-Q-048 for api standards |

## Performance-test conditions

| Condition | Exact value |
|---|---|
| Reference hardware | Standard 16 GB profile: API 1 GB, PostgreSQL 1 GB, worker 512 MB; local Docker Compose with pinned images. |
| Seed data | 5 doctors, 20 patients, 2 weeks schedules, at least 100 appointments. |
| Availability Must run | 20 concurrent users, 2-minute warm-up, 5-minute measured run, p95 below 500 ms, 0% errors. |
| Availability Should Locust run | 50 concurrent users, 2-minute warm-up, 5-minute measured run, p95 below 300 ms, below 1% errors. |
| Request mix | 80% GET /api/v1/doctors/{id}/availability, 15% GET /api/v1/doctors, 5% GET /api/v1/health/ready. |
| Booking latency run | 100 non-conflicting booking requests using unique free slots; p95 below 800 ms. |
| Evidence | Save Locust text or HTML report and note commit SHA, machine profile, seed timestamp, and database size. |

## Defect report fields

Each bug report MUST include: title, environment, commit SHA, endpoint/service/local command, requirement IDs, fixture hash and saved demo date, reproduction steps, exact expected/actual result, status/Problem Details, sanitized request-ID logs, severity, owner, and fix verification test ID. Use local Mailpit API/outbox or Locust output as operational evidence; no business-interface screenshots are a deliverable.

## Entry criteria

- Alembic migrations run successfully on an empty PostgreSQL 18.x database.
- Seed command can create the demo clinic without real personal data.
- .env.example exists and has no secrets.
- Ruff, mypy, and unit tests pass locally before API integration tests begin.
- Test factories can create users, doctors, schedules, appointments, and tokens.
- Locked dependencies and pinned images are cached before offline runtime acceptance; demo and test volumes are isolated.

## Exit criteria

- All Must-priority tests in this catalog pass.
- At least 50 meaningful Python cases pass; catalog coverage cannot be replaced by duplicate assertions.
- Coverage gates pass separately for line and branch thresholds.
- Same-slot concurrency gives exactly 1 HTTP 201 and 9 HTTP 409.
- Patient parallel booking limit gives at most 3 successful bookings.
- Readiness, logs, Problem Details, and authorization matrix tests pass.
- OpenAPI document check, availability EXPLAIN ANALYZE, and report EXPLAIN ANALYZE evidence are named and stored.
- Performance evidence meets Must targets; Should Locust evidence is present if FR-PERF-01 is claimed.
- The local API collection or Python HTTP demonstration completes without publishing raw tokens, credentials, or real personal data.
- TC-LOCAL-001 through TC-LOCAL-006 pass with health/availability JSON, persisted appointment/audit/fee/token evidence, loopback bindings, and safe reset.

[Back to README](../README.md)
