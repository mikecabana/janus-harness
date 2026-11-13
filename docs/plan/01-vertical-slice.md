# Milestone 1 — One issue becomes one pull request

**Goal:** the smallest complete path through every layer, with Claude Code as the only harness and a deliberately plain UI.

**Exit condition (§17 Milestone 1):** one explicitly labelled issue becomes a PR with a stable execution ID and a validated Git SHA, with no duplicate dispatch and no automatic merge.

**Workflow in this milestone:** `implement → validate → deliver` ([ADR 0009](../adr/0009-implementation-defaults.md)). Plan and review stages arrive in Milestone 3.

## Order

```text
M1.1 shared/config/events ─┬─► M1.2 database ─► M1.3 core ─► M1.4 queue ─► M1.7 api ─► M1.12 web ─┐
                           ├─► M1.5 agent-registry ───────────────────────────┐                   │
                           ├─► M1.6 github ───────────────────────────────────┤                   ├─► M1.13 deploy ─► M1.14 smoke
                           ├─► M1.8 sandbox ─┬─► M1.10 validation ────────────┼─► M1.11 runner ───┘
                           └─► M1.9 claude ──┘                                │
```

## The path being built

1. The scheduler polls GitHub, finds an issue labelled `agent:ready`, and in one transaction writes an execution and its first job.
2. The runner claims the job with a lease and a fencing token.
3. `prepare_workspace`: sync the bare mirror, create a per-run checkout at the base SHA, create a clone-mode sandbox from it ([ADR 0001](../adr/0001-host-git-layout.md)).
4. `run_agent_stage`: run Claude Code headlessly with the compiled agent definition; stream normalised events to the API.
5. `collect_commit`: fetch the expected branch from the sandbox, verify it, record the SHA.
6. `run_validation`: run the project's validation profile inside the sandbox against a clean checkout of that SHA ([ADR 0003](../adr/0003-exact-sha-validation-and-review.md)).
7. `push_branch`: push that exact SHA to `refs/heads/factory/<project>/<issue>-<short-id>` with a short-lived, single-repository token supplied by the API for this job only.
8. The API finds or creates the PR, projects labels, and moves the execution to `PR_OPEN`.
9. `cleanup_sandbox` runs once the cleanup gate (§15.4) passes.
10. The reconciler observes merge and required checks, then marks the execution `COMPLETED`.

Job kinds are a closed enum. The runner accepts nothing else (§14.2).

---

## M1.1 — Shared contracts, config schemas, event schema

**Build**

- `packages/shared`: ID types and generators (`ex_`, `st_`, `ag_`, `job_` prefixes), DTOs used by web/api/runner, and sanitisers for sandbox names (2–63 characters, `sbx` naming rules), branch names and repository slugs.
- `packages/config`: Zod schemas and loaders for `config/projects.yaml`, `config/defaults.yaml`, `config/models.yaml`, `config/validation-profiles.yaml` and `workflows/*.yaml`; file checksums; the resolved immutable **execution configuration snapshot** type (§5.1); precedence merge (§5.1) in which security policy cannot be overridden by a lower layer.
- `packages/events`: the `FactoryEvent` envelope (§10.3), per-type payload schemas, payload size bounds, and the redaction pass (token patterns, `Authorization` headers, `.env`-style assignments).
- Validation-profile schema: `setup` commands and named `checks`, each an argv array with a timeout. No shell strings.

**Tests:** schema acceptance and rejection tables; sanitiser property tests (never empty, never exceeds limits, never contains characters outside the allowed set, distinct inputs that collide are disambiguated by ID suffix); redaction fixtures; precedence tests showing a project cannot raise a factory cap.

**Done when:** the sample files in `config/`, `workflows/` and `agents/` load, and a malformed file produces an error that names the file and field.

**Spec:** §5.1–5.3, §10.3, §10.4, §18.1.

---

## M1.2 — Database schema

**Build**

- `packages/database` with Drizzle: `projects`, `github_tasks`, `workflows`, `agent_definitions`, `executions`, `stage_runs`, `agent_instances`, `sandbox_instances`, `jobs`, `events`, `artifacts`, `github_links`, `github_outbox`, `audit_log`, plus `operators` and `sessions` for login ([ADR 0007](../adr/0007-dashboard-authentication.md)). `approvals` and `budgets_usage` are created now and used from Milestone 2.
- Every constraint in §10.2: unique task per project and issue node ID; unique execution idempotency key; a partial unique index for one active execution per task, with the active states listed in the migration; unique sandbox name; unique branch ref per repository; unique job dedupe key; monotonic `events.id` with `(execution_id, id)` and `(project_id, id)` indexes.
- `projects`, `workflows` and `agent_definitions` are an index of the files, written only by the config sync ([ADR 0004](../adr/0004-config-source-of-truth.md)).
- Migration runner, and a startup check that refuses to run against a schema version it does not know (§16.2).
- A test helper that starts Postgres in a container and gives each test file its own database.

**Tests:** each uniqueness rule is violated on purpose and must fail; migrations apply to an empty database and are idempotent.

**Done when:** `pnpm db:migrate` builds the schema from nothing and the constraint tests pass.

**Spec:** §10.1, §10.2.

---

## M1.3 — Core state machine

**Build**

- `packages/core`, pure TypeScript: execution states and stage states as separate machines (§9.1), stage results (§9.2), an explicit transition table, and a `transition()` function that returns the new state plus the events to append, or a typed rejection.
- The invariants in §9.6 as checks inside `transition()`.
- Workflow validation for this milestone: a linear chain of known stage kinds. Cycles and fan-out are rejected with a clear message until Milestone 3.
- Ports as interfaces only: `ExecutionStore`, `JobQueue`, `EventLog`, `GitHubPort`, `Clock`, `IdSource`.

**Tests:** every legal transition; every illegal pair rejected; property test that no sequence of inputs reaches `COMPLETED` without a merged-PR reference; property test that every state change yields at least one event.

**Done when:** the core package has no runtime dependency outside `shared`, `events` and `config`, and the boundary test from M0.1 passes.

**Spec:** §9.1, §9.2, §9.6, §16.2.

---

## M1.4 — Job queue, leases and event append

**Build**

- Postgres implementations of the M1.3 ports in `packages/database`.
- Claim with `SELECT … FOR UPDATE SKIP LOCKED`; lease expiry; a fencing generation that increases on every claim; heartbeat extension; dedupe keys.
- Terminal writes are accepted only from the current lease holder with the current fencing generation.
- State change and event append in the same transaction (§9.6 item 8).
- Lease expiry does **not** requeue. It marks the job `lost` and the execution `RECOVERY_REQUIRED` (§9.3). Automatic reconciliation arrives in M3.5.
- Event batch ingest with a per-job sequence number, so a repeated batch is a no-op.
- Config sync: load files, upsert the index tables with checksums, record a version when a checksum changes.

**Tests (real Postgres):** two concurrent claimers never get the same job; a stale fencing generation cannot write a result; a duplicate event batch changes nothing; an expired lease does not produce a second active job; two concurrent schedulers cannot create two active executions for one task.

**Done when:** the §18.2 concurrency tests that concern the queue pass.

**Spec:** §9.3, §10.2, §18.2.

---

## M1.5 — Agent registry and prompt bundle

**Build**

- `packages/agent-registry`: parse Markdown with YAML frontmatter under size limits, validate the portable core and the `factory` namespace (§5.4, §5.5), compute the source hash, and report unknown keys as warnings.
- The **prompt bundle** compiler (§6.2): agent instructions, stage context, issue title and body, the filtered comments ([ADR 0009](../adr/0009-implementation-defaults.md)), branch name to create, and the completion contract. Issue content is placed in a clearly delimited block and described as untrusted input.
- The completion contract as a JSON Schema: summary, files changed, tests run, assumptions, unresolved risks, and a `status` of `done` or `blocked` with a reason.
- `agents/senior-developer.md`, `workflows/mvp.yaml`, and sample `config/*.yaml`.

**Tests:** frontmatter fixtures (valid, malformed, oversized, duplicate names); the body is preserved byte for byte; the bundle hash is stable for identical inputs and changes when any input changes.

**Done when:** the bundle for a sample issue can be rendered to a file and its hash is stored in the execution snapshot.

**Spec:** §5.4–5.6, §6.2, §6.4, §6.5 steps 1 and 6.

---

## M1.6 — GitHub integration

**Build**

- `packages/github` behind `GitHubPort`, with two auth providers ([ADR 0006](../adr/0006-github-authentication.md)): GitHub App installation tokens, and a PAT provider that refuses to start when `NODE_ENV=production`.
- Token minting narrowed per request to one repository and the permissions the operation needs. A separate `mintPushToken(repo)` returns a contents-write token for one repository, used only for M1.11's push job.
- Issue discovery (§8.3): poll by label and `since` with an overlap window, pagination, ETags, PR records filtered out, jittered interval, high-water mark stored per project.
- Label projection (§8.2) that adds and removes only `agent:*` labels.
- PR find-or-create keyed by head branch and the stored mapping (§8.6), with the reviewer request from ADR 0009.
- Observation of PR state, merge, reviews and required checks.
- Rate-limit and secondary-limit handling with backoff; typed errors that separate "permission denied" from "transient".
- The outbox worker contract: every write to GitHub is an idempotent outbox row first.
- `docs/setup/github-app.md`: creating the App, the permission set actually used by the calls above, installing it on selected repositories.

**Tests:** an in-memory fake of `GitHubPort` for other packages; request-level tests for pagination, ETag 304, rate-limit backoff, PR adoption after a simulated timeout, and permission denial surfacing as a blocking, actionable error.

**Done when:** against the disposable repository, the package can list labelled issues, swap labels, and create then re-find one PR without duplicating it.

**Spec:** §8.1–8.6, §18.2.

---

## M1.7 — API and scheduler

**Build**

- `apps/api` on Hono. Controllers call services; services depend on the M1.3 ports.
- Operator auth (ADR 0007): login, logout, signed expiring session cookie (`HttpOnly`, `Secure`, `SameSite=Strict`), CSRF token on mutations, login rate limiting, an `operators` row seeded from a bootstrap password hash.
- Browser endpoints needed now (§11.2): `health`, `projects` (GET; PATCH limited to enable and pause, per ADR 0004), `projects/:id/tasks`, `tasks/:id/queue`, `executions`, `executions/:id`, `executions/:id/events` (paged), `executions/:id/cancel`.
- Runner endpoints (§11.3) on the same process but a separate router: `register`, `claim`, `heartbeat`, `events`, `result`, `observations`. Authenticated by the runner token plus job ID and fencing generation. Unreachable from the browser route.
- Loops started as modules, not request handlers (§3.2): GitHub discovery, scheduler (admission with a global cap and a per-project cap), outbox worker, PR reconciler.
- The job-chaining service: on each job result, call `core.transition()` and enqueue the next job in the same transaction.
- Request IDs, body-size limits, Zod validation on every input, UTC everywhere (§11.1).
- Audit log rows for queue and cancel.

**Tests:** session, CSRF and runner-token rejection; a full execution driven through the API with the fake GitHub and a scripted fake runner; the cancel request sets intent and does not claim success.

**Done when:** with fakes in place of GitHub and the runner, an issue goes from discovered to `PR_OPEN` to `COMPLETED` through the real HTTP surface and a real database.

**Spec:** §9.3, §11.1–11.3, §14.3 items 1 and 11.

---

## M1.8 — Sandbox adapter and process manager

**Build**

- `packages/sandbox`: `SandboxAdapter` (§7.3) implemented over the `sbx` CLI using the command forms recorded in `docs/COMPATIBILITY.md`, with `allowedHosts` replaced as described in [ADR 0002](../adr/0002-network-policy-model.md).
- `create` always passes `--clone`, `--name`, `--cpus`, `--memory` and `--no-progress`. `listOwned` filters by the `factory-` name prefix and never touches other sandboxes (§15.3).
- A process manager: argv-only spawn, stdout as a byte stream split into complete lines with a line-size cap and a malformed-line counter, stderr captured separately and bounded, backpressure, exit code and signal recorded.
- Independent timeouts: overall, idle (no output), and per Git operation.
- Cancel escalation designed from the M0.3 probe: graceful signal, then `sbx stop`, then a recorded "could not confirm termination" result that maps to `lost`.
- `FakeSandboxAdapter` that replays recorded streams and can simulate hangs, crashes and partial lines.

**Tests:** line splitting across chunk boundaries and UTF-8 boundaries, oversized lines, invalid JSON, interleaved stderr (§18.1); timeout and cancel paths against the fake. Live adapter tests are opt-in under `tests/e2e`.

**Done when:** the opt-in live test creates a sandbox, runs a command, and removes it on the host, and the fake passes the same contract test suite.

**Spec:** §7.1–7.4, §15.3.

---

## M1.9 — Claude harness adapter

**Build**

- `packages/harnesses/common`: `HarnessAdapter`, `HarnessCapabilities`, `AgentEvent`, and a shared contract test suite every adapter must pass.
- `packages/harnesses/claude`: `probe()` (version and capabilities), `compileAgent()` (writes the prompt bundle and system-prompt file), `buildLaunchCommand()` using the flags verified in M0.4, `normalizeEvent()` from stream-json to `tool_started`, `tool_finished`, `agent_message`, `usage_reported` and `process_exited`, and `buildResumeCommand()`.
- The agent definition is always injected explicitly. A file under `.claude/agents/` is never treated as evidence that the agent ran (§0).
- The prompt reaches the sandbox by file or stdin, never as an argument that could leak or be mis-parsed (§7.4).
- Unknown native event types are kept as diagnostics with a warning and never crash the run (§10.3).
- The native session ID is captured for resume.

**Tests:** the M0.4 fixtures replayed through the normaliser, including an unknown event type and a truncated stream; launch commands snapshot-tested; a test that the resolved instructions hash appears in the execution snapshot and in the launched command's inputs (§18.3 last item).

**Done when:** every recorded fixture normalises without loss of tool calls or results, and the probe fails with a clear message on an unrecognised stream shape.

**Spec:** §6.4–6.6, §7.3–7.5, §18.1.

---

## M1.10 — Validation

**Build**

- `packages/validation`: given a sandbox, a SHA and a validation profile, create a clean detached worktree of that SHA inside the sandbox, run `setup` then each check there, and return a structured report (per-check exit code, duration, bounded output).
- The report is stored as an artifact with a checksum.
- A failed check is `failed_retryable` while the stage has attempts left; the report is the structured input to the next implementation attempt (§9.4). In this milestone the retry is a fresh agent run in the same sandbox with the report attached.

**Tests:** against the fake sandbox: pass, fail, timeout, and a check that writes huge output.

**Done when:** a deliberately failing profile blocks the push in the M1.14 smoke test.

**Spec:** §5.3 (`kind: validation`), §9.2, §9.4, §14.2, ADR 0003.

---

## M1.11 — Runner

**Build**

- `apps/runner`: registration with version and capability report, job poller, heartbeat, batched event upload with sequence numbers and bounded local buffering.
- Handlers for the closed job enum: `prepare_workspace`, `run_agent_stage`, `collect_commit`, `run_validation`, `push_branch`, `cleanup_sandbox`.
- `GitArtifactService` (§7.6, ADR 0001):
  - mirror sync from an explicit URL with a read token passed through the environment, never written to Git config or argv;
  - per-run checkout creation and removal under `/var/lib/factory/runs/<execution-id>/`;
  - fetch of the one expected branch from the `sandbox-<name>` remote;
  - verification: ref name matches exactly, the base SHA is an ancestor, no merge commits unless policy allows, object integrity (`git fsck` on the fetched range), changed paths checked against a deny list (for example `.github/workflows/**` unless the project allows it);
  - push of the exact SHA to an explicitly constructed destination URL and ref, non-force.
- The cleanup gate (§15.4) as one function that returns the unmet conditions.
- Startup inventory: list owned sandboxes, post them to `/internal/runner/observations`, and claim no work until the API acknowledges (§13.4).
- `infra/factory-runner.service` (§13.4) and `docs/setup/runner.md`.

**Tests:** handlers against the fake sandbox and a local bare repository standing in for GitHub; verification rejects a wrong branch, a rewritten base, a merge commit and a denied path; the push never uses a remote name or URL taken from the sandbox; the cleanup gate refuses when a commit is unpushed.

**Done when:** on the host, the runner takes a scripted job sequence from the real API and leaves a pushed branch on the disposable repository.

**Spec:** §7.3, §7.4, §7.6, §13.4, §13.5, §15.3, §15.4.

---

## M1.12 — Minimal dashboard

**Build**

- `apps/web` on Next.js App Router with Tailwind and shadcn/ui.
- Same-origin `/api/*` forwarded to the API by rewrites; middleware redirects unauthenticated requests to the login page. The API remains the only place sessions are validated (ADR 0007).
- Pages: login; projects with enable and pause; task list per project with a Queue button; execution list; execution detail with state, stage rail, paged event list refreshed by polling, branch, SHA, PR link, failure reason, and Cancel.
- Unavailable data is shown as "unavailable", never estimated (§12.1 D).

**Tests:** component tests for the state badge and stage rail; one browser test for login, queue and cancel against the API with fakes.

**Done when:** an execution can be followed from queued to PR-open in the browser without reading logs.

**Spec:** §12.1 C and D (minimal), §17 Milestone 1.

---

## M1.13 — Deployment

**Build**

- `apps/web/Dockerfile`, `apps/api/Dockerfile`, `compose.yaml` and `.env.example` from §13.2–13.3, plus the pieces that skeleton omits: a one-shot migration service, API healthcheck, session secret, GitHub App key mounted as a file secret, pinned image versions.
- The runner port published on `127.0.0.1` only, with the port configurable.
- `infra/traefik-labels.md` describing how the labels map to the operator's existing Traefik.
- `scripts/migrate.sh`.
- `docs/setup/remote-debugging.md`: the SSH, `tmux` and `sbx exec` guidance from §19 for inspecting a sandbox by hand. Interactive sandboxes stay separate from factory-owned ones.

**Tests:** `docker compose config` validates; a compose-based test brings up `db` and `api` and checks health; a check that the `db` service publishes no ports and that `web` and `api` mount no Docker socket.

**Done when:** the stack starts behind the existing Traefik and the dashboard login is reachable from another LAN device.

**Spec:** §3.3, §13.1–13.5, §19.

---

## M1.14 — Live smoke test and exit check

**Build**

- `scripts/smoke-test.sh`: against the disposable repository, create an issue, label it `agent:ready`, wait for the PR, and assert: exactly one execution, exactly one PR, the PR head SHA equals the validated SHA, the labels moved from `agent:ready` to `agent:review`, nothing was merged.
- A second run that kills the runner during `run_agent_stage` and restarts it, asserting no second sandbox and no second execution.
- A third run with a failing validation profile, asserting nothing was pushed.

**Done when:** all three pass on the host. This is the Milestone 1 exit condition.

**Spec:** §0 acceptance target items 1–7, §18.3, §17 Milestone 1.
