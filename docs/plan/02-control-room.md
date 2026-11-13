# Milestone 2 — Control room and observability

**Goal:** the operator can see what every agent is doing and act on it from the dashboard.

**Exit condition (§17 Milestone 2):** the operator can diagnose agent state and blockers without attaching to a sandbox. With this milestone the spec's MVP definition (§18.6) is met: live status, auth, cancellation, retry limits, and one restart scenario.

**Order:** M2.1 first. M2.2–M2.4 depend on it and are independent of each other. M2.5 → M2.6. M2.7 and M2.8 can land any time after M2.1.

---

## M2.1 — Durable event stream over SSE

**Build**

- `GET /api/v1/events/stream` (§11.4): `text/event-stream`, `id:` set to the durable event ID, `event:` set to the normalised type, heartbeat comments, filters by project and execution.
- Resume with `Last-Event-ID` or `?after=`: replay from Postgres, then switch to live.
- Live wake-up through Postgres `LISTEN/NOTIFY` carrying only the new event ID; the payload is always read from the table. Bounded per-client queues; a slow client is disconnected and resumes by cursor.
- The web app forwards the stream without buffering.
- Client hook that orders by event ID, drops duplicates, and treats events as a signal to refetch materialised state, not as the state itself (§11.5).

**Tests:** reconnect replays exactly the missed range (§18.2); duplicate and out-of-order delivery; slow-client disconnect; a test through the real Traefik route on the host confirming events arrive within a second and are not batched.

**Spec:** §11.4, §11.5, §20 risk 7.

---

## M2.2 — Overview and fleet

**Build**

- `GET /api/v1/fleet`: current agent instances, sandbox health, leases.
- Health for API, database, GitHub sync (last success, rate-limit headroom), runner (last heartbeat, versions) and the sandbox daemon (as reported by the runner).
- Overview page (§12.1 A): health, fleet counts, queue sizes, PRs awaiting review, recent failures, activity feed.
- Fleet page (§12.1 B): one card per agent **instance** with role, harness, model, project, issue, stage, last observed action, start time, last activity and sandbox health; filters; a dense table view when there are more than ten.
- A factory-wide pause switch, admin only, audited.
- Host resource figures (active microVMs, disk from `sbx df`) shown only when the runner reports them; otherwise "unavailable".

**Tests:** fleet query against seeded data; UX checks from §18.5 for "all active instances in one view" and filtering.

**Spec:** §12.1 A and B, §12.2, §18.5.

---

## M2.3 — Execution detail

**Build**

- Live timeline on the execution page, driven by M2.1: tool calls grouped with their results, search and filter, expandable sanitised output.
- Diagnostic stderr, artifacts (validation report, completion summary), test results, branch, SHA and PR link.
- The resolved configuration snapshot: agent file and hash, harness and CLI version, model, kit, workflow version, base SHA, and which repository instruction files were loaded (§5.6).
- Failure classification shown as a reason, not just a state: agent blocked, validation failed, infrastructure, GitHub, network policy, timeout, cancelled, lost.
- Retry history and recovery evidence.

**Tests:** timeline rendering from the recorded fixtures; classification table tests.

**Spec:** §12.1 D, §18.5.

---

## M2.4 — Projects view

**Build**

- Project page (§12.1 C): repository, default branch, enabled state, workflow, caps, synced issue queue with labels, running and blocked tasks, latest PRs with check status.
- Issue and PR details link out to GitHub; nothing is edited here.
- Read-only display of the project's effective configuration and the file and commit it came from ([ADR 0004](../adr/0004-config-source-of-truth.md)).
- Read-only registry endpoints and list pages: `GET /api/v1/agents`, `GET /api/v1/workflows`, and `GET /api/v1/settings` with secrets redacted. Editing arrives in M4.6.

**Spec:** §11.2, §12.1 C, F (read-only) and G (read-only).

---

## M2.5 — Operator actions, roles and audit

**Build**

- `POST /executions/:id/retry` with a required reason; bounded by policy; a retry beyond policy creates an approval request instead of running.
- Cancel end to end: intent recorded, the runner performs the M1.8 escalation, and the execution becomes `CANCELLED` only with termination evidence. If termination cannot be confirmed it becomes `RECOVERY_REQUIRED`.
- Approvals inbox (§12.1 E): `GET /approvals`, `POST /approvals/:id/decision` with actor, decision, reason and timestamp. An approval changes factory state only; it never merges anything.
- Roles `viewer`, `operator`, `admin` enforced on every mutation, even though one account exists ([ADR 0007](../adr/0007-dashboard-authentication.md)).
- Audit log entries for every mutation, with an audit page.

**Tests:** role matrix for every mutating endpoint; cancel never yields success; retry limit enforced; audit row written in the same transaction as the change.

**Spec:** §11.2, §12.1 E, §14.3 items 1 and 11, §18.5.

---

## M2.6 — Rework after the PR is open

Implements [ADR 0008](../adr/0008-post-pr-rework.md).

**Build**

- The reconciler detects a failed required check or a "changes requested" review on an open factory PR, moves the execution to `BLOCKED` with that reason, and projects `agent:blocked`.
- A Rework action on the execution page, with optional operator notes.
- Rework creates a new stage attempt: a per-run checkout at the PR branch head, a fresh sandbox, and a prompt containing the operator notes, the failing check names and conclusions, and review comments from users with write access.
- The result goes through the same collect, validate and push path. The push is fast-forward only onto the same branch; the PR is updated, never recreated.
- Rework attempts count against the stage's `maxAttempts`.
- A closed, unmerged PR moves the execution to `BLOCKED` with the choices "re-queue as a new execution" or "cancel" (§8.6).

**Tests:** check failure and review-requested detection with the fake GitHub; rework push is rejected if it is not a fast-forward; the attempt limit holds; the closed-unmerged path never reports completion.

**Spec:** §8.6, §9.4, ADR 0008.

---

## M2.7 — Blocked network requests

Implements the visibility half of [ADR 0002](../adr/0002-network-policy-model.md).

**Build**

- The runner watches the policy log and pending approvals for sandboxes it owns, using the forms recorded in M0.3, and emits an event per blocked destination.
- The execution page lists blocked hosts. A run that ends while requests were blocked is classified "network policy" with the host names.
- An admin action "allow this host for this project", which records an audit entry and tells the operator the exact file change to make in the project's config. The factory does not approve requests on its own.

**Tests:** parsing of recorded policy-log output; classification.

**Spec:** §7.2 network bullet, §14.3 item 6, ADR 0002.

---

## M2.8 — Usage, redaction and retention

**Build**

- `usage_reported` events stored in `budgets_usage` with a confidence marker; the UI shows tokens and cost only when the harness reported them (§12.1 D, ADR 0005).
- Redaction hardening: a corpus of secret-shaped strings run through ingest on every build.
- Retention jobs with configurable periods: raw tool output, events, artifacts, and stopped sandboxes kept for debugging after a failure.
- Export and erase for one execution's logs (§10.4).
- Raw harness streams stored as compressed files under the artifact directory, referenced by checksum.

**Tests:** redaction corpus; retention removes only what is past its period and never an artifact referenced by an active execution.

**Spec:** §10.4, §14.3 item 12, §18.1.

---

## Exit check

- Kill and restart the API during a run: the dashboard reconnects, replays, and shows the correct state.
- The §18.5 UX list passes, except the items that need a second harness or the registry (Milestone 4).
- The §18.6 MVP definition is checked line by line and the result recorded in the PR.
