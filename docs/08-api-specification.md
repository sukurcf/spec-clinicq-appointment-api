# API specification

Purpose: This document defines the ClinicQ REST API contract for the `clinicq-api` student implementation.

## API-wide rules

- Base path: `/api/v1`.
- Local base URL: `http://127.0.0.1:8001/api/v1`; API clients or Python HTTP tests are the business demonstration tools.
- Format: JSON request/response bodies and OpenAPI JSON; selected Should reports may return CSV. Built-in `/docs` is optional local documentation inspection, not student-built interface work.
- Authentication: protected endpoints require `Authorization: Bearer <access_token>`.
- Access token lifetime: 15 minutes. Refresh token lifetime: 7 days.
- Stored timestamps: UTC. Display timestamps in appointment and availability responses use ISO 8601 with Asia/Kolkata offset, for example `2026-10-05T10:00:00+05:30`.
- Versioning: all product endpoints are under `/api/v1`. Breaking changes require `/api/v2`; additive fields may be added to `/api/v1`.
- Pagination: list endpoints use `limit` and `offset`. Default `limit` is 20, maximum is 100, and `offset` defaults to 0.
- Sorting: doctor search supports `name_asc`, `fee_asc`, `fee_desc`, and `next_available_asc`. Appointment lists sort by `starts_at` ascending for upcoming and descending for past unless a route states otherwise.
- If PostgreSQL is unavailable during a protected operation, return `503` Problem Details type `database-not-ready` without driver exceptions or partial appointment/audit writes.

## Endpoint summary

| Method | Path | Roles | FR IDs |
|---|---|---|---|
| POST | /api/v1/auth/register | Public patient | FR-AUTH-01 |
| POST | /api/v1/auth/login | Public | FR-AUTH-02 |
| POST | /api/v1/auth/refresh | Public with refresh token | FR-AUTH-03 |
| POST | /api/v1/auth/logout | Authenticated users | FR-AUTH-03 |
| POST | /api/v1/admin/staff-users | Admin | FR-AUTH-04 |
| GET | /api/v1/admin/staff-users | Admin | FR-AUTH-04 |
| POST | /api/v1/admin/staff-users/{user_id}/deactivate | Admin | FR-AUTH-04 |
| POST | /api/v1/admin/specializations | Admin | FR-DOC-01 |
| GET | /api/v1/specializations | Patient, Doctor, Receptionist, Admin | FR-DOC-01 |
| POST | /api/v1/admin/doctors | Admin | FR-DOC-01 |
| GET | /api/v1/doctors/{doctor_id} | Patient, Doctor, Receptionist, Admin | FR-DOC-01 |
| PATCH | /api/v1/admin/doctors/{doctor_id} | Admin | FR-DOC-01 |
| POST | /api/v1/admin/doctors/{doctor_id}/deactivate | Admin | FR-DOC-01 |
| PUT | /api/v1/admin/doctors/{doctor_id}/weekly-sessions | Admin | FR-DOC-02 |
| POST | /api/v1/admin/doctors/{doctor_id}/leaves | Admin | FR-DOC-02 |
| GET | /api/v1/doctors/{doctor_id}/availability | Patient, Doctor, Receptionist, Admin | FR-AVAIL-01 |
| GET | /api/v1/doctors | Patient, Doctor, Receptionist, Admin | FR-SEARCH-01 |
| POST | /api/v1/appointments | Patient, Receptionist, Admin | FR-BOOK-01, FR-BOOK-02 |
| GET | /api/v1/patients/me | Patient | FR-APPT-02 |
| GET | /api/v1/patients/me/appointments | Patient | FR-APPT-02 |
| PATCH | /api/v1/patients/me/profile | Patient | FR-APPT-02 |
| POST | /api/v1/appointments/{appointment_id}/cancel | Patient, Receptionist, Admin | FR-APPT-01, FR-APPT-02, FR-APPT-03 |
| GET | /api/v1/reception/schedule | Receptionist, Admin | FR-APPT-03 |
| POST | /api/v1/appointments/{appointment_id}/check-in | Receptionist, Admin | FR-APPT-01, FR-APPT-03 |
| POST | /api/v1/appointments/{appointment_id}/no-show | Receptionist, Admin | FR-APPT-01, FR-APPT-03 |
| GET | /api/v1/doctors/me/schedule | Doctor | FR-APPT-04 |
| POST | /api/v1/appointments/{appointment_id}/complete | Doctor | FR-APPT-01, FR-APPT-04 |
| GET | /api/v1/appointments/{appointment_id}/audit | Receptionist, Admin | FR-AUDIT-01 |
| GET | /api/v1/admin/reports/doctor-performance | Admin | FR-REPORT-01 |
| GET | /api/v1/health/live | Public | FR-OPS-01 |
| GET | /api/v1/health/ready | Public | FR-OPS-01 |
| GET | /api/v1/openapi.json | Public | FR-API-01 |
| GET | `/docs` | Public in local/dev | FR-API-01 |

## Error format

All errors MUST use RFC 9457 Problem Details.

```json
{
  "type": "slot-already-booked",
  "title": "Slot already booked",
  "status": 409,
  "detail": "Dr. Rohan Mehta already has an active appointment at 2026-10-05T10:00:00+05:30.",
  "instance": "/api/v1/appointments",
  "errors": [
    {"field": "starts_at", "code": "slot-already-booked"}
  ],
  "request_id": "req_01JZ9E2S6Y3K"
}
```

Problem Details `type` values used by Must endpoints are: `email-already-registered`, `password-policy-failed`, `account-temporarily-locked`, `invalid-credentials`, `token-expired`, `invalid-token`, `account-deactivated`, `refresh-token-reused`, `refresh-token-revoked`, `permission-denied`, `doctor-not-active`, `specialization-not-active`, `fee-must-be-non-negative`, `too-many-sessions-for-weekday`, `sessions-overlap`, `booking-start-in-past`, `booking-window-exceeded`, `slot-not-on-schedule`, `slot-not-available`, `slot-already-booked`, `booking-limit-reached`, `patient-doctor-date-limit-reached`, `schedule-change-conflicts`, `invalid-status-transition`, `cancellation-window-closed`, `no-show-too-early`, `invalid-date-range`, `pagination-limit-exceeded`, `database-not-ready`, and `not-found`.

## Auth flows

### Register patient

POST /api/v1/auth/register

Auth: none. Creates only a `PATIENT` account and linked patient profile.

Request fields: `email` string, `password` string, `full_name` string, `phone` Indian 10-digit string. Test passwords are generated locally and are not written in this spec.

```json
{
  "email": "asha.kulkarni@example.in",
  "password": "<password>",
  "full_name": "Asha Kulkarni",
  "phone": "9876543210"
}
```

Response `201`:

```json
{
  "user_id": "11111111-1111-4111-8111-111111111111",
  "patient_id": "22222222-2222-4222-8222-222222222222",
  "email": "asha.kulkarni@example.in",
  "role": "PATIENT",
  "full_name": "Asha Kulkarni"
}
```

Register patient outcomes: `201`, `409` type `email-already-registered`, `422` type `password-policy-failed`.

### Login

POST /api/v1/auth/login

Auth: none. Five failures in 15 minutes lock the account for 15 minutes.

Request:

```json
{
  "email": "asha.kulkarni@example.in",
  "password": "<password>"
}
```

Response `200`:

```json
{
  "access_token": "<access_token>",
  "refresh_token": "<refresh_token>",
  "token_type": "bearer",
  "expires_in": 900,
  "user": {
    "id": "11111111-1111-4111-8111-111111111111",
    "email": "asha.kulkarni@example.in",
    "role": "PATIENT"
  }
}
```

Login outcomes: `200`, `401` type `invalid-credentials`, `401` type `account-deactivated`, `429` type `account-temporarily-locked`.

### Refresh

POST /api/v1/auth/refresh

Auth: refresh token in body. A successful call rotates the refresh token. Reusing a rotated token revokes the full token family.

Request:

```json
{
  "refresh_token": "<refresh_token>"
}
```

Response `200`:

```json
{
  "access_token": "<access_token>",
  "refresh_token": "<new_refresh_token>",
  "token_type": "bearer",
  "expires_in": 900
}
```

Refresh outcomes: `200`, `401` type `refresh-token-reused`, `401` type `refresh-token-revoked`, `401` type `account-deactivated`.

### Logout

POST /api/v1/auth/logout

Auth: bearer token and presented refresh token. Revokes the presented refresh token.

Request:

```json
{
  "refresh_token": "<new_refresh_token>"
}
```

Response `204`: empty body.

Logout outcomes: `204`, `401` type `invalid-token`, `401` type `token-expired`, `401` type `refresh-token-revoked`, `401` type `account-deactivated`.

## Admin users and doctors

### Create staff user

POST /api/v1/admin/staff-users

Auth: Admin.

Request fields: `email`, `temporary_password`, `role` (`DOCTOR`, `RECEPTIONIST`, or `ADMIN`), `full_name`, optional `doctor_id`.

```json
{
  "email": "neha.patil@example.in",
  "temporary_password": "<temporary_password>",
  "role": "RECEPTIONIST",
  "full_name": "Neha Patil"
}
```

Response `201`:

```json
{
  "user_id": "33333333-3333-4333-8333-333333333333",
  "email": "neha.patil@example.in",
  "role": "RECEPTIONIST",
  "is_active": true
}
```

Create staff user outcomes: `201`, `403` type `permission-denied`, `409` type `email-already-registered`, `422` type `password-policy-failed`.

### List staff users

GET /api/v1/admin/staff-users?limit=20&offset=0

Auth: Admin.

Response `200`:

```json
{
  "items": [
    {"user_id": "33333333-3333-4333-8333-333333333333", "email": "neha.patil@example.in", "role": "RECEPTIONIST", "is_active": true}
  ],
  "limit": 20,
  "offset": 0,
  "total": 1
}
```

List staff users outcomes: `200`, `403` type `permission-denied`, `422` type `pagination-limit-exceeded`.

### Deactivate staff user

POST /api/v1/admin/staff-users/{user_id}/deactivate

Auth: Admin. The user cannot log in, refresh, or use existing access tokens after deactivation.

Request:

```json
{"reason": "Receptionist left clinic"}
```

Response `200`:

```json
{
  "user_id": "33333333-3333-4333-8333-333333333333",
  "email": "neha.patil@example.in",
  "role": "RECEPTIONIST",
  "is_active": false
}
```

Deactivate staff user outcomes: `200`, `403` type `permission-denied`, `404` type `not-found`.

### Create specialization

POST /api/v1/admin/specializations

Auth: Admin.

Request:

```json
{"name": "General Medicine"}
```

Response `201`:

```json
{"id": "44444444-4444-4444-8444-444444444444", "name": "General Medicine", "active": true}
```

Create specialization outcomes: `201`, `403` type `permission-denied`.

### List specializations

GET /api/v1/specializations

Auth: bearer token for Patient, Doctor, Receptionist, or Admin.

Response `200`:

```json
{"items": [{"id": "44444444-4444-4444-8444-444444444444", "name": "General Medicine"}]}
```

List specializations outcomes: `200`, `401` type `invalid-token` when a bearer token is malformed.

### Create doctor

POST /api/v1/admin/doctors

Auth: Admin.

Request fields: `full_name`, `specialization_id`, `consultation_fee_inr`, optional `user_id`. Must-scope `slot_length_minutes` is `15`.

```json
{
  "full_name": "Dr. Rohan Mehta",
  "specialization_id": "44444444-4444-4444-8444-444444444444",
  "consultation_fee_inr": "600.00",
  "slot_length_minutes": 15
}
```

Response `201`:

```json
{
  "doctor_id": "55555555-5555-4555-8555-555555555555",
  "full_name": "Dr. Rohan Mehta",
  "specialization": "General Medicine",
  "consultation_fee_inr": "600.00",
  "slot_length_minutes": 15,
  "active": true
}
```

Create doctor outcomes: `201`, `403` type `permission-denied`, `422` type `specialization-not-active` when specialization is inactive, `422` type `fee-must-be-non-negative`.

### Get doctor

GET /api/v1/doctors/{doctor_id}

Auth for Get doctor: Patient, Doctor, Receptionist, or Admin.

Response `200`:

```json
{
  "doctor_id": "55555555-5555-4555-8555-555555555555",
  "full_name": "Dr. Rohan Mehta",
  "specialization": "General Medicine",
  "consultation_fee_inr": "600.00",
  "slot_length_minutes": 15,
  "active": true
}
```

Get doctor outcomes: `200`, `404` type `not-found`, `404` type `doctor-not-active` for inactive public access.

### Update doctor

PATCH /api/v1/admin/doctors/{doctor_id}

Auth: Admin.

Request:

```json
{
  "consultation_fee_inr": "700.00",
  "specialization_id": "44444444-4444-4444-8444-444444444444"
}
```

Response `200`:

```json
{"doctor_id": "55555555-5555-4555-8555-555555555555", "consultation_fee_inr": "700.00", "active": true}
```

Update doctor outcomes: `200`, `403` type `permission-denied`, `422` type `fee-must-be-non-negative`, `404` type `not-found`.

### Deactivate doctor

POST /api/v1/admin/doctors/{doctor_id}/deactivate

Auth: Admin. Historical appointments remain.

Request:

```json
{"reason": "Doctor left clinic"}
```

Response `200`:

```json
{"doctor_id": "55555555-5555-4555-8555-555555555555", "active": false}
```

Deactivate doctor outcomes: `200`, `403` type `permission-denied`, `409` type `schedule-change-conflicts`, `404` type `not-found`.

### Replace weekly sessions

PUT /api/v1/admin/doctors/{doctor_id}/weekly-sessions

Auth: Admin. Each weekday has 1 or 2 non-overlapping sessions. Times are Asia/Kolkata local times.

Request:

```json
{
  "sessions": [
    {"weekday": "MONDAY", "session_number": 1, "starts_at_local": "09:00", "ends_at_local": "13:00"},
    {"weekday": "MONDAY", "session_number": 2, "starts_at_local": "17:00", "ends_at_local": "20:00"}
  ]
}
```

Response `200`:

```json
{"doctor_id": "55555555-5555-4555-8555-555555555555", "sessions_saved": 2}
```

Replace weekly sessions outcomes: `200`, `403` type `permission-denied`, `409` type `schedule-change-conflicts`, `422` type `too-many-sessions-for-weekday`, `422` type `sessions-overlap`, `404` type `not-found`.

Conflict response includes affected IDs:

```json
{
  "type": "schedule-change-conflicts",
  "title": "Schedule change conflicts with appointments",
  "status": 409,
  "detail": "The change conflicts with active appointments.",
  "instance": "/api/v1/admin/doctors/55555555-5555-4555-8555-555555555555/weekly-sessions",
  "affected_appointment_ids": ["77777777-7777-4777-8777-777777777777"]
}
```

### Create leave or holiday

POST /api/v1/admin/doctors/{doctor_id}/leaves

Auth: Admin.

Request:

```json
{
  "starts_at": "2026-10-05T04:30:00Z",
  "ends_at": "2026-10-05T05:30:00Z",
  "reason": "Conference"
}
```

Response `201`:

```json
{
  "leave_id": "66666666-6666-4666-8666-666666666666",
  "doctor_id": "55555555-5555-4555-8555-555555555555",
  "starts_at": "2026-10-05T04:30:00Z",
  "ends_at": "2026-10-05T05:30:00Z"
}
```

Leave intervals are half-open `[starts_at, ends_at)`. A slot is blocked when any part of it intersects the leave.

Create leave or holiday outcomes: `201`, `403` type `permission-denied`, `409` type `schedule-change-conflicts`, `404` type `not-found`.

## Search and availability

### Search doctors

GET /api/v1/doctors?specialization=General%20Medicine&name=roh&available_on=2026-10-05&limit=20&offset=0&sort=next_available_asc

Auth for Search doctors: Patient, Doctor, Receptionist, or Admin.

Response `200`:

```json
{
  "items": [
    {
      "doctor_id": "55555555-5555-4555-8555-555555555555",
      "full_name": "Dr. Rohan Mehta",
      "specialization": "General Medicine",
      "consultation_fee_inr": "600.00",
      "next_available_at": "2026-10-05T10:00:00+05:30"
    }
  ],
  "limit": 20,
  "offset": 0,
  "total": 1,
  "sort": "next_available_asc"
}
```

Search doctors outcomes: `200`, `422` type `pagination-limit-exceeded`.

### Get availability

GET /api/v1/doctors/{doctor_id}/availability?date=2026-10-05

Auth for Get availability: Patient, Doctor, Receptionist, or Admin.

Response `200`:

```json
{
  "doctor_id": "55555555-5555-4555-8555-555555555555",
  "date": "2026-10-05",
  "time_zone": "Asia/Kolkata",
  "slot_length_minutes": 15,
  "slots": [
    {"starts_at": "2026-10-05T10:00:00+05:30", "ends_at": "2026-10-05T10:15:00+05:30"},
    {"starts_at": "2026-10-05T10:30:00+05:30", "ends_at": "2026-10-05T10:45:00+05:30"}
  ]
}
```

Get availability outcomes: `200`, `404` type `doctor-not-active`, `422` type `booking-window-exceeded`.

Slot starts are generated from the session start in 15-minute steps. A slot is returned only when its end is no later than the session end. The date window includes starts strictly after now and no later than now plus 30 days.

For local acceptance, use document 06's fresh recorded fixture and saved `demo_date`, not the illustrative slots above. Its expected response has exactly four slots from `09:00` through `09:45` IST, fixed doctor UUID, timezone, and 15-minute length. With the restricted automated-test clock, date `2026-10-05` and the full JSON in document 06 are exact assertions. External network access is disabled during that operation.

## Appointments

### Book appointment

POST /api/v1/appointments

Auth: Patient books self; Receptionist or Admin books a registered patient. A Patient request MUST omit `patient_id`; the API derives it from the token. A Receptionist or Admin request MUST include `patient_id`.

Request:

```json
{
  "doctor_id": "55555555-5555-4555-8555-555555555555",
  "patient_id": "22222222-2222-4222-8222-222222222222",
  "starts_at": "2026-10-05T10:00:00+05:30",
  "reason": "Fever consultation"
}
```

For a Patient caller, the request omits `patient_id`:

```json
{
  "doctor_id": "55555555-5555-4555-8555-555555555555",
  "starts_at": "2026-10-05T10:00:00+05:30",
  "reason": "Fever consultation"
}
```

Response `201`:

```json
{
  "appointment_id": "77777777-7777-4777-8777-777777777777",
  "doctor_id": "55555555-5555-4555-8555-555555555555",
  "patient_id": "22222222-2222-4222-8222-222222222222",
  "status": "BOOKED",
  "starts_at": "2026-10-05T10:00:00+05:30",
  "ends_at": "2026-10-05T10:15:00+05:30",
  "fee_at_booking_inr": "600.00"
}
```

Book appointment outcomes: `201`, `403` type `permission-denied`, `404` type `doctor-not-active`, `409` type `slot-already-booked`, `409` type `slot-not-available` for leave conflicts, `409` type `booking-limit-reached`, `409` type `patient-doctor-date-limit-reached`, `422` type `booking-start-in-past`, `422` type `booking-window-exceeded`, `422` type `slot-not-on-schedule` for off-schedule starts.

### Get patient profile

GET /api/v1/patients/me

Auth: Patient.

Response `200`:

```json
{
  "patient_id": "22222222-2222-4222-8222-222222222222",
  "full_name": "Asha Kulkarni",
  "phone": "9876543210",
  "email": "asha.kulkarni@example.in"
}
```

Get patient profile outcomes: `200`, `403` type `permission-denied`.

### Patient appointment list

GET /api/v1/patients/me/appointments?view=upcoming&limit=20&offset=0

Auth: Patient.

Response `200`:

```json
{
  "items": [
    {
      "appointment_id": "77777777-7777-4777-8777-777777777777",
      "doctor_name": "Dr. Rohan Mehta",
      "status": "BOOKED",
      "starts_at": "2026-10-05T10:00:00+05:30"
    }
  ],
  "view": "upcoming",
  "limit": 20,
  "offset": 0
}
```

Patient appointment list outcomes: `200`, `403` type `permission-denied`, `422` type `pagination-limit-exceeded`.

### Update patient profile

PATCH /api/v1/patients/me/profile

Auth: Patient.

Request:

```json
{"full_name": "Asha S. Kulkarni", "phone": "9876543210"}
```

Response `200`:

```json
{"patient_id": "22222222-2222-4222-8222-222222222222", "full_name": "Asha S. Kulkarni", "phone": "9876543210"}
```

Update patient profile outcomes: `200`, `403` type `permission-denied`.

### Cancel appointment

POST /api/v1/appointments/{appointment_id}/cancel

Auth: Patient for own `BOOKED` appointment at least 2 hours before start; Receptionist or Admin before start.

Request:

```json
{"reason": "Patient recovered"}
```

Response `200`:

```json
{
  "appointment_id": "77777777-7777-4777-8777-777777777777",
  "from_status": "BOOKED",
  "status": "CANCELLED",
  "cancelled_at": "2026-10-05T09:00:00+05:30"
}
```

Cancel appointment outcomes: `200`, `403` type `permission-denied`, `409` type `cancellation-window-closed`, `409` type `invalid-status-transition`, `404` type `not-found`.

### Reception schedule

GET /api/v1/reception/schedule?date=2026-10-05

Auth: Receptionist or Admin.

Response `200`:

```json
{
  "date": "2026-10-05",
  "doctors": [
    {
      "doctor_id": "55555555-5555-4555-8555-555555555555",
      "doctor_name": "Dr. Rohan Mehta",
      "appointments": [
        {"appointment_id": "77777777-7777-4777-8777-777777777777", "patient_name": "Asha Kulkarni", "starts_at": "2026-10-05T10:00:00+05:30", "status": "BOOKED"}
      ]
    }
  ]
}
```

Reception schedule outcomes: `200`, `403` type `permission-denied`.

### Check in

POST /api/v1/appointments/{appointment_id}/check-in

Auth: Receptionist or Admin.

Request:

```json
{"reason": "Patient arrived at front desk"}
```

Response `200`:

```json
{"appointment_id": "77777777-7777-4777-8777-777777777777", "from_status": "BOOKED", "status": "CHECKED_IN"}
```

Check in outcomes: `200`, `403` type `permission-denied`, `409` type `invalid-status-transition`, `404` type `not-found`.

### Mark no-show

POST /api/v1/appointments/{appointment_id}/no-show

Auth: Receptionist or Admin. Allowed at least 15 minutes after appointment start.

Request:

```json
{"reason": "Patient did not arrive by 10:16 IST"}
```

Response `200`:

```json
{"appointment_id": "77777777-7777-4777-8777-777777777777", "from_status": "BOOKED", "status": "NO_SHOW"}
```

Mark no-show outcomes: `200`, `403` type `permission-denied`, `409` type `no-show-too-early`, `409` type `invalid-status-transition`, `404` type `not-found`.

### Doctor day schedule

GET /api/v1/doctors/me/schedule?date=2026-10-05

Auth: Doctor.

Response `200`:

```json
{
  "doctor_id": "55555555-5555-4555-8555-555555555555",
  "date": "2026-10-05",
  "appointments": [
    {"appointment_id": "77777777-7777-4777-8777-777777777777", "patient_name": "Asha Kulkarni", "starts_at": "2026-10-05T10:00:00+05:30", "status": "CHECKED_IN"}
  ]
}
```

Doctor day schedule outcomes: `200`, `403` type `permission-denied`.

### Complete appointment

POST /api/v1/appointments/{appointment_id}/complete

Auth: assigned Doctor.

Request:

```json
{"reason": "Consultation completed"}
```

Response `200`:

```json
{"appointment_id": "77777777-7777-4777-8777-777777777777", "from_status": "CHECKED_IN", "status": "COMPLETED"}
```

Complete appointment outcomes: `200`, `403` type `permission-denied`, `409` type `invalid-status-transition`, `404` type `not-found`.

### Appointment audit

GET /api/v1/appointments/{appointment_id}/audit

Auth: Receptionist or Admin.

Response `200`:

```json
{
  "appointment_id": "77777777-7777-4777-8777-777777777777",
  "items": [
    {"from_status": null, "to_status": "BOOKED", "actor_user_id": "11111111-1111-4111-8111-111111111111", "changed_at": "2026-10-02T04:30:00Z", "reason": "Fever consultation"},
    {"from_status": "BOOKED", "to_status": "CHECKED_IN", "actor_user_id": "33333333-3333-4333-8333-333333333333", "changed_at": "2026-10-05T04:32:00Z", "reason": "Patient arrived at front desk"}
  ]
}
```

Appointment audit outcomes: `200`, `403` type `permission-denied`, `404` type `not-found`.

## Reports

### Doctor performance

GET /api/v1/admin/reports/doctor-performance?from_date=2026-10-01&to_date=2026-10-31

Auth: Admin. Implemented with hand-written SQL and bound parameters. Revenue is the sum of immutable booking-time fees for completed appointments whose start is in the inclusive IST date range. Appointment counts include all statuses in the range. Revenue rank uses standard rank ties.

Response `200`:

```json
{
  "from_date": "2026-10-01",
  "to_date": "2026-10-31",
  "items": [
    {
      "doctor_id": "55555555-5555-4555-8555-555555555555",
      "doctor_name": "Dr. Rohan Mehta",
      "appointments": 42,
      "completed": 38,
      "no_shows": 3,
      "revenue_inr": "22800.00",
      "revenue_rank": 1
    }
  ]
}
```

Doctor performance outcomes: `200`, `403` type `permission-denied`, `422` type `invalid-date-range` when `to_date` is before `from_date`.

## Health and OpenAPI

### Liveness

GET /api/v1/health/live

Auth: none. Must not query the database.

Response `200`:

```json
{"status": "live", "request_id": "req_01JZ9E2S6Y3K"}
```

Liveness outcomes: `200`.

Clients may send `X-Request-ID: req-local-live` for deterministic evidence; the liveness body echoes that request ID and has `status=live`. It must remain `200` when PostgreSQL is stopped.

### Readiness

GET /api/v1/health/ready

Auth: none. Checks database connectivity.

Response `200`:

```json
{"status": "ready", "database": "ok"}
```

Readiness outcomes: `200`, `503` type `database-not-ready`.

For `X-Request-ID: req-local-ready` with PostgreSQL unavailable, the exact failure contract is:

```json
{
  "type": "database-not-ready",
  "title": "Database not ready",
  "status": 503,
  "detail": "The local PostgreSQL service is unavailable.",
  "instance": "/api/v1/health/ready",
  "errors": [],
  "request_id": "req-local-ready"
}
```

### OpenAPI

`GET /api/v1/openapi.json` returns the generated OpenAPI document and is the required machine-readable contract. `GET /docs` is the built-in Swagger documentation console in local/dev only; students do not implement or style it. No offline acceptance depends on documentation-console assets or hosted API-client synchronization.

OpenAPI outcomes: `200`.

## Should-scope endpoints

If implemented, Should and Could endpoints MUST still use `/api/v1`, bearer tokens, Problem Details, and the same pagination rules.

| Endpoint | Roles | Requirement |
|---|---|---|
| POST /api/v1/reception/walk-in-patients | Receptionist, Admin | FR-PAT-01 |
| POST /api/v1/appointments/{appointment_id}/notes | assigned Doctor | FR-CLIN-01 |
| GET /api/v1/patients/me/appointments/{appointment_id}/notes | Patient owner | FR-CLIN-01 |
| GET /api/v1/admin/reports/appointments-per-day.csv | Admin | FR-REPORT-02 |
| POST /api/v1/appointments/{appointment_id}/reschedule | Patient, Receptionist, Admin | FR-APPT-05 |
| POST /api/v1/waitlist | Patient | FR-WAIT-01 |

[Back to README](../README.md)
