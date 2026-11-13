# ADR 0003 — Validate and review the exact committed SHA

**Status:** Accepted 2026-11-13
**Affects:** §3.4, §7.2 (review bullet), §14.4

## Context

The spec asks for review "in a separate read-only sandbox/checkout or a harness configuration with verifiable read-only behavior" (§7.2) and says a prompt that says "do not edit" is not enough (§14.4). Verifiable read-only behaviour differs per harness and may not exist for some.

The spec also runs validation in the execution's sandbox (§3.4) without saying which tree is validated. The agent's working tree can contain uncommitted files, so a validation that passes there says nothing certain about the commit that will be pushed.

## Decision

**Validation**

1. The runner first fetches the agent's branch and records the candidate SHA.
2. Validation runs inside the execution's sandbox, in a clean detached worktree of that SHA, using the project's validation profile (`setup`, then `checks`).
3. Only that SHA can be pushed. If anything changes the branch afterwards, the new SHA must be validated again.

**Review**

1. The review stage runs in its own sandbox, created from a per-execution checkout (ADR 0001) at the candidate SHA.
2. The runner takes one thing from that sandbox: the reviewer's findings, validated against a schema.
3. The runner never fetches Git objects from a review sandbox. Whatever the reviewer writes there is discarded when the sandbox is removed.

"Read-only" therefore means the reviewer's writes cannot reach the branch, whatever the harness allows inside the sandbox.

## Consequences

- No dependence on per-harness read-only flags, so review works the same way for every harness.
- Validation pays for a clean dependency install per SHA. This is the price of validating what will be pushed.
- One extra sandbox per review stage.
- Validation still runs in a VM the agent has had root in. A project that wants validation isolated from anything the agent installed can use a fresh sandbox later (see `docs/plan/05-later.md`). CI on GitHub remains the independent boundary, as the spec says (§7.6).

## Not chosen

**Harness or sandbox read-only enforcement** as worded in the spec. It needs verification per harness and blocks on any harness that cannot prove it.
