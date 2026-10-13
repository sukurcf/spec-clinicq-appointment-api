# Users and roles

## Roles

| Role | Responsibility |
|---|---|
| Data Engineer | Builds and runs the loader, inspects the quarantine, and maintains tests. |
| Trainer | Reviews outputs, tests the local contract, and grades the submission. |

## User stories

| ID | Story | Requirements |
|---|---|---|
| US-01 | As a Data Engineer, I want to load valid orders once so that a repeated file does not inflate the target. | FR-LOAD-01, FR-LOAD-03, FR-LOAD-04 |
| US-02 | As a Data Engineer, I want rejected rows and reason codes so that I can correct source data without losing it. | FR-LOAD-01, FR-LOAD-02 |
| US-03 | As a Trainer, I want deterministic counts and persistent local data so that I can verify a clean run and replay. | FR-LOAD-04, FR-LOAD-05 |

## Ownership and permissions

| Action | Data Engineer | Trainer |
|---|---|---|
| Read the fictional source and quarantine | Own local files | Review submitted evidence |
| Run a load or inspect the target | Local development database | Run the documented acceptance checks |
| Reset LoadLite data | Only with explicit confirmation | Only with explicit confirmation |
| Change the specification | Through the agreed training review | Approves specification changes |

All sample records are fictional. The trainer uses the same CLI output, target counts, and test evidence as the learner; there is no separate administrative feature.

[Back to README](../README.md)
