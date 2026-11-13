# Milestone 3 — Multi-project scheduling and the full workflow engine

**Goal:** several repositories, the complete `plan → implement → validate → review → deliver` workflow with a bounded repair loop, and recovery that has been tested by breaking things.

**Exit condition (§17 Milestone 3):** at least two repositories run concurrent isolated work without cross-repo credential or path leaks, and without double execution after a restart.

**Order:** M3.1 and M3.2 are independent. M3.3 needs M3.2. M3.4 needs M3.2. M3.5 is independent; M3.6 needs M3.5. M3.7 needs M3.1. M3.8 needs M3.2.

---

## M3.1 — Admission, fairness and budgets

**Build**

- Admission checks (§9.3): project enabled, global cap, per-project cap, per-harness and per-provider concurrency, and daily and weekly budgets.
- Budgets in the order the spec gives (§20): wall-clock time and attempts first; cost only when it was measured.
- Fair round-robin across projects so one busy repository cannot starve another; explicit operator queueing ranks above auto-discovery when configured.
- Disk-pressure admission: the runner reports `sbx df` and free space; below a threshold no new sandbox is created, and the dashboard raises an alert with a safe cleanup queue (§15.4).
- Backoff between attempts.

**Tests:** admission table tests; a fairness simulation with a skewed queue; budget exhaustion moves work to `BLOCKED` with a clear operator action (§9.4).

**Spec:** §3.4, §9.3, §9.4, §18.1.

---

## M3.2 — Workflow engine

**Build**

- Stage graphs with `dependsOn`: validated as a DAG, then executed sequentially within one execution (§9.5).
- Explicit repair-loop semantics, since a DAG does not describe cycles (§5.3): a stage may declare `onFailure: { returnTo: <stage>, maxAttempts: n }`. A failed validation or a review with blocking findings returns to the implementation stage with the structured findings attached, until the budget is spent.
- Retry strategy per attempt (§9.4): resume the known session in the known sandbox when the harness reported resume support and the sandbox is healthy; otherwise a fresh attempt from the last committed SHA.
- Artifact handoff between stages (the plan is an input to implementation).
- Workflow versioning: a changed file is a new version; running executions keep their snapshot (§5.1).
- `workflows/standard.yaml`, `bugfix.yaml`, `documentation.yaml`.

**Tests:** DAG validation (cycle, missing dependency, unknown agent); repair loop stops at the limit; a workflow edited mid-run does not change the running execution; transient infrastructure retries are counted separately from repair attempts.

**Spec:** §5.3, §9.4, §9.5, §18.1.

---

## M3.3 — Plan and review stages

**Build**

- `agents/software-architect.md`, `agents/code-reviewer.md`, `agents/qa-engineer.md`.
- Plan stage: the output is an `implementation-plan` artifact validated against a schema; optional plan-acceptance approval per workflow.
- Review stage as [ADR 0003](../adr/0003-exact-sha-validation-and-review.md) describes: a separate sandbox cloned from a per-run checkout at the candidate SHA; the runner takes only the schema-validated findings (severity, file, line, description, blocking or not) and never fetches Git objects from it; the sandbox is removed afterwards.
- Blocking findings feed the repair loop from M3.2.
- The PR body includes the review summary.

**Tests:** a reviewer that modifies files changes nothing in the delivered branch; malformed findings are a stage failure, not a pass; findings round-trip into the next implementation prompt.

**Spec:** §5.3, §7.2, §14.4, ADR 0003.

---

## M3.4 — GitHub lifecycle rules

**Build**

- Completion rules from the workflow's `completion` block: merged PR, required checks, human approval (§5.3).
- Required-check tracking that reads the branch protection or ruleset to learn which checks are required, and falls back to "all checks" with a visible warning when it cannot read them.
- Issue edited while running (§8.5): record the new version, and apply the project's policy, either continue with a visible warning or pause for a decision. The live prompt is never changed silently.
- Label removed or issue closed while running (§8.3): apply the configured cancel or block rule, and only after the change is seen on two consecutive successful polls.
- Periodic full reconciliation of every active task, PR and check, independent of `updated_at`.

**Tests:** each rule against the fake GitHub, including a transient API failure that must not cancel work.

**Spec:** §8.3, §8.5, §8.6.

---

## M3.5 — Recovery and reconciliation

**Build**

- The full algorithm in §15.2 on API start: load non-terminal executions and leases; pause dispatch until the runner has registered and reported its inventory, with a configurable grace period; reconcile GitHub; reconcile sandboxes and sessions; then decide per execution.
- Decisions, each with an event trail:
  - process alive and lease recoverable → resume monitoring;
  - process exited with the commit present → ingest and continue to the next stage;
  - process state unknown → `RECOVERY_REQUIRED`, no re-dispatch;
  - sandbox gone but commit or PR durable → continue from the last committed stage;
  - sandbox gone and nothing preserved → record lost work and ask for an explicit retry.
- Lease expiry now triggers this reconciliation instead of going straight to `RECOVERY_REQUIRED` (replacing the M1.4 behaviour). It still never launches a second copy.
- Orphan handling: sandboxes with the factory prefix that the database does not know are reported for inspection and never removed automatically; third-party sandboxes are never touched (§15.3).
- A safe garbage-collection queue that applies the cleanup gate.
- Operator actions on a `RECOVERY_REQUIRED` execution: probe again, resume, retry from clean, abandon.

**Tests:** one test per branch of the decision list above, using the fake sandbox and fake GitHub.

**Spec:** §9.3, §15.1–15.4.

---

## M3.6 — Failure-injection suite

**Build**

- `tests/recovery/`: kill and restart each of web, API, runner and a sandbox mid-stage; reboot the host where practical.
- Simulated GitHub rate limiting, an expired App token, a provider timeout, invalid JSON events, a full disk, malformed frontmatter, a dropped SSE connection, and a PR creation that succeeds on GitHub but times out locally.
- Assertions after every scenario: no duplicate active execution; the state is recovered or `RECOVERY_REQUIRED`; no unpushed branch was removed.

**Done when:** the suite is runnable with one command on the host and its results are recorded in the PR.

**Spec:** §18.4.

---

## M3.7 — Multi-project isolation

**Build**

- A second project enabled end to end.
- Tests that a push token minted for project A is rejected for project B's repository; that per-run paths are derived only from allowlisted project IDs; that an issue cannot redirect work to another repository (§5.2, §14.3 item 2).
- Two executions on the same project at once produce independent branches and share no working tree (§18.3).
- A sandbox cannot reach the runner API port or a LAN address (§18.3), using the M0.3 result as the baseline.

**Done when:** the Milestone 3 exit condition is demonstrated on the host with two repositories.

**Spec:** §14.2, §14.3 items 2, 3 and 6, §18.3.

---

## M3.8 — Approval gates

**Build**

- Policy-driven approval points (§12.1 E): plan acceptance, retry beyond policy, scope change reported by the agent, risky validation result.
- `requiresHumanApproval` on any stage (§5.3), with a deadline and an outcome on expiry.
- `AWAITING_APPROVAL` wired into the execution state machine.

**Tests:** a stage with an approval gate does not start until approved; denial moves to `BLOCKED` with the reason; every decision has an actor and a reason.

**Spec:** §5.3, §9.1, §12.1 E.
