# Milestone 5 — Later

Outline only. Nothing here is scheduled, and each item needs its own design and approval before work starts. The order is the spec's priority order (§17 Milestone 5).

| # | Item | What it would need first |
|---|---|---|
| 1 | Read-only planning agent that proposes issues for human approval | A rule that planning output can never trigger its own execution (§8.7); an approval step before any issue is created |
| 2 | DAG fan-out and parallel subtasks with an integration coordinator | A separate sandbox or worktree per subtask (§9.5); a merge-conflict policy; a parent execution model |
| 3 | Automatic agent or model selection from cost and quality signals | Reliable usage data from M2.8; a policy that forbids silent moves to a more expensive provider (§9.4) |
| 4 | Local or hosted model routing | The experimental `sbx run --model` path verified per harness (§6.6) |
| 5 | GitHub Projects v2 status projection | App permission review; GraphQL sync as a projection only (§8.1) |
| 6 | In-browser terminal | A separately authenticated, audited PTY gateway (§12.3); until then, SSH and `tmux` as in §19 |
| 7 | Remote runners and other sandbox providers | Runner identity per host; the existing adapter interfaces should already allow it |
| 8 | Multiple operators and project-level roles | Building on the role column from ADR 0007 |
| 9 | Host resource scheduling and cost analytics | Measured data from M3.1 |

Smaller items deferred from earlier milestones:

- A fresh-sandbox option for validation, for projects that want validation isolated from anything the agent installed (noted in ADR 0003).
- A read-only GitHub token scoped to one sandbox, for projects with private dependencies (noted in ADR 0002).
- Optional automatic repair of failing CI checks (the alternative not chosen in ADR 0008).
- Traefik forward-auth as a second login path (the alternative not chosen in ADR 0007).
- A notification channel beyond GitHub's own (ADR 0009).
- Experimental `sbxenv.yaml` environment files in place of CLI flags (§7.1).
