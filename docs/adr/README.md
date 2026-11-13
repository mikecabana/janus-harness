# Architecture decision records

The spec ([`SOFTWARE_FACTORY_ARCHITECTURE.md`](../SOFTWARE_FACTORY_ARCHITECTURE.md)) asks that deviations be recorded here instead of silently replacing accepted decisions (§21 item 12). Where an ADR and the spec disagree, the ADR wins for the sections it names.

| # | Title | Status | Spec sections affected |
|---|---|---|---|
| [0001](0001-host-git-layout.md) | Bare mirror plus a per-execution checkout | Accepted 2026-11-13 | §7.1, §7.2, §7.6, §13.5 |
| [0002](0002-network-policy-model.md) | Network policy is machine-level, not per `SandboxSpec` | Accepted 2026-11-13 | §7.2, §7.3, §14.3 item 6 |
| [0003](0003-exact-sha-validation-and-review.md) | Validate and review the exact committed SHA | Accepted 2026-11-13 | §3.4, §7.2, §14.4 |
| [0004](0004-config-source-of-truth.md) | Files are the source of truth for configuration | Accepted 2026-11-13 | §5.2, §10.1, §11.2, §12.1 F |
| [0005](0005-claude-authentication.md) | Claude subscription by default, API key optional | Accepted 2026-11-13 | §6.5, §14.5 |
| [0006](0006-github-authentication.md) | GitHub App in deployment, PAT for local tests | Accepted 2026-11-13 | §8.4 |
| [0007](0007-dashboard-authentication.md) | App-local single operator, sessions owned by the API | Accepted 2026-11-13 | §3.2, §11.1, §14.5 |
| [0008](0008-post-pr-rework.md) | Manual rework after the PR is open | Accepted 2026-11-13 | §8.6, §9.1, §9.4 |
| [0009](0009-implementation-defaults.md) | Implementation defaults | Proposed; accepted by merging the plan PR | several |

## Format

Each record has: **Status**, **Context**, **Decision**, **Consequences**, and where relevant **To verify** (the Milestone 0 probe that could overturn it) and **Not chosen**.

A record is never edited to change its decision. A new record supersedes it, and the old one's status line points to the new one.
