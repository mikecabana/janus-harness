# Milestone 0 — Foundations and compatibility spike

**Goal:** a buildable empty monorepo, and written evidence of how the installed `sbx` and Claude Code actually behave, before any adapter depends on that behaviour.

**Exit condition (§17 Milestone 0):** a runner-owned, unattended test task produces a commit in a disposable sandbox, and that commit is fetched into a factory-owned checkout without touching any interactive user's repository. `docs/COMPATIBILITY.md` records the verified behaviour and the installed versions.

**Where this runs:** on the host server. The implementing session is resumed there before M0.2 starts. M0.1 needs no host access and can be done from anywhere. Later milestones are built there too (see [Where the work runs](README.md#where-the-work-runs)).

**Order:** M0.1 → M0.2 → M0.3 → M0.4 → M0.5.

---

## M0.1 — Repository scaffold

**Build**

- pnpm workspace at the repo root: `package.json`, `pnpm-workspace.yaml`, `tsconfig.base.json`, `.gitignore`, `.nvmrc`/`engines` pinned to the Node version found on the host.
- The layout in §16.1, each as an empty package that builds: `apps/{web,api,runner}` and `packages/{core,database,github,agent-registry,harnesses,sandbox,events,validation,config,shared}`, with `packages/harnesses` holding `common`, `claude`, `codex`, `opencode`, `grok` subdirectories. npm scope is `@janus/*` ([ADR 0009](../adr/0009-implementation-defaults.md)).
- Root scripts from §16.3: `lint`, `typecheck`, `test`, `dev`, `db:migrate` (the last two are stubs until M1).
- Lint and format with one tool (Biome), Vitest for tests, TypeScript project references for incremental builds.
- GitHub Actions workflow running lint, typecheck and test on every PR.
- A dependency-boundary test: it reads each package's `package.json` and fails if `packages/core` declares anything outside `shared`, `events` and `config`, or if `apps/web` declares `database` or `runner`. pnpm's strict resolution makes an undeclared import a build error, so declared dependencies are the boundary.
- Empty `agents/`, `workflows/`, `config/`, `infra/`, `scripts/`, `tests/{fixtures,integration,e2e,recovery}`.

**Tests:** the boundary test; one trivial test per package so the runner is wired.

**Done when:** `pnpm install --frozen-lockfile && pnpm lint && pnpm typecheck && pnpm test` pass locally and in CI.

**Spec:** §16.1–16.3.

---

## M0.2 — Host audit

**Build**

- `scripts/doctor.sh`, read-only and idempotent. It reports, and never changes:
  - OS, kernel, architecture, KVM availability, CPU count, memory, free disk under `/var/lib`.
  - Docker Engine version; `sbx version`; `sbx diagnose --format json`.
  - Whether the current account is signed in to `sbx`, and which network preset is active (`sbx policy ls`).
  - Which secrets exist by service name only (`sbx secret ls`), and a **failure** if a `github` secret is present for the factory account ([ADR 0002](../adr/0002-network-policy-model.md)).
  - The Traefik Docker network and entrypoint names, and whether the runner loopback port (default 8787) is free.
  - Node and pnpm versions.
- Output as Markdown to stdout, so it can be redirected into `docs/COMPATIBILITY.md`.

**Operator steps documented here:** creating the `factory` account, `sbx login` with a Docker PAT, selecting the Balanced preset, and the one-time Claude sign-in for that account.

**Done when:** `doctor.sh` runs clean as the `factory` account and its output is committed as the first section of `docs/COMPATIBILITY.md`.

**Spec:** §13.1, §13.6 steps 1–3, §22 version gate.

---

## M0.3 — `sbx` behaviour probes

A script-driven probe suite, kept in the repo as `packages/sandbox/probe/` so it can be re-run after any `sbx` upgrade. Each probe records the exact command, exit code, and trimmed stdout/stderr. It works on a throwaway local Git repository it creates itself, and removes only sandboxes it created.

| # | Question | Why it matters |
|---|---|---|
| 1 | Does `sbx create --clone --name … --cpus … --memory … claude <path>` work from a checkout with a detached HEAD and no remotes? | ADR 0001 layout |
| 2 | What does the in-sandbox clone look like: path, current branch, remotes? Is a GitHub remote on the source copied, and a local-path remote not? | ADR 0001; "agent has no `origin`" |
| 3 | Exact JSON shape of `sbx ls --json`, `sbx inspect`, `sbx rm --json`, `sbx stop --json` | Adapter parsing |
| 4 | `sbx exec` exit codes: success, non-zero command, missing sandbox (125), missing executable (127) | Result classification |
| 5 | Do `-w`, `-e` and `--env-file` on `sbx exec` behave as documented? Can a prompt be delivered on stdin with `-i`? Does `sbx cp` work in both directions? | Prompt delivery without argv (§7.4) |
| 6 | When the host `sbx exec` process receives SIGTERM or SIGKILL, does the process inside the sandbox stop? What does `sbx stop` do to a running exec? | Cancellation (§7.4) |
| 7 | Does `git fetch sandbox-<name> <branch>` work while running, fail when stopped, and work again after restart? Is the remote removed by `sbx rm`? | Git handoff (§7.6) and cleanup gate |
| 8 | Stdout framing through `sbx exec`: are long lines delivered intact, is stderr separate, is there any prefixing? | NDJSON parser |
| 9 | What does a blocked outbound request look like inside the sandbox, in `sbx policy log --json`, and in `sbx policy approval ls`? What is the syntax for a sandbox-scoped allow rule? | ADR 0002 |
| 10 | Can the sandbox reach the host loopback API port or a LAN address under the Balanced preset? | §14.3 item 6 |
| 11 | Two creates at once from two per-run checkouts of the same repository: any interference? | Concurrency |
| 12 | Create time, idle memory, and disk per sandbox (`sbx df`) | Default caps |
| 13 | What does `sbx events --json` emit across create, exec, stop and rm? | Health and inventory |
| 14 | Does the sandbox survive a `sbx daemon restart` and a host reboot, and in what state? | Recovery (§15) |

**Done when:** every row has a recorded answer in `docs/COMPATIBILITY.md`, and the raw outputs behind rows 3, 4, 8, 9 and 13 are committed under `tests/fixtures/sbx/<version>/` with account identifiers removed. CI cannot run `sbx`, so these fixtures are what the fake adapter replays. A row whose answer contradicts an ADR stops the milestone until a superseding ADR is approved.

**Spec:** §7.1 VERIFY list, §17 Milestone 0, §20.

---

## M0.4 — Claude Code headless probes

Run inside a probe sandbox created in M0.3.

| # | Question | Why it matters |
|---|---|---|
| 1 | Does `claude -p --output-format stream-json --verbose` run unattended under the configured auth mode? What does an auth failure look like on stdout, stderr and exit code? | ADR 0005; fail-closed |
| 2 | Record complete streams for: a trivial answer, a file edit plus commit, a failing shell command, a timeout, a cancelled run | Fixtures for the normaliser (M1.9) |
| 3 | Does `--append-system-prompt-file` deliver the agent definition, and can the run prove which instructions it received? | §0 implementation rule; §18.3 last item |
| 4 | Shape of the final `result` event with `--json-schema`; behaviour when the model's output does not satisfy the schema | Structured completion summary (§7.4) |
| 5 | Do `--session-id` and `--resume` work across separate `sbx exec` calls in the same sandbox? | Repair loop and rework |
| 6 | Does `--max-budget-usd` have any effect under subscription auth? Is usage or cost reported in the stream? | Budgets (ADR 0005) |
| 7 | Which repository files does Claude Code load on its own (`CLAUDE.md`, `.claude/settings.json`, `.claude/agents/`), and how do `--setting-sources` and `--bare` change that? | §5.6: show what was loaded, avoid duplicating it |
| 8 | Does `--dangerously-skip-permissions` run without any prompt inside the sandbox? | Unattended execution |
| 9 | Claude Code version inside the kit image, and whether it can be pinned | §22 version gate |

**Done when:** recorded streams are committed under `tests/fixtures/claude/<cli-version>/`, with any secrets or account identifiers removed, and every row has an answer in `docs/COMPATIBILITY.md`.

**Spec:** §7.5, §6.6, §17 Milestone 0.

---

## M0.5 — End-to-end spike and write-up

**Build**

- One throwaway script that chains the verified commands: sync a bare mirror of the disposable repository, make a per-run checkout at a base SHA, create a clone-mode sandbox, run a headless Claude task that creates `factory/<project>/<n>-<id>` and commits, fetch that branch, verify its parent is the base SHA, remove the sandbox, remove the per-run checkout.
- It does **not** push or open a PR. That is M1.
- `docs/COMPATIBILITY.md` finalised: installed versions, probe answers, deviations, and a short list of the exact command forms the adapters will use.

**Done when:** the script runs unattended as the `factory` account, the fetched commit is present in the per-run checkout, and no other repository on the host was touched. This is the spec's Milestone 0 exit condition.

**Spec:** §17 Milestone 0, §21 item 2.
