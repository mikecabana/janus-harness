# Implementation plan

> **Status:** Proposed. Merging the PR that adds this directory approves the plan and the decision records in [`docs/adr/`](../adr/README.md). No application code exists yet.
> **Spec:** [`docs/SOFTWARE_FACTORY_ARCHITECTURE.md`](../SOFTWARE_FACTORY_ARCHITECTURE.md) (referred to below as "the spec", with section numbers as `§N`).
> **Written:** 2026-11-13

This plan turns the spec into an ordered set of work packages. Each work package is sized to be one reviewable pull request with its own tests and a concrete "done when".

## Contents

| File | Covers |
|---|---|
| [`00-foundations.md`](00-foundations.md) | Milestone 0: repository scaffold, host audit, `sbx` and Claude Code probes |
| [`01-vertical-slice.md`](01-vertical-slice.md) | Milestone 1: one labelled issue becomes one pull request |
| [`02-control-room.md`](02-control-room.md) | Milestone 2: live dashboard, operator actions, rework |
| [`03-workflow-engine.md`](03-workflow-engine.md) | Milestone 3: multi-project scheduling, plan/review stages, recovery |
| [`04-harnesses-portability.md`](04-harnesses-portability.md) | Milestone 4: Codex/OpenCode/Grok, agent registry, backup and restore |
| [`05-later.md`](05-later.md) | Milestone 5: outline only |
| [`traceability.md`](traceability.md) | Where each spec requirement, security control and acceptance test lands |
| [`PROGRESS.md`](PROGRESS.md) | What has been built, tested and demonstrated so far |

## Feasibility review

The spec was checked against the current Docker Sandboxes documentation and the Claude Code CLI (2.1.323) on 2026-11-13. The design is feasible and no DECIDED item needs to be reversed.

### Confirmed against upstream documentation

| Spec claim | Finding |
|---|---|
| `sbx create --clone` → `sbx exec` → `git fetch sandbox-<name>` → `sbx rm` (§7.1) | Documented by Docker as the CI pattern, in that form. |
| Headless Linux host (§13.1) | Supported. Without a Secret Service, `sbx` stores secrets in a `0700` directory under `~/.config/com.docker.sandboxes`. |
| Local sandbox compute cost | Free; a Docker account and `sbx login` (PAT via `--password-stdin`) are required. |
| Credentials stay outside the sandbox (§14.3 item 10) | The host proxy replaces a sentinel with the real credential after the request leaves the microVM. |
| Machine-readable CLI output (§7.4) | `--json`/`--format json` on `ls`, `rm`, `stop` and others; `sbx events --json` is NDJSON. `sbx exec` returns the command's exit status, 125 if it could not run it, 127 for a missing executable. |
| Resource limits (§3.4) | `--cpus` and `--memory` exist on `sbx create`. |
| Claude headless mode (§7.5) | `-p`, `--output-format stream-json`, `--verbose`, `--agent`, `--append-system-prompt-file`, `--json-schema`, `--session-id`, `--resume`, `--max-budget-usd`, `--setting-sources` are all present. |
| No built-in Grok kit (§6.6) | Built-in agents are claude, codex, copilot, cursor, devin, docker-agent, droid, kiro, opencode, shell. |

### Where the spec and upstream behaviour disagree

| # | Finding | Resolution |
|---|---|---|
| 1 | `sbx` writes a `sandbox-<name>` remote into the source repository and copies that repository's network remotes into the agent's clone. One shared checkout per repo (§7.6) therefore shares git config and a moving HEAD between concurrent executions, and hands the agent an `origin`. | [ADR 0001](../adr/0001-host-git-layout.md) |
| 2 | `SandboxSpec.allowedHosts` (§7.3) cannot be implemented. Local `sbx create` accepts only `--deny-network`; allow rules come from a machine-wide preset plus `sbx policy` rules. The Balanced preset allows code hosts, so "the agent cannot push" depends on never storing a `github` secret for the factory account. | [ADR 0002](../adr/0002-network-policy-model.md) |
| 3 | A "verifiably read-only" reviewer (§7.2, §14.4) depends on per-harness flags that differ and may not exist. | [ADR 0003](../adr/0003-exact-sha-validation-and-review.md) |
| 4 | Projects, workflows and agents exist both as files in Git (§5) and as DB tables with mutation endpoints (§10.1, §11.2), with no stated winner. | [ADR 0004](../adr/0004-config-source-of-truth.md) |
| 5 | Nothing says what happens after a PR is open if a required check fails or the reviewer requests changes (§8.6, §9.1). | [ADR 0008](../adr/0008-post-pr-rework.md) |
| 6 | Local sandboxes have no run timeout (`--ttl` is cloud-only) and default to 50% of host memory each. | No ADR: the spec already makes the runner responsible for timeouts (§7.4). The runner always passes explicit `--cpus`/`--memory` (M1.8). |

### Not yet verified

Anything that needs a live `sbx` run. The review was done from inside an `sbx` sandbox, which cannot run `sbx` itself (no binary, no `/dev/kvm`, no virtualization CPU flags). These items are the Milestone 0 probes in [`00-foundations.md`](00-foundations.md), and they run on the host before any adapter code is written.

## Decision log

Decisions made with the operator on 2026-11-13.

| Decision | Outcome | Record |
|---|---|---|
| Host Git layout | Bare mirror per repo plus a throwaway checkout per execution, pinned to the base SHA | [ADR 0001](../adr/0001-host-git-layout.md) |
| Who pushes to GitHub | The host runner, never the agent (confirms §7.6 and §14.3 item 3) | No ADR; spec unchanged |
| Network policy | Machine-level Balanced preset plus factory-managed rules; replaces `allowedHosts` | [ADR 0002](../adr/0002-network-policy-model.md) |
| Validation and review | Run against the exact committed SHA; review in a separate sandbox that is never fetched from | [ADR 0003](../adr/0003-exact-sha-validation-and-review.md) |
| Package layout | Ten packages exactly as §16.1 | No ADR; spec unchanged |
| Config source of truth | Files in Git; the DB holds a synced read-only index and immutable snapshots | [ADR 0004](../adr/0004-config-source-of-truth.md) |
| Claude authentication | Subscription OAuth by default, Anthropic API key as a config switch | [ADR 0005](../adr/0005-claude-authentication.md) |
| GitHub authentication | GitHub App in deployment; PAT provider for local smoke tests only | [ADR 0006](../adr/0006-github-authentication.md) |
| Dashboard login | App-local single operator; sessions owned by the API | [ADR 0007](../adr/0007-dashboard-authentication.md) |
| After the PR is open | Observe only; a manual Rework action adds commits to the same branch | [ADR 0008](../adr/0008-post-pr-rework.md) |
| Delivery | One branch and one PR when Milestones 0–4 are complete; no per-package PRs | [How implementation proceeds](#how-implementation-proceeds) |
| Where the work runs | The implementing session runs on the host for every milestone, for now | [Where the work runs](#where-the-work-runs) |

[ADR 0009](../adr/0009-implementation-defaults.md) collects smaller defaults that were **not** discussed one by one (naming, first workflow, which issue comments reach the prompt, notifications, commit identity, where validation commands live, a changed-path deny list, initial limits, retention, tooling). Review it with the same care as the rest; merging this PR approves it.

## Milestone map

```text
M0 Foundations ──► M1 Vertical slice ──► M2 Control room ──► M3 Workflow engine ──► M4 Harnesses + portability
   scaffold           issue → PR            live dashboard       multi-project           Codex / OpenCode / Grok
   host audit         Claude only           cancel/retry         plan + review           registry editor
   sbx probes         minimal UI            rework               recovery                backup / restore
```

| Milestone | Work packages | Exit condition |
|---|---|---|
| M0 | 5 | A runner-owned unattended task produces a commit in a disposable sandbox and it is fetched into a factory-owned checkout. `docs/COMPATIBILITY.md` records the verified `sbx` and Claude Code behaviour. |
| M1 | 14 | One issue labelled `agent:ready` becomes a PR with a stable execution ID and a validated SHA, with no duplicate dispatch and no automatic merge. |
| M2 | 8 | The operator can diagnose any execution, cancel, retry and rework it from the dashboard without attaching to a sandbox. This milestone completes the spec's MVP definition (§18.6). |
| M3 | 8 | Two repositories run concurrent isolated work with no cross-repo leaks and no double execution after a restart. |
| M4 | 8 | A new host can be deployed from Git plus a restored DB, and one workflow uses two different verified harnesses. |
| M5 | outline | Not scheduled. |

Work package IDs are `M<milestone>.<n>`. Within a milestone, the "Depends on" line of each package gives the order; packages with no dependency between them can be built in either order.

## How implementation proceeds

1. **One branch, one pull request.** All of Milestones 0–4 are built on `feat/factory-mvp`, and the PR is opened once they are complete. There are no intermediate PRs. Each work package is one commit, or a short series, whose message starts with its ID (`M1.3: ...`), so the final PR can still be read package by package. Milestone 5 stays an outline and is not part of that PR.
2. **Every work package is green before the next one starts:** `pnpm lint`, `pnpm typecheck`, `pnpm test`. Integration tests that need Postgres run against a container; tests that need a live `sbx` are opt-in and run on the host (see [Where the work runs](#where-the-work-runs)).
3. **Tests arrive with the code they cover** (§21 item 9). Parser fixtures come from real recorded streams captured in Milestone 0, not from hand-written guesses.
4. **Deviations become ADRs** (§21 item 12). If a probe contradicts an ADR or the spec, the work stops at that package, a new or superseding ADR is proposed, and it is approved before code depends on it.
5. **Nothing is claimed as working until it has been run.** [`PROGRESS.md`](PROGRESS.md) records, per work package, whether it is built, whether its tests pass, and whether its "done when" has been demonstrated on the host. The final PR description is written from it.

## Where the work runs

The implementing session runs on the host server for every milestone, for now.

- **Why:** the development sandbox cannot run `sbx` (no binary, no `/dev/kvm`, no virtualization CPU flags). The probes, the live adapter tests, deployment and every milestone's exit check need the host, and working in one place avoids handing results back and forth.
- **Accounts:** the session runs under the operator's own account. Commands that must run as `factory` (probes, the runner, smoke tests) use whatever access the operator grants for that, for example `sudo -u factory`. M0.2 records which.
- **Scope on the host:** the session touches only this repository, `/var/lib/factory`, `factory-` sandboxes and the disposable GitHub repository. Anything else on the server (Traefik config, system services, other repositories) is changed only with the operator's go-ahead at that step.
- **Fixtures are still recorded.** CI cannot run `sbx` either, so M0.3 commits raw `sbx` output for the fake adapter to replay.

Moving later milestones into a sandbox remains possible. The host-dependent "done when" steps would then become scripts the operator runs. It is not planned.

## Operator prerequisites

These need the operator, because they involve accounts, secrets or host privileges. Each is needed by the milestone shown and is documented step by step in the work package that consumes it.

| Needed by | Prerequisite |
|---|---|
| M0 | Resume the implementing session on the host server, with a way for it to run commands as `factory`. |
| M0 | A dedicated `factory` UNIX account with `sbx` installed, `sbx login` done with a Docker PAT, and the Balanced network preset selected. |
| M0 | One interactive Claude sign-in as the `factory` account (host-side OAuth), or an Anthropic API key stored with `sbx secret set anthropic`. |
| M0 | A disposable GitHub repository for smoke tests. |
| M1 | A private GitHub App created and installed on the disposable repository (steps in M1.6). |
| M1 | The name of the existing Traefik Docker network and entrypoint, and a LAN hostname for the dashboard. |
| M1 | Branch protection on the target repository requiring one approving review. |
| M2 | Nothing new. |
| M3 | A second repository with the GitHub App installed. |
| M4 | Credentials for any additional harness to be enabled (OpenAI for Codex, and so on). |

## Standing constraints

These come from the spec and hold for every work package.

- Existing Traefik only; no second reverse proxy. No Gas City, Plane, Redis, BullMQ or Temporal (§2).
- New runtime dependencies need a stated failure they solve (§21 item 5).
- `packages/core` imports no HTTP framework, GitHub SDK, `sbx` code or Drizzle schema (§16.2). The boundary is enforced by a test over each package's declared dependencies (M0.1).
- All host processes are started with `spawn(executable, args, { shell: false })` (§7.4). No issue text, label, branch or prompt is ever interpolated into a shell string.
- The agent never holds a GitHub write credential. The runner pushes one verified SHA to one constructed `refs/heads/factory/...` ref (§7.6).
- A sandbox is removed only after the cleanup gate in §15.4 passes.
- Uncertain state becomes `RECOVERY_REQUIRED`, never success (§9.1).
- Agents never merge or deploy (§0).

## Risks carried into implementation

| Risk | Where it is retired |
|---|---|
| Killing the host `sbx exec` process may not stop the process inside the sandbox | M0.3 probe; cancel escalation designed in M1.8 from the result |
| Clone mode may behave differently from a detached-HEAD per-run checkout (ADR 0001) | M0.3 probe; fallback is a named throwaway branch at the base SHA |
| Sandbox-scoped allow rules may not exist in the form ADR 0002 assumes | M0.3 probe; fallback is global rules derived from the union of enabled projects |
| Subscription OAuth may not work unattended under a service account, or may not report cost | M0.4 probe; API key path is the fallback (ADR 0005) |
| Claude stream-json event shapes change between CLI versions | Recorded fixtures plus a runner probe that fails clearly on an unknown shape (§22 version gate) |
| Several microVMs exhaust memory or disk on home hardware | Explicit per-sandbox limits in M1.8; disk-pressure admission in M3.1 |
| SSE is buffered by the Next.js proxy or Traefik | M2.1 tests through the real Traefik route |
