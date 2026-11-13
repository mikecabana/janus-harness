# ADR 0006 — GitHub App in deployment, PAT for local tests

**Status:** Accepted 2026-11-13
**Affects:** §8.4

## Context

The spec calls for a private GitHub App (§8.4). A personal access token is quicker to set up, which matters for local smoke tests, but has costs in deployment:

- PRs opened with the operator's token are authored by the operator, who then cannot approve them under a required-review rule.
- The token is long-lived and not scoped per operation.

## Decision

1. `packages/github` exposes one `GitHubAuthProvider` interface with two implementations.
2. **`AppAuthProvider`** is the deployment path: installation tokens minted per operation, narrowed to one repository and the permissions that operation needs.
3. **`PatAuthProvider`** exists for local development and smoke tests. It refuses to start when `NODE_ENV=production`.
4. The App's private key lives only in the API process. The runner receives a single-repository, contents-write installation token inside the claim response for a `push_branch` job, and a read token for mirror sync. Tokens reach Git through the environment, never through argv or Git config.
5. The permission set is derived from the calls actually made and documented in `docs/setup/github-app.md` (M1.6).

## Consequences

- PRs are authored by the App's bot identity, so the operator can approve them.
- One-time setup of the App by the operator (a prerequisite for Milestone 1).
- Two auth paths to keep working; the shared interface and fake keep the cost small.

## Not chosen

- **App only.** One path, but local smoke tests would need the App installed on every scratch repository.
- **PAT first, App later.** Fastest start, but the self-approval problem appears as soon as branch protection is on.
