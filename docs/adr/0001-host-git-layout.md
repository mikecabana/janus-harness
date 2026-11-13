# ADR 0001 — Bare mirror plus a per-execution checkout

**Status:** Accepted 2026-11-13
**Affects:** §7.1, §7.2, §7.6 steps 1–6, §13.5

## Context

The spec keeps "a clean, machine-owned local checkout per registered repo" and creates each clone-mode sandbox from it (§7.6 step 1).

Docker's documentation for clone mode says:

- `sbx` adds a `sandbox-<name>` remote to the **source** repository's Git config, updates its URL when the sandbox restarts, and removes it on `sbx rm`.
- `sbx` copies the source repository's network remotes, such as `origin`, into the in-sandbox clone. Local-path remotes are not copied.

With one checkout per repository this causes three problems:

1. Two concurrent executions on the same project (`maxConcurrentExecutions: 2` in the spec's own example) share one Git config and one object store on the host.
2. The checkout's HEAD moves every time the runner syncs the default branch, so "pinned to a known base SHA" (§7.6 step 2) is true only by timing.
3. The agent's clone inherits an `origin` that points at GitHub.

## Decision

The runner keeps two kinds of directory per project.

```text
/var/lib/factory/repos/<project>.git        bare mirror, long-lived
/var/lib/factory/runs/<execution-id>/src    per-execution checkout, short-lived
```

- **Mirror.** Fetched from GitHub by explicit URL before each execution. The token is supplied through the environment for that one command and is never written to Git config.
- **Per-execution checkout.** A local clone of the mirror, checked out detached at the execution's base SHA, with its `origin` remote removed. It is the path given to `sbx create --clone`.
- **Collecting work.** The runner fetches the one expected branch from `sandbox-<name>` into the per-execution checkout, verifies it (§7.6 step 5), and pushes the exact SHA from there to an explicitly constructed GitHub URL and ref.
- **Lifetime.** The per-execution checkout is removed with its sandbox, after the cleanup gate (§15.4) passes.
- **Rework and review** (ADR 0008, ADR 0003) create their own per-execution checkout at the SHA they need.

## Consequences

- Concurrent executions on one project share nothing writable on the host.
- The base SHA is fixed by construction.
- The agent's clone has no GitHub remote. Combined with ADR 0002, the agent has neither a destination nor a credential for a push.
- More disk churn: one working tree per execution. Local clones hard-link objects, so the cost is mostly the checked-out files.
- `/var/lib/factory/runs/` is added to the host directory layout in §13.5.

## To verify (M0.3 probes 1, 2, 7, 11)

- Clone mode accepts a source checkout with a detached HEAD and no remotes. Fallback: a throwaway local branch at the base SHA.
- A local-path remote really is left out of the in-sandbox clone.
- Two sandboxes created at once from two per-execution checkouts of the same mirror do not interfere.

## Not chosen

**One shared checkout per repository**, as written. Simpler on disk, but it has the three problems above.
