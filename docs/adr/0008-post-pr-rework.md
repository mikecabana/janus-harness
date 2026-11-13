# ADR 0008 — Manual rework after the PR is open

**Status:** Accepted 2026-11-13
**Affects:** §8.6, §9.1, §9.4

## Context

The spec covers the path up to an open PR and the terminal cases (merged, or closed without merging). It says "any rework updates must be explicit and auditable" and forbids force pushes (§8.6), but does not say what the factory does when a required check fails on the PR or the reviewer requests changes.

## Decision

1. **The factory observes.** After the PR is open it takes no action on its own.
2. **Blocking signals.** A failed required check, or a review that requests changes, moves the execution to `BLOCKED` with that reason and projects `agent:blocked` on the issue.
3. **Rework is an operator action.** A Rework button on the execution, with optional notes, exposed as `POST /api/v1/executions/:id/rework`.
4. **What a rework does.** It starts a new stage attempt in a fresh sandbox created from a per-execution checkout (ADR 0001) at the PR branch head. The prompt contains the operator's notes, the failing check names and conclusions, and review comments from users with write access.
5. **Delivery.** The result goes through the same collect, validate and push path. The push must be a fast-forward of the same branch. The PR is updated, never replaced.
6. **Limits.** Rework attempts count against the stage's `maxAttempts`. Past the limit, a rework needs an approval.
7. **Closed without merging.** The execution moves to `BLOCKED` and the operator chooses between re-queueing the issue as a new execution on a new branch, or cancelling.

## Consequences

- The agent never acts on CI output or review text without the operator choosing to send it.
- Review comments and check output become prompt input during a rework. They are treated as untrusted, like issue text.
- If the default branch has moved and the PR conflicts, the factory does not rebase. The operator resolves it or re-queues the issue.

## Not chosen

- **Automatic repair of failing checks.** More autonomy and more spend, and CI logs become an input the agent acts on unprompted. Deferred (see `docs/plan/05-later.md`).
- **Observe only, no rework path.** Simplest, but every review comment would cost a full new execution and a new PR.
