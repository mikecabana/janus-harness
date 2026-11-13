# Traceability

Where each part of the spec lands in the plan. Work package IDs refer to the milestone files in this directory. "Operator" means a prerequisite listed in the [README](README.md#operator-prerequisites).

## Minimal end-to-end acceptance target (§0)

| # | Target | Work packages |
|---|---|---|
| 1 | Register one GitHub repo | M1.1 (config schema), M1.4 (config sync), M4.6 (from the dashboard) |
| 2 | Label an issue `agent:ready` or queue it | M1.6, M1.7 |
| 3 | Claimed once, versioned workflow started | M1.4, M1.7 |
| 4 | Named isolated `sbx --clone` sandbox | M1.8, M1.11 |
| 5 | Agent reads instructions and issue, commits, emits events | M1.5, M1.9, M1.11 |
| 6 | Runner retrieves work, validates independently, PR opened with control-plane credentials | M1.10, M1.11, M1.6 |
| 7 | Dashboard shows state, tools, tests, links, blockers, actions | M1.12, M2.2, M2.3, M2.5 |
| 8 | Branch protection governs merge; factory observes and completes | Operator, M1.7 (reconciler), M3.4 |
| 9 | Restarts do not duplicate jobs or lose status | M1.14 (runner), M2 exit check (API), M3.5, M3.6 |

## Spec sections

| Section | Work packages |
|---|---|
| §3 Architecture and deployment boundaries | M1.7, M1.11, M1.13 |
| §5.1 Config precedence and snapshots | M1.1, M3.2 |
| §5.2 Project registry | M1.1, M1.4, M4.6, M3.10 (`protectedPaths`), M3.12 (`pathSurfaces`) |
| §5.3 Workflow definition | M1.1, M1.3 (linear), M3.2 (full), M3.12 (risk-based routing) |
| §5.4–5.6 Agent definitions | M1.5, M2.3 (loaded-instructions display) |
| §6.1–6.3 Compatibility, import, export | M4.2, M4.3, M4.4, M4.5 |
| §6.4–6.5 Types and resolution | M1.9 (Claude only), M4.1 (full) |
| §6.6 Harness order and gates | M0.4, M4.2–M4.4 |
| §7.1–7.2 Sandbox strategy and isolation | M0.3, M1.8, ADR 0001, ADR 0002 |
| §7.3 Adapter contracts | M1.8, M1.9 |
| §7.4 Process management | M1.8 |
| §7.5 Headless patterns | M0.4, M1.9, M4.2–M4.4 |
| §7.6 Git handoff | M1.11, ADR 0001 |
| §8.1–8.3 Source of truth, labels, polling | M1.6, M3.4 |
| §8.4 GitHub App | M1.6, ADR 0006 |
| §8.5 Discovery and duplicates | M1.2, M1.4, M3.4 |
| §8.6 PR delivery | M1.6, M2.6, M3.4 (base freshness), ADR 0008 |
| §8.7 Planning pipeline | M5 |
| §8.8 Triage stage | M3.11, ADR 0010 |
| §9.1–9.2 States and stage results | M1.3, M3.8, M3.9 (result envelope, decision requests) |
| §9.3 Scheduling | M1.4, M1.7, M3.1 |
| §9.4 Retry | M1.10, M2.5, M3.2 |
| §9.5 Parallelism | M3.2 (sequential stages), M5 (fan-out) |
| §9.6 Invariants | M1.3, M1.4, M3.10 (transition evidence) |
| §10.1–10.2 Tables and constraints | M1.2 |
| §10.3 Event model | M1.1, M1.9 |
| §10.4 Artifacts and storage | M1.10, M2.8 |
| §11.1 API rules | M1.7 |
| §11.2 Browser endpoints | See the table below |
| §11.3 Runner endpoints | M1.7, M1.11 |
| §11.4–11.5 SSE | M2.1 |
| §12 Dashboard | M1.12, M2.2–M2.7, M3.9 (decision requests), M4.6 |
| §12.3 Interactive terminal | M5 |
| §13 Deployment | M1.13, M4.7 |
| §14 Security | See the table below |
| §15.1–15.4 Recovery and cleanup | M1.11 (gate, inventory), M3.5, M3.6 |
| §15.5 Backup and restore | M4.8 |
| §16 Repository structure | M0.1 |
| §17 Milestones | The milestone files, one to one |
| §18 Verification | See the table below |
| §19 Remote access to an interactive sandbox | M1.13 (documentation only) |
| §20 Open choices and risks | README risks table, M0.3, M0.4 |
| §22 Version gate | M0.2, M0.5, M4.7 |

## Browser endpoints (§11.2)

| Endpoint | Work package |
|---|---|
| `GET /health` | M1.7 |
| `GET /projects` | M1.7 |
| `POST /projects` | M4.6 (ADR 0004) |
| `PATCH /projects/:id` | M1.7 (enable, pause), M4.6 (other safe settings) |
| `GET /projects/:id/tasks` | M1.7 |
| `POST /tasks/:id/queue` | M1.7 |
| `GET /executions`, `GET /executions/:id` | M1.7 |
| `POST /executions/:id/cancel` | M1.7, M2.5 (full escalation) |
| `POST /executions/:id/retry` | M2.5 |
| `POST /executions/:id/rework` (new, ADR 0008) | M2.6 |
| `GET /executions/:id/events` | M1.7 |
| `GET /events/stream` | M2.1 |
| `GET /agents`, `GET /workflows`, `GET /settings` | M2.4 |
| `POST /agents/import`, `GET /agents/:id/compatibility` | M4.5 |
| `GET /harnesses` | M4.1 |
| `GET /approvals`, `POST /approvals/:id/decision` | M2.5 |
| `GET /fleet` | M2.2 |

## Mandatory security controls (§14.3)

| # | Control | Where it is implemented | Where it is tested |
|---|---|---|---|
| 1 | UI authentication, CSRF, roles | M1.7, M2.5, ADR 0007 | M1.7, M2.5 |
| 2 | Repository allowlist | M1.1, M1.6 | M3.7 |
| 3 | No broad agent GitHub token | M0.2 (doctor fails on a `github` secret), M1.11, ADR 0001, ADR 0002 | M1.11, M3.7 |
| 4 | GitHub branch protection | Operator; M3.4 reads required checks and warns when branches need not be up to date | M1.14 (nothing merged), M3.4 |
| 5 | Per-stage capability enforcement | M3.3 (review by discard), M4.1 (unsupported requirement blocks) | M3.3, M4.1 |
| 6 | Network restrictions | ADR 0002, M0.3 probes 9 and 10, M2.7 | M3.7 |
| 7 | Resource and budget caps | M1.8 (CPU, memory, timeouts), M1.7 (concurrency), M3.1 (budgets) | M1.8, M3.1 |
| 8 | Prompt-injection resistance | M1.5 (untrusted block, comment filter), M1.11 (closed job enum, commit verification) | M1.5, M1.11 |
| 9 | No host shell injection | M1.1 (sanitisers), M1.8 (argv-only spawn), M1.11 | M1.1, M1.8 |
| 10 | Secret handling | M1.6 (tokens by environment only), M1.11, M2.8 (redaction) | M2.8 |
| 11 | Audit trail | M1.7, M2.5, M4.6 | M2.5 |
| 12 | Redaction and retention | M1.1, M2.8 | M2.8 |
| 13 | Server egress | M1.13 (Compose networks) | M1.13 |
| 14 | Credential rotation | M4.7 | M4.7 |

Control 5 has a known limit: `capabilities` in an agent file are recorded and displayed from Milestone 1 but are not enforced as tool restrictions. Enforcement comes from what the sandbox can reach and what the runner will accept. The spec says the same (§5.4).

## Verification (§18)

| Section | Work packages |
|---|---|
| §18.1 Unit tests | M1.1, M1.3, M1.5, M1.8, M1.9, M2.1, M2.8, M3.1, M3.2, M3.9, M3.10, M3.12, M4.1 |
| §18.2 Database and API integration | M1.4, M1.6, M1.7, M2.1 |
| §18.3 Sandbox and Git end to end | M1.11, M1.14, M3.3, M3.7, M3.11, M4.2 |
| §18.4 Failure injection | M1.14 (runner kill), M3.6 |
| §18.5 UX acceptance | M2.2, M2.3, M2.5, M4.5, M4.6, M1.13 (LAN reachability) |
| §18.6 MVP definition of done | M2 exit check |
