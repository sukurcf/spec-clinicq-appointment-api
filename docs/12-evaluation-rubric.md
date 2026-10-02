# Evaluation rubric

Purpose: This document defines mandatory gates, scoring weights, bonus rules, deductions, grade bands, and viva questions for ClinicQ.

## Mandatory gates

If any gate fails, the result is Rework required for any score.

| Gate | ClinicQ evidence required |
|---|---|
| CI is green on `main` | GitHub Actions show all required jobs passing. |
| Coverage thresholds are met | Total coverage is at least 85% line and 75% branch; service coverage is at least 95% line and 85% branch; auth coverage is at least 90% line and 80% branch; report coverage is at least 85% line and 75% branch. |
| Clean clone starts locally | Trainer follows README and starts API plus PostgreSQL on the lite profile. |
| No secrets in Git history | Gitleaks and manual review find no committed tokens, passwords, or real patient data. |
| Student can explain code | Student explains booking concurrency, refresh rotation, indexes, and state machine during viva. |
| Must FRs work in demo | All Must endpoints and the Bruno or Postman collection run against seed data. |

## Weighted rubric

| Category | Points | Excellent | Good | Needs work |
|---|---:|---|---|---|
| Functional completeness (Must FRs work in the demo) | 30 | All Must FRs work. Booking conflicts, lockout, refresh reuse, schedule conflict, audit, report, and collection evidence are shown. | One minor Must demo issue exists, but tests prove the behavior and no core booking rule is broken. | Any core Must flow is missing, or booking limits can be bypassed. |
| Code quality and design | 15 | Clear layered FastAPI design. Services own business rules. Transactions are explicit. ADRs explain concurrency, token, SQL report, e-mail, and deployment choices. | Structure is understandable. A few modules are too large, but core rules are isolated. | Business rules are spread across handlers, or concurrency relies on in-memory checks. |
| Testing and coverage | 15 | At least 50 strong tests cover positive, negative, authorization, concurrency, report, and token cases. Required coverage targets pass. | Coverage passes and important negative cases exist, but a few role pairs or edge cases are weak. | Coverage fails, concurrency tests are absent, or tests use only happy paths. |
| DevOps: CI/CD, Docker, reproducibility | 10 | Clean clone starts in 10 steps or fewer. Compose profiles work. CI gates, scans, migrations, and seed are reliable. | Local setup works with one documented workaround. CI is mostly complete. | Setup depends on hidden local state, unpinned images, or manual database edits. |
| Documentation (README, ADRs, API/CLI/pipeline docs) | 10 | README, OpenAPI, ADRs, query-plan notes, and collection instructions are clear and consistent. | Docs explain setup and core decisions, but one evidence note is thin. | Docs are stale, missing ADRs, or do not show how to demo ClinicQ. |
| Git and engineering practices (PRs, commits, issues) | 5 | Small PRs, good issue links, Conventional Commits, and clear PR checklists are visible. | Most changes use PRs and readable commits. | Large unreviewed commits or direct pushes hide the work history. |
| Final demo and viva | 15 | Demo is smooth, under 10 minutes, and the student answers SQL, HTTP, JWT, Docker, and race-condition questions with examples. | Demo covers the main path. Student answers most questions with some prompting. | Demo fails often, or the student cannot explain their own implementation. |
| **Total** | **100** |  |  |  |

## Bonus rules

- Bonus is up to +10 points.
- Bonus applies only when the base score is 60 or more.
- Bonus applies only when all mandatory gates pass.
- Final score is capped at 100.

| Bonus item | Maximum points | Conditions |
|---|---:|---|
| FR-TEST-01 | 3 | Schemathesis contract run covers `/api/v1` and finds no undocumented 500 errors. |
| FR-APPT-05 | 3 | Reschedule changes old and new slots atomically or rolls both back. |
| FR-WAIT-01 | 3 | Waitlist uses FIFO notification with transaction-safe tests. |
| FR-DEPLOY-01 | 1 | Free cloud demo includes cost warning and destroy-after-demo proof. |

## Deductions

| Issue | Deduction |
|---|---:|
| Uses real patient data or real clinic data | Mandatory gate failure |
| Commits a secret or raw token | Mandatory gate failure |
| Uses `latest` image tags in required services | -5 |
| Missing Bruno or Postman collection | -5 |
| No EXPLAIN ANALYZE evidence for required queries | -5 |
| No ADR for concurrency strategy | -5 |
| API errors do not use Problem Details | -5 |
| Direct pushes to `main` for most work | -3 |
| Demo exceeds 10 minutes without completing core flow | -3 |
| README setup takes more than 10 steps from clean clone | -3 |

## Grade bands

| Score | Grade |
|---:|---|
| 85–100 | Distinction |
| 70–84 | Merit |
| 60–69 | Pass |
| Below 60 | Rework required |

## Viva questions

1. How does ClinicQ prevent two patients from booking Dr. Rohan at 10:00 IST?
2. Why are `BOOKED` and `CHECKED_IN` the only active statuses?
3. What happens when a rotated refresh token is reused?
4. Why is the patient-doctor-date limit based on Asia/Kolkata, not UTC?
5. How did you test the 5-failure login lockout and HTTP 429 response?
6. Which indexes help the availability query, and what did EXPLAIN ANALYZE show?
7. Why is the doctor performance report written in SQL instead of only ORM code?
8. What is the difference between HTTP 401, 403, 409, 422, and 429 in ClinicQ?
9. How does the lite Docker profile fit an 8 GB laptop?
10. If Celery retries a reminder, when can a duplicate e-mail still happen?

[Back to README](../README.md)
