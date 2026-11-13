# ADR 0009 — Implementation defaults

**Status:** Proposed. Accepted by merging the plan PR. Unlike ADRs 0001–0008, these items were **not** discussed one by one; each fills a gap the spec leaves open with the choice the implementer would otherwise make silently.

Object to any item in review and it will be changed before code depends on it.

## 1. Naming

- npm scope `@janus/*`; root package `janus-harness`.
- "Factory" stays as the domain word, as in the spec: sandbox names `factory-<project>-<issue>-<short-id>`, branches `factory/<project>/<issue>-<short-id>`, host paths under `/var/lib/factory`, UNIX account `factory`.
- Labels are the spec's `agent:*` set (§8.2).

## 2. First workflow

Milestones 1 and 2 use `implement → validate → deliver`, the reduced workflow the spec allows (§5.3). Plan and review stages arrive in Milestone 3.

## 3. What issue content reaches the agent

- The issue title and body, snapshotted when the execution is claimed (§8.5).
- Comments only from authors whose association with the repository is `OWNER`, `MEMBER` or `COLLABORATOR`.
- All of it is placed in a delimited block that the prompt describes as untrusted input.

Reason: on a public repository anyone can comment, and the spec treats issue comments as untrusted (§14.2).

## 4. Notifications

No separate channel. When a PR opens, the App requests a review from the operator's GitHub login, set per project, and GitHub notifies them. Blocked executions are visible in the dashboard and through the `agent:blocked` label.

## 5. Commit identity

Commits made in the sandbox are authored as the GitHub App's bot identity (`<app-slug>[bot]` with its `noreply` address), set through environment variables on `sbx exec`. Commits are not signed. A branch rule that requires signed commits would need a later decision.

## 6. Where validation commands live

In the factory's `config/validation-profiles.yaml`, never in the target repository. Reason: a file in the target repository could be edited by the agent in the same commit it is supposed to gate.

## 7. Changed-path deny list

The runner refuses to push a commit that changes `.github/workflows/**` unless the project's config allows it. Reason: workflow files run with the repository's CI credentials (§7.6 step 5, "risky paths").

## 8. Initial limits

Global concurrency 2; per-project 1; per sandbox 2 CPUs and 4 GiB; stage timeout 30 minutes; idle timeout 5 minutes; 3 attempts per stage. All configurable in `config/defaults.yaml`. They will be revisited against the host's real capacity in M0.3 probe 12.

## 9. Retention

Raw tool output 14 days; events 90 days; sandboxes of failed executions kept 24 hours when disk allows. All configurable (§10.4, §15.4).

## 10. Tooling

- Node version as found on the host and recorded in `docs/COMPATIBILITY.md`; pnpm via Corepack.
- Biome for lint and format; Vitest for tests; TypeScript project references.
- Postgres 17 in a container for integration tests.
- Playwright for the few browser tests.
- No Turborepo until build times call for it (§16.1).

## 11. Documentation layout

- The spec stays at `docs/SOFTWARE_FACTORY_ARCHITECTURE.md` and is not edited; ADRs record changes to it.
- `docs/plan/` holds this plan; `docs/adr/` the decision records; `docs/setup/` operator guides; `docs/COMPATIBILITY.md` the verified host and CLI behaviour.
