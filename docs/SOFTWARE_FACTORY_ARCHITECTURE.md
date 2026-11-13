# Software Factory — Architecture, Product Specification, and Implementation Handoff

> **Status:** Proposed architecture / implementation specification (not a claim that this system has been implemented).  
> **Last reviewed:** 2026-11-12  
> **Target:** Self-hosted, LAN-accessible, multi-project AI software development factory using GitHub, Docker Sandboxes (`sbx`), pluggable coding-agent harnesses, a custom TypeScript orchestrator, and a live visual dashboard.  
> **Audience:** Any software engineer or coding agent implementing the system from scratch.  
> **Document convention:** **DECIDED** = accepted architectural direction; **PROPOSED** = design choice to implement and validate; **VERIFY** = upstream behavior or infrastructure requirement to test against the installed version; **LATER** = intentionally outside MVP.

---

## Navigation

- [0. Executive summary and handoff instructions](#0-executive-summary-and-handoff-instructions)
- [1. Product vision and principles](#1-product-vision-and-principles)
- [2. Decisions and evolution of the design](#2-decisions-and-evolution-of-the-design)
- [3. Logical and physical architecture](#3-logical-and-physical-architecture)
- [4. Stack and architectural rationale](#4-stack-and-architectural-rationale)
- [5. Canonical configuration: projects, workflows, agents](#5-canonical-configuration-projects-workflows-agents)
- [6. Agent/harness compatibility and compilation](#6-agentharness-compatibility-and-compilation)
- [7. Docker Sandbox execution strategy and runner adapter](#7-docker-sandbox-execution-strategy-and-runner-adapter)
- [8. GitHub integration: work discovery, repositories, PRs, CI](#8-github-integration-work-discovery-repositories-prs-ci)
- [9. Orchestrator state machine and scheduler](#9-orchestrator-state-machine-and-scheduler)
- [10. Persistence and data model](#10-persistence-and-data-model)
- [11. HTTP APIs, runner protocol, and realtime streaming](#11-http-apis-runner-protocol-and-realtime-streaming)
- [12. Dashboard: information architecture and UX requirements](#12-dashboard-information-architecture-and-ux-requirements)
- [13. Compose deployment using existing Traefik](#13-compose-deployment-using-existing-traefik)
- [14. Security and authorization model](#14-security-and-authorization-model)
- [15. Failure recovery, reconciliation, cleanup, and backup](#15-failure-recovery-reconciliation-cleanup-and-backup)
- [16. Repository and module structure](#16-repository-and-module-structure)
- [17. Detailed implementation milestones](#17-detailed-implementation-milestones)
- [18. Verification, quality gates, and acceptance criteria](#18-verification-quality-gates-and-acceptance-criteria)
- [19. Remote access to an existing interactive Claude sandbox](#19-remote-access-to-an-existing-interactive-claude-sandbox)
- [20. Open implementation choices and risk register](#20-open-implementation-choices-and-risk-register)
- [21. Instructions for the next implementation agent](#21-instructions-for-the-next-implementation-agent)
- [22. Authoritative references and verification checklist](#22-authoritative-references-and-verification-checklist)
- [23. Final acceptance statement](#23-final-acceptance-statement)

---

## 0. Executive summary and handoff instructions

Build a portable software factory that accepts work from one or more GitHub repositories, dispatches controlled workflows to specialized coding **agents**, executes each stage inside a Docker Sandbox microVM through a **harness** (Claude Code, Codex, OpenCode, and eventually Grok Build), streams structured execution events to a custom dashboard, validates output, creates GitHub pull requests, and waits for human approval/merge. A task is only marked completed after the agreed merge/CI completion rules succeed.

Use the user's **existing Traefik instance** as the only reverse proxy: do **not** install Caddy. No Plane, Linear clone, Gas City, or another orchestrator. GitHub Issues is the default work backlog; GitHub Projects v2 is an optional presentation/planning layer, not a prerequisite. This project owns *execution*, not general project management.

The recommended stack is **TypeScript + Node.js**, **Next.js** (dashboard), **Hono** (thin API), a **framework-independent custom orchestrator**, **PostgreSQL + Drizzle**, **Server-Sent Events**, **Docker Compose** for application services, and a **host-side systemd runner** that invokes `sbx`. Fastify remains an acceptable alternative to Hono; there is no throughput-driven reason to adopt it. Start with a PostgreSQL-backed job queue/leases rather than introducing Redis, BullMQ, Temporal, or a message broker.

**The primary user-facing abstraction is an Agent, not a Model.** An *Agent Definition* is a reusable Markdown document with YAML frontmatter describing a specialist and its instructions. It may specify preferred model and harness, but stages reference an agent by ID. At execution, the factory resolves `stage -> agent definition -> harness -> model -> Docker Sandbox runtime`. An *Agent Instance* is a specific active or historical run of an agent definition. Harnesses consume compiled/injected agent definitions: native profile file support is optional and implementation-specific.

**Important implementation rule:** Code should not conflate a *harness-native subagent* with a *factory-controlled independent workflow execution*. A file in `.claude/agents/` or `.opencode/agents/` does not, by itself, guarantee the requested specialist has been invoked. The orchestrator must explicitly launch the selected role or inject its instructions into a stage session; native subagent activation is an adapter optimization, not a correctness assumption.

### Minimal end-to-end acceptance target

1. Register one GitHub repo in the dashboard/config.
2. Label an issue `agent:ready` (or explicitly queue it in the dashboard).
3. The scheduler claims the issue once and starts a versioned workflow.
4. The host runner creates a named, isolated `sbx --clone` sandbox using the selected harness.
5. The agent reads project instructions and the issue, performs a bounded implementation, commits changes, and emits parseable events.
6. The runner retrieves committed work, validates it independently, and opens a GitHub PR using control-plane credentials.
7. The dashboard shows agent state, tool activity, test outcomes, Git commit/PR links, resource usage, blockers, and actions to cancel/retry/review.
8. GitHub branch protection and human review govern merge; the system observes CI and merge, then completes the issue.
9. Restarting API, runner, or host does not silently duplicate jobs or lose durable status.

### Explicit non-goals for the initial release

- Do **not** build another general issue tracker or require Plane.
- Do **not** replace GitHub or host GitHub itself.
- Do **not** add Gas City or a heavyweight orchestration framework.
- Do **not** build an independent container runtime; rely on installed Docker `sbx`.
- Do **not** expose a publicly reachable inbound web endpoint or require public GitHub webhooks.
- Do **not** let coding agents merge protected branches or deploy unattended.
- Do **not** assume internal reasoning traces, true progress percentages, or precise costs are available from every harness.
- Do **not** require every coding harness to share identical agent-definition metadata.
- Do **not** make automatic, unconstrained issue creation/planning part of the MVP.

---

## 1. Product vision and principles

### 1.1 Product vision

A self-hosted **visual software factory** running on a home server: multiple GitHub projects, reusable agents and workflows, isolated concurrent executions, reliable task/PR lifecycle, and a control room showing the state of the whole system at a glance. It must be plug-and-play, portable to another compatible server, and understandable to users who do not want to watch terminals continuously.

### 1.2 Design principles

1. **Few dependencies.** Reuse GitHub, Docker Sandboxes, and existing Traefik. Everything else is a small self-owned application.
2. **GitHub is authoritative for work and code.** GitHub Issues/Projects and PRs; local DB stores execution state and history.
3. **Deterministic control plane, nondeterministic agents.** Claude/Codex/Grok decide how to solve tasks; the orchestrator alone decides dispatch, retries, ownership, approvals, and completion.
4. **Agents are reusable definitions.** Agent Markdown + YAML frontmatter is source controlled, with adapters for different harnesses.
5. **Harness ≠ model ≠ agent ≠ runtime.** They are separate concepts with separate versioned configurations.
6. **One execution, one isolation boundary.** Default one Docker Sandbox per independently working task/stage execution; no shared mutable working tree across active jobs.
7. **Fail closed.** If credentials, permissions, policy enforcement, or state recovery are uncertain, block and request human review instead of guessing.
8. **No public inbound network.** Poll GitHub over outbound HTTPS; LAN-only dashboard and internal APIs.
9. **Portable by configuration and backup.** Versioned agents/workflows/project registry, Compose for services, systemd for the host runner, reproducible kits and documented restore.
10. **Observable from day one.** Every transition and action emits durable events, and a dashboard can replay them.
11. **Incremental autonomy.** First deliver safe PRs; then add QA, retries, subtask dependency graphs, broader planning, and automated routing.

### 1.3 Terminology

| Term | Precise meaning |
|---|---|
| **Project** | A registered GitHub repository and project-specific configuration. May link to an optional GitHub Project v2 board. |
| **GitHub Task** | An indexed issue eligible for agent work; GitHub remains authoritative for its content/status. |
| **Workflow** | Versioned declarative pipeline of stages and transition rules. |
| **Stage** | A specific unit of work in a workflow (plan, implement, validate, review, deliver, etc.). |
| **Agent Definition** | Reusable named specialist with Markdown instructions and YAML frontmatter. |
| **Agent Instance** | One agent session/run created for an execution stage; many instances may use the same definition. |
| **Model** | Underlying model/endpoint and tuning parameters. |
| **Harness** | Coding-agent software/tool loop and CLI: Claude Code, Codex, OpenCode, Grok Build, etc. |
| **Runtime** | Environment isolating harness execution; MVP uses Docker Sandboxes on local host. |
| **Runner** | Trusted host-side Node service that controls `sbx`, Git checkout/fetch/push, and reports events. |
| **Execution** | An attempt to complete a GitHub Task with a specific workflow/config snapshot. |
| **Job** | Durable, leasable control-plane command consumed by the host runner. |
| **Artifact** | Plan, validation report, commit SHA, diff summary, review findings, PR URL, etc. |
| **Event** | Append-only timestamped fact about a project/execution/stage/agent. |

---

## 2. Decisions and evolution of the design

These are not unresolved suggestions; they preserve the conversation's architectural decisions and rejected alternatives.

| Area | Decision | Rejected / rationale |
|---|---|---|
| Orchestration | **Custom TypeScript orchestration harness** | No Gas City; avoid third-party orchestration and opaque workflows. |
| Project management | **GitHub Issues + PRs**, optional Projects v2 | No Plane unless a concrete missing GitHub capability later justifies it. |
| Reverse proxy | **Reuse existing Traefik** | Remove Caddy and do not bring up another reverse proxy. |
| Dashboard | **Custom Next.js dashboard** | It should visualize execution, not duplicate GitHub's planning UI. |
| Backend language | **TypeScript/Node.js** | Shared types, CLI process management, GitHub SDK, familiar ecosystem. |
| HTTP API | **Hono** (recommended), Fastify acceptable | Throughput isn't decisive; keep HTTP thin and framework-independent. |
| Persistence | **PostgreSQL + Drizzle** | SQLite viable for tiny demos but concurrency/recovery better served by Postgres. |
| Queue | **Postgres transactional claims + leases** | Avoid Redis/BullMQ/Temporal unless concrete scale/reliability need emerges. |
| Realtime | **SSE for dashboard**, REST for commands | WebSockets only if interactive terminal/bi-directional streaming becomes essential. |
| Execution | **Docker `sbx` microVM sandboxes** | Not ordinary Docker containers with host daemon socket. |
| Runner deployment | **Host-side Node runner via systemd** | `sbx` is a host service; don't mount privileged Docker/sbx sockets into web services. |
| Git workspace | **Clone-mode sandbox per execution** | Avoid direct mount/shared mutable worktrees. Consider mountless later. |
| Agent vendors | **Pluggable harness adapters** | Claude Code first; Codex second; OpenCode and Grok Build as supported backends. |
| Agent UX | **Agent-first** | Model-first was considered, but agent definitions better capture role, constraints, and tool use. |
| Agent format | **Markdown + YAML frontmatter** | Portable core, compilation/adaptation for differing native harness schemas. |
| Runtime role config | **Stages reference agent IDs** | Don't hardcode raw model or harness selections into every workflow stage. |
| Human approvals | **Required for merge/deploy** | No unattended privileged actions in MVP. |
| Network | **LAN only / outbound GitHub & model APIs** | No public ingress/tunnels needed. |
| GitHub sync | **Polling and reconciliation** | Public webhooks not required; optional private webhook relay later. |
| Portability | **Compose + host runner + Git-backed config + backup/restore** | Not “one Compose file does everything”; be honest about the host `sbx` dependency. |

**Prior art (PROPOSED; adopted by ADR 0010).** The patterns below are drawn from common practice in agent-driven issue-to-PR systems and the CI/CD and code-review tooling around them. Only ideas are adopted, no third-party code. Each pattern is adapted to this spec's sandboxed, human-merge model where it appears:

| Pattern | Section |
|---|---|
| Agent results lead with a closed-enum verdict and a short summary; blocked results carry one specific decision request | §9.2 |
| Every transition names the evidence it needs; the orchestrator refuses the move otherwise | §9.6 |
| Optional read-only triage stage: accept (possibly rescoped), ask the author, or propose rejection; rates risk and surfaces | §8.8 |
| Review stages and agent variants picked from a deterministic risk/surface table | §5.3 |
| Re-test against the current base before merge | §8.6, §14.3 |

---

## 3. Logical and physical architecture

### 3.1 Logical components

```text
Local browser / laptop (LAN)
           |
           | https://factory.home.arpa
           v
      Existing Traefik
           |
           v
   Next.js dashboard (web)
           |
           | authenticated HTTP; SSE through same-origin proxy
           v
   Hono API / Control Plane -----------------> GitHub REST / GraphQL
           |                                      ^
           |                                      | Issues, Projects v2,
           |                                      | PRs, checks / merge
           v                                      |
   PostgreSQL ------------------------------------+
     |       persistent tasks, executions, leases,
     |       events, configurations, artifacts
     |
     |  runner-only HTTP bound to host loopback
     v
   Host-side Node runner (systemd; trusted service account)
     |  - host Git cache / trusted Git operations
     |  - sbx CLI lifecycle and process management
     |  - normalize harness output to events
     v
   Docker sandbox daemon on host
     |            |            |
     v            v            v
 Claude sandbox  Codex sandbox  OpenCode / custom Grok sandbox
     |            |            |
     +------------+------------+
                  |
          Outbound model endpoints

No direct inbound Internet -> Traefik; no Docker socket in web/API.
```

### 3.2 Deployment responsibilities

**Docker Compose services**:
- `web`: Next.js dashboard with authentication/session management, same-origin API routing, and SSE presentation.
- `api`: Hono control API, orchestrator/scheduler, GitHub polling/reconciliation, runner job endpoint, and SSE event service. Keep long loops as separately started modules rather than HTTP handlers; could split into another process later without changing contracts.
- `db`: PostgreSQL, private Docker network and persistent volume.

**Pre-existing infrastructure**:
- Traefik, attached by an external Compose network. It must not be duplicated in this project.
- LAN DNS for a chosen internal domain, e.g. `factory.home.arpa`.
- TLS termination managed by the user's Traefik setup; trust/internal certificate setup is deployment-specific.

**Host process**:
- `runner` systemd service under dedicated `factory` UNIX account, using installed `sbx` and a restricted local Git cache.
- `sbx` microVM runtime and harness kits installed/authenticated for headless operation.

**External**:
- GitHub (repos, issues, optional Projects v2, PRs, CI).
- Anthropic/OpenAI/xAI or another model endpoint. This platform is self-hosted, but GitHub and cloud model APIs are not offline services.

### 3.3 Deployment boundaries

1. **Public route:** Traefik exposes `web` only on explicitly allowed LAN entrypoints. Apply firewall rules (IPv4 and IPv6), not just DNS obscurity.
2. **Application route:** Next.js server forwards authenticated `/api/*` requests to `api` on a private Compose network; no public API router is required.
3. **Runner route:** `api` listens on an additional host mapping `127.0.0.1:8787:3001` exclusively for the host runner, authenticated with a separate runner identity and job lease token. Restrict runner endpoints under `/internal/runner/*`.
4. **Database route:** PostgreSQL is *never* published to the host or Traefik; reachable only from `api` via a private DB network.
5. **Outbound route:** API requires outbound HTTPS for GitHub; sandboxes need explicitly allowed provider/registry/package endpoints through Docker Sandbox network policies.
6. **No socket forwarding:** Neither Next.js nor API gets the host Docker socket or `sbx` daemon socket.

### 3.4 Runtime resource model

- One *sandbox per independent execution* initially; stages may reuse that execution's sandbox for implementation/testing, while review ideally occurs from an independent read-only checkout or separate sandbox/session.
- Global and project-level caps for concurrent executing sandboxes. Start conservatively; each sandbox is a microVM and may run its own Docker daemon/images.
- Apply CPU/memory limits and run timeouts via supported `sbx` options/environment files. Disk quotas/cleanup must be monitored separately.
- Inactive sandboxes can be stopped for recovery, but removal is gated on durable Git/artifact persistence.
- One process may host multiple agent instances sequentially over time, but running agents and their individual session IDs remain distinct records.

---

## 4. Stack and architectural rationale

| Layer | Recommendation | Why | Notes |
|---|---|---|---|
| UI | Next.js (App Router), React, Tailwind, shadcn/ui | Flexible interactive control room and familiar stack | Server-rendered shell + client live views. |
| API | Hono on Node.js | Lightweight, typed, simple endpoints | Fastify is an equivalent alternative, not a blocker. |
| Business orchestration | Plain TypeScript library | Clear deterministic state machine, easily tested | No external orchestration SDK required. |
| Runner | Node.js + `child_process.spawn` | Manage `sbx`, Git, CLI streams | Prefer spawn argument arrays; no interpolated shell. |
| Database | PostgreSQL + Drizzle ORM | Transactions, uniqueness, leases, event history | Postgres is also the initial queue. |
| Validation | Zod schemas | Validate YAML, API payloads, event envelopes, configs | Errors surface clearly in dashboard. |
| GitHub | Octokit / GitHub REST and GraphQL | GitHub App auth, issues, PRs, optional Projects v2 | Use installation tokens, not broad PAT where possible. |
| Realtime | SSE (`text/event-stream`) | Server-to-browser state and activity | Support cursor/replay and heartbeat. |
| Isolation | Docker Sandboxes (`sbx`) | MicroVM workspace and daemon isolation | CLI first; evaluate official SDK only after validation. |
| Reverse proxy | Existing Traefik | Eliminates redundant dependency | Only `web` has Traefik labels. |
| Services | Docker Compose | Portable application deployment | Host runner separately installed through systemd. |
| Repo/agent config | Markdown / YAML in Git | Portable, human- and agent-editable | Strong schema validation and version snapshots. |
| Logs | Structured JSON events in Postgres + optional compressed files | Trace and replay execution | Retention/redaction policy required. |

**Why not Fastify?** It would work well, but the system is limited by long-running external CLI/API operations, not raw HTTP request throughput. Hono gives a smaller HTTP layer. Architectural success does **not** depend on Hono: controllers should call independent orchestrator services, and the HTTP interface must remain replaceable.

**Why not a pure Next.js monolith?** One app is technically possible and could be used for a proof of concept. A separate API process gives a clearer long-lived scheduler, SSE endpoint, and runner-facing interface. This separation is preferred but can be consolidated in a very early spike.

**Why not a message broker?** A transactional Postgres queue with `FOR UPDATE SKIP LOCKED`, leases, and event replay is sufficient for a small home-server factory. Later scale-out may justify a dedicated queue; the job interface should stay stable.

---

## 5. Canonical configuration: projects, workflows, agents

### 5.1 Configuration ownership and precedence

Three configuration layers:

1. **Factory policy** (trusted; always enforced): allowed repos, approval requirements, runtime security, network policy, global caps, credential boundaries, dangerous actions denied.
2. **Project/workflow settings** (trusted if reviewed): repository metadata, allowed harnesses/models, validation commands, stage sequence, per-project concurrency/budget, project-specific agent overrides.
3. **Agent/task instructions** (behavioral, not security): reusable specialist's body, repo `AGENTS.md` / `CLAUDE.md`, issue descriptions/acceptance criteria, per-task context.

Precedence for configurable *defaults*: factory defaults → project defaults → workflow stage overrides → selected agent definition → explicitly allowed run override. **Security policy is not overridden by any lower-trust input.** A repository or issue may instruct behavior, but cannot grant secrets or expand permissions.

The system should store a fully resolved immutable **execution configuration snapshot** (workflow version, agent file checksum, harness version, model ID, sandbox kit, project commit SHA, policy version) before starting a job. Changing definitions affects future jobs, not ones already running.

### 5.2 Project registry (`config/projects.yaml`, proposed schema)

```yaml
schemaVersion: 1
projects:
  - id: photo-share
    repository: your-org/photo-share
    defaultBranch: main
    enabled: true
    workflow: standard
    issuePickupLabel: agent:ready
    maxConcurrentExecutions: 2
    requireHumanMergeApproval: true
    githubProjectV2: null
    validationProfile: nextjs
    # PROPOSED (§5.3, §9.6): repo paths a branch may not change without a human
    # decision, and paths whose change forces a review surface.
    protectedPaths: [".github/workflows/**", "vitest.config.ts"]
    pathSurfaces:
      security: ["src/auth/**", "drizzle/**", "package.json", "pnpm-lock.yaml"]
      ui: ["src/app/**", "src/components/**"]

  - id: polypolicies
    repository: your-org/polypolicies
    defaultBranch: main
    enabled: true
    workflow: standard
    issuePickupLabel: agent:ready
    maxConcurrentExecutions: 1
    requireHumanMergeApproval: true
    githubProjectV2: null
    validationProfile: nextjs
```

Repository slugs and all derived paths must be normalized and allowlisted. Do not accept arbitrary filesystem paths or remote URLs from issues. Per-project config may eventually be managed via dashboard and committed back to a Git-managed factory config repository; configuration edits require validation and an audit record.

### 5.3 Workflow definition (`workflows/standard.yaml`, proposed schema)

```yaml
schemaVersion: 1
id: standard
name: Standard issue-to-PR workflow
version: 1

stages:
  - id: plan
    kind: agent
    agent: software-architect
    output: implementation-plan

  - id: implement
    kind: agent
    agent: senior-developer
    dependsOn: [plan]
    maxAttempts: 3

  - id: validate
    kind: validation
    dependsOn: [implement]
    profile: project-default

  - id: review
    kind: agent
    agent: code-reviewer
    dependsOn: [validate]
    requiresReadOnlyWorkspace: true

  - id: deliver
    kind: github-pull-request
    dependsOn: [review]
    requiresHumanApproval: false

completion:
  requireMergedPullRequest: true
  requireRequiredChecks: true
  requireHumanApprovalForMerge: true
```

**Interpretation:** stages are deterministic orchestrator nodes. `kind: agent` dispatches an agent instance through a selected harness; `kind: validation` is a trusted runner-configured command stage; `kind: github-pull-request` is performed by the trusted GitHub service rather than delegated to Claude. There should be an explicit implementation/review correction loop bounded by `maxAttempts`; never assume a YAML DAG naturally describes cycles without defined semantics.

A reduced **MVP workflow** can be `implement → validate → deliver`; add plan and review once the vertical slice works reliably.

**Risk-based stage routing (PROPOSED, post-MVP).** When a workflow starts with a `triage` stage (§8.8), the review stages and their agents can come from triage's risk and surfaces through a deterministic table, instead of every reviewer running on every task:

```yaml
routing:
  # Code-review agent by risk. There is always at least one review stage.
  review-code:
    low: code-reviewer-fast
    medium: code-reviewer
    high: code-reviewer
  # Review stages added per surface.
  surfaces:
    ui: [review-ux]
    security: [review-security]
  # Applied when there is no triage result: the most conservative set.
  default: { risk: high, surfaces: [ui, security] }
```

- The table selects **agent IDs**, not models (§1.2 principle 5). Each agent definition carries its own model preference, so a cheaper reviewer is a separate agent definition such as `code-reviewer-fast`.
- Triage's rating is model output derived from untrusted issue text, so it can only add review on top of a configured floor. The project's `pathSurfaces` (§5.2) is evaluated by the orchestrator against the candidate SHA's diff after implementation, and can only add surfaces. A task triaged `low` whose diff touches `src/auth/**` still gets the security review.
- The resolved stage set is written into the execution configuration snapshot (§5.1) and shown on the execution page with the reason each stage runs: risk, surface or path rule.

### 5.4 Agent definitions — accepted canonical format

**DECIDED:** canonical files are Markdown with YAML frontmatter, one agent per file, stored in `agents/<slug>.md`. The body is the behavioral prompt. The factory validates a portable core and stores optional factory-specific metadata under one namespace. A `model: inherit` preference is permitted; exact upstream IDs can be selected by project/stage policy.

```markdown
---
name: senior-developer
description: Implements production-quality features and bug fixes
model: inherit

factory:
  preferredHarness: auto
  capabilities:
    - repository.read
    - repository.write
    - shell.execute
  execution:
    timeout: 30m
    maxAttempts: 3
---

# Senior Developer

You are a senior software engineer implementing approved GitHub issues.

## Responsibilities
- Understand the issue's acceptance criteria.
- Follow established project architecture and instructions.
- Make focused changes, add appropriate tests, and validate your work.
- Commit the result to the assigned branch.

## Constraints
- Avoid unrelated changes and never inspect production secrets.
- Do not merge PRs, deploy, or change access permissions.
- Report blockers rather than silently expanding scope.

## Completion
- Summarize changes, tests, assumptions, and unresolved risks.
```

This is the **factory's** schema, not a universal upstream format. A harness may ignore custom `factory` keys if copied verbatim; the compiler must strip or map them and issue diagnostics. `capabilities` in Markdown are desired capabilities, **not** proof of enforced tool authorization.

### 5.5 Shared portable subset and harness-specific metadata

Portable core:
- `name` (nonempty unique slug; factory requires explicit name)
- `description` (use-case, trigger, intent)
- Markdown body (instructions)

Partially portable:
- `model` (`inherit` or alias; upstream model identifier formats differ)
- tool access controls (names, deny semantics, and execution mode differ)
- read-only/write/edit/shell restrictions
- nested delegation/background modes

Nonportable (keep in `factory` or an adapter overlay):
- runtime, sandbox limits, GitHub permissions, max retries, timeout, budget, app policy, approvals, CLI syntax, runner behavior.

### 5.6 Repository instructions versus specialist definitions

- `AGENTS.md`: repository-wide instructions used by Codex and various other harnesses; not generally a *named specialist* definition.
- `CLAUDE.md`: Claude Code's own repository instruction convention.
- `SKILL.md`: a reusable skill/procedure bundle, not equivalent to an agent definition.
- `agents/reviewer.md`: the factory's standalone specialist definition; optionally compiled into native subagent format.

Keep repository content untrusted relative to factory authorization policy. Handle contradictory `AGENTS.md`/`CLAUDE.md` explicitly; display which instructions were loaded and in what order, and avoid blindly duplicating text in contexts where the harness loads it automatically.

---

## 6. Agent/harness compatibility and compilation

### 6.1 What is verified as of the review date

| Harness | Native custom agent form | Common location | Factory approach |
|---|---|---|---|
| **Claude Code** | Markdown + YAML frontmatter | `.claude/agents/*.md` | Compile supported fields and explicitly invoke/select specialist or inject instructions in headless stage. |
| **OpenCode** | Markdown + YAML frontmatter (also config JSONC) | `.opencode/agents/*.md` | Compile native profile; use harness CLI/API for actual selection. |
| **Cursor** | Markdown + YAML frontmatter | `.cursor/agents/*.md` | Import/export compatibility; not required as MVP headless runner. |
| **GitHub Copilot** | Markdown + YAML frontmatter | `.github/agents/*.agent.md` | Import/export compatibility; not required as MVP headless runner. |
| **Codex CLI** | Standalone custom agents in TOML | `.codex/agents/*.toml` | Compile name/description/body to TOML `developer_instructions`; use validated headless launch mechanism. |
| **Grok Build** | Headless CLI verified; universal agent-file schema **not established** | TBD | Inject compiled role instructions as a prompt, or adopt native format only after a compatibility probe. |

**Crucial distinction:** Markdown/YAML *shape* is common, not the metadata *schema*. Agents in these directories are frequently **subagents** that a parent harness may invoke autonomously. The factory needs an independent, explicitly selected stage worker. A compiled native file is not evidence that the correct stage agent ran.

### 6.2 Importer/exporter behavior

Import from:
- `.claude/agents/**/*.md`
- `.opencode/agents/**/*.md`
- `.cursor/agents/**/*.md`
- `.github/agents/**/*.agent.md`
- `.codex/agents/**/*.toml` (lossy when needed)

For each file:
1. Parse frontmatter/TOML using a hardened parser and size limits.
2. Normalize to the canonical `AgentDefinition` and preserve original source metadata.
3. Map known fields; mark unsupported or ambiguous settings as **warnings**, not silent no-ops.
4. Preserve instructions exactly unless user explicitly edits.
5. Detect same-name conflicts and require deterministic selection/rename.
6. Display a compatibility report in UI (full, partial, unsupported).
7. Export only validated native configurations; never turn a requested deny into an allow.
8. Test native loading via versioned fixture/probe where supported.

The compiler may generate a **runtime prompt bundle** even when no native profile can be emitted. This is the required fallback for independent stage execution.

### 6.3 Example Codex output (proposed compilation)

```toml
name = "senior-developer"
description = "Implements production-quality features and bug fixes"
developer_instructions = """
# Senior Developer

You are a senior software engineer implementing approved GitHub issues.
Follow the repository instructions. Implement the acceptance criteria.
Do not merge pull requests or deploy.
"""
```

The actual compiler inserts the full Markdown body and maps supported model fields only after checking the installed Codex schema. Do not infer that TOML policies can independently tighten GitHub/sandbox access. The factory and runtime enforce permissions.

### 6.4 Proposed types

```ts
type HarnessId = "claude" | "codex" | "opencode" | "grok";
type RuntimeId = "docker-sandbox";

type AgentDefinition = {
  id: string;
  name: string;
  description: string;
  instructionsMarkdown: string;
  modelPreference: "inherit" | string;
  preferredHarness: HarnessId | "auto";
  capabilities: string[];
  sourcePath: string;
  sourceHash: string;
  version: number;
};

type HarnessCapabilities = {
  headless: boolean;
  structuredOutput: boolean;
  resumeSession: boolean;
  nativeAgentFile: boolean;
  nativeAgentSelection: boolean;
  modelOverride: boolean;
  toolEvents: boolean;
  usageEvents: boolean;
};

type ResolvedAgent = {
  definition: AgentDefinition;
  harness: HarnessId;
  model: string;        // validated upstream ID / adapter alias resolution
  kit: string;          // installed sbx kit reference or built-in name
  invocation: { executable: string; args: string[]; workdir?: string };
  compiledInstructionsPath?: string;
  capabilities: HarnessCapabilities;
};
```

### 6.5 Agent resolution algorithm

1. Load/validate the selected agent definition by ID and checksum.
2. Determine allowed harnesses/models from factory policy + project/workflow overrides.
3. Choose preferred harness if compatible; otherwise use *only* an explicitly allowed fallback.
4. Verify installed CLI/kit/version, provider authentication, required feature support (headless, read-only, session capture, etc.).
5. Resolve exact model ID from a versioned model registry or harness default; **fail if unavailable**, do not silently substitute another model.
6. Compile agent definition to native config and/or isolated prompt bundle.
7. Apply real permissions at sandbox/runner/GitHub layers.
8. Record resolved harness/model/kit/CLI versions/config hashes in execution snapshot.
9. Launch; normalize native structured events into a stable factory schema.

**User-facing UI:** “Architect”, “Senior Developer”, “QA Engineer”, “Code Reviewer” are agents; model and harness are advanced configurable properties of each definition or workflow override. Optional roles can choose different harnesses (e.g. Claude plans, Codex implements, Grok reviews) without exposing raw CLI plumbing to ordinary users.

### 6.6 Harness order and verification gates

- **MVP:** Claude Code adapter, since an existing Claude Code `sbx` environment is already in use.
- **Next:** Codex adapter; Docker lists Codex among built-in kits.
- **Then:** OpenCode as general provider-flexible harness; inspect its exact installed version and API/CLI format.
- **Then:** Grok Build; xAI documents `grok -p` and `--output-format streaming-json`, but Docker does not list Grok as a built-in sandbox kit. Build a custom kit or use a compatible harness only after testing.
- **Optional:** Cursor/Copilot support as import/export targets before committing to them as automated headless workers.

The factory must not claim every agent supports every model. Docker's `sbx run --model` feature is currently documented as **experimental** for built-in Claude/Codex/OpenCode; consider it a separately gated enhancement, not the core resolution contract.

---

## 7. Docker Sandbox execution strategy and runner adapter

### 7.1 Verified Docker Sandbox boundaries

Docker documents the headless automation pattern **`sbx create` → `sbx exec` → retrieve Git work → `sbx rm`**. Clone mode creates a separate working clone inside the sandbox; the host checkout is exposed read-only at `/run/sandbox/source` in that mode. Clone mode is chosen at **creation time** and cannot be retrofitted on an existing sandbox. A sandbox and its internal files survive stop/restart but are destroyed by `sbx rm`.

Relevant examples, to be checked against the installed `sbx` version before hardcoding:

```bash
# Run from the dedicated factory-owned main checkout of a Git repository.
cd /var/lib/factory/repos/photo-share

# Create a named isolated clone-mode sandbox without attaching.
sbx create --clone --name factory-photo-share-42-a1 claude .

# Execute a headless Claude task. In production the runner generates a complete
# prompt, ensures the working directory/ref is correct, and streams JSON output.
sbx exec factory-photo-share-42-a1 -- \
  claude --dangerously-skip-permissions \
  -p "Implement the assigned task; commit changes on the assigned branch" \
  --output-format stream-json --verbose

# The agent must create/commit its own isolated feature branch.
# Fetch while the sandbox is reachable; remote is created by clone-mode sbx.
git fetch sandbox-factory-photo-share-42-a1

# Review the expected ref/commit before pushing from trusted host tooling.
git log --oneline sandbox-factory-photo-share-42-a1/factory/photo-share/42-a1

# Remove only after the work and required artifacts are durably preserved.
sbx rm factory-photo-share-42-a1
```

**VERIFY:** the precise workdir inside a chosen kit, default branch name, CLI option order, process cancellation, and stream framing are version-sensitive. Use feature probes during installation. Never derive a sandbox name directly from unsanitized user input. `--dangerously-skip-permissions` is a harness permission-bypass mode that may be appropriate *inside the microVM*, but it is not a security boundary by itself; sandbox network/resource policy and trusted host credentials still govern actual privileges.

**Alternative:** Docker now documents experimental `sbxenv.yaml` files for portable runtime settings (resources, tools, credentials, lifecycle). These are attractive for reproducible kits, but `sbx env` and some schema details are experimental. Start with a narrowly validated CLI adapter; optionally generate/version `sbxenv.yaml` and use `sbx env create --force` / `sbx env exec` after conformance tests. Keep env files outside agent-writable mounts because they can contain host lifecycle commands.

### 7.2 Isolation model

- **Default:** each execution gets a unique, dedicated clone-mode sandbox, named with a sanitized project ID, issue number, and attempt/execution identifier.
- Do not run two independent writing executions against the same checkout. If multiple stages share one execution sandbox sequentially, guarantee single active writer.
- **Review:** preferably run review in a separate *read-only* sandbox/checkout or a harness configuration with verifiable read-only behavior; treat a prompt saying “do not edit” as insufficient.
- **Alternative LATER:** mountless sandbox where the checkout lives only in the microVM and commits are pushed to a trusted remote; stronger separation from the host workspace but requires controlled Git bootstrap and credential mechanics.
- The host repo cache must be factory-owned, stripped of local personal files, and used only as a base checkout. Do not point clone mode at an individual's active working tree.
- Resource limits: per-sandbox CPUs/memory, max runtime, max local disk footprint, max nested Docker build storage, and global concurrency. Inspect actual `sbx` CLI options at install time.
- Network: allow only model providers, required package registries, and explicitly approved endpoints. Docker's default network policy is restrictive; do not assume a sandbox can reach GitHub or an arbitrary package registry until rules are checked.
- Avoid granting agents unbounded external GitHub credentials. The control plane/runner should handle pushes and PR creation.

### 7.3 Host runner adapter contracts

```ts
type SandboxId = string;
type ExecutionId = string;

type SandboxSpec = {
  executionId: ExecutionId;
  sandboxName: string;
  kit: string;
  mainCheckoutPath: string;
  cloneMode: true;
  cpuCount?: number;
  memoryLimit?: string;
  allowedHosts: string[];
};

type LaunchSpec = {
  command: string;
  args: string[];
  cwd?: string;
  timeoutMs: number;
  promptBundlePath?: string;
};

interface SandboxAdapter {
  create(spec: SandboxSpec): Promise<{ sandboxId: SandboxId; name: string }>;
  exec(id: SandboxId, launch: LaunchSpec): AsyncIterable<RawProcessEvent>;
  inspect(id: SandboxId): Promise<SandboxHealth>;
  stop(id: SandboxId): Promise<void>;
  remove(id: SandboxId): Promise<void>;
  listOwned(): Promise<SandboxHealth[]>;
  retrieveCommits(id: SandboxId, expectedBranch: string): Promise<GitArtifact>;
}

interface HarnessAdapter {
  id: HarnessId;
  probe(): Promise<{ version: string; capabilities: HarnessCapabilities }>;
  compileAgent(def: AgentDefinition, context: StageContext): Promise<CompiledAgent>;
  buildLaunchCommand(compiled: CompiledAgent, context: StageContext): LaunchSpec;
  normalizeEvent(raw: RawProcessEvent): AgentEvent | null;
  buildResumeCommand?(sessionId: string, context: StageContext): LaunchSpec;
}
```

**Separate these adapters:** `SandboxAdapter` provisions and controls compute, `HarnessAdapter` handles CLI/session/prompt/streaming specifics, and `GitArtifactService` safely checks and pushes commits. The scheduler should depend on interfaces, not `sbx` command strings.

### 7.4 Process management requirements

- Use `spawn(executable, args, {shell:false})`; no shell interpolation of issue titles, labels, names, or user prompts.
- Send large prompts via a private temporary file or stdin if the harness supports it, not fragile/secret-revealing command arguments.
- Capture stdout as a **byte stream** and parse complete line-delimited JSON objects with buffering, line-size caps, malformed-event counters, and backpressure.
- Capture stderr separately (bounded, redacted, timestamped); some tools produce diagnostic text on stderr despite valid JSON stdout.
- Persist native session IDs and process identity, if exposed.
- Implement graceful cancel, process-group termination, `sbx stop` fallback, and forced recovery path. Cancellation must not mark success.
- Record exit code and signal, and distinguish **process completion** from **task validation success**.
- Require an agent-produced structured completion summary and committed Git artifact; never infer success from a friendly natural-language answer alone.
- Support independent timeouts: overall run, per stage, command idle timeout, heartbeat expiry, and Git API timeout.
- Publish a `process_started`, `tool_call`, `tool_result`, `usage_reported`, `process_exited`, and `artifact_collected` event as applicable.
- Never display hidden/private model reasoning as if it were available. A “processing” state is inferred from runtime state; agent tool/output summaries are real observable events.

### 7.5 Provider-specific headless patterns

**Claude Code** (documented concept):

```bash
claude -p "<task>" --output-format stream-json --verbose
```

CLI supports a noninteractive print mode and structured output. Verify agent selection and resume flags from the installed version. Existing interactive Claude sessions are not assumed adoptable by the factory without explicit import/reconciliation.

**Codex CLI** (documented concept):

```bash
codex exec --json "<task>"
```

Verify exact installed Codex output framing, write/sandbox flags, session resume syntax, and whether native custom agent invocation is supported in headless `exec` for the selected version. Do not rely on interactive `/subagents` commands for unattended work.

**OpenCode:** use its documented agent definition and headless/SDK interface for installed version; record adapter-specific feature support.

**Grok Build** (xAI documentation):

```bash
grok -p "<task>" --output-format streaming-json
```

Docker's built-in sandbox kit list does **not** establish a native `grok` kit. Package Grok Build through a custom kit or run it through a proven compatible harness; use xAI-authenticated credentials and a tested CLI. Do not assume its agent frontmatter support matches Claude/OpenCode.

### 7.6 Git handoff and trust boundary

1. Runner keeps a clean, machine-owned local checkout per registered repo. Synchronize with allowed GitHub default branch before creating each execution.
2. Create a unique sandbox clone pinned to a known base SHA; instruct agent to create `factory/<project>/<issue>-<execution-short>` branch.
3. Agent modifies/commits inside sandbox. A missing commit is a blocked/failed stage, not a PR-worthy artifact.
4. Runner fetches the expected branch from the temporary `sandbox-<name>` remote while sandbox exists.
5. Runner verifies expected ref, base commit, branch prefix, Git object integrity, allowed repo identity, absence of unexpected merge parent/risky paths if policy disallows them, and required validations.
6. Runner pushes **exact verified SHA** to an explicitly constructed `refs/heads/factory/...` destination on an allowlisted GitHub remote using a short-lived credential only outside sandbox.
7. GitHub service creates/updates a PR with issue linkage, tests, summary, and artifact references. Use idempotency lookup keyed by execution/branch and issue.
8. Preserve branch SHA and PR URL in Postgres. Only then clean up sandbox. PR merges and CI are observed asynchronously.

If the agent has altered its in-sandbox `origin`, that must not change the trusted host push destination. Never run an untrusted `git push` command assembled from the agent's output. Git object content is still untrusted; downstream CI remains a separate security boundary.

---

## 8. GitHub integration: work discovery, repositories, PRs, CI

### 8.1 Source of truth

- **GitHub Issues:** title, body, labels, comments, author, assignees, open/closed state, acceptance criteria.
- **GitHub Projects v2 (optional):** planning board/status/fields; sync selectively via GraphQL and only when configured.
- **GitHub Git repos:** source, branches, code review, commits.
- **GitHub Pull Requests and checks:** proposed changes, review approval, CI, merge result.
- **Factory Postgres:** agent execution state, scheduling, stage/instance history, prompts/config snapshots, errors, budgets, events, sandbox inventory.

Never use a GitHub issue label as the **only** scheduling lock. External labels may be edited and polling can be concurrent. Use a unique DB lease and reconciliation logic.

### 8.2 Proposed issue-label convention

| Label | Projection meaning |
|---|---|
| `agent:ready` | Human-approved eligibility for dispatch. |
| `agent:working` | Factory currently owns an active execution. |
| `agent:blocked` | Human input, failing prerequisite, or recovery needed. |
| `agent:review` | PR ready or requires review. |
| `agent:done` | Completion conditions met (merged + required checks). |

Avoid auto-pickup of every new issue. A human sets `agent:ready` or presses Queue. A separate factory project policy can disable pickup globally or per repo. The projector should update labels conservatively; never erase unrelated GitHub labels.

### 8.3 Inbound sync on a LAN-only host

**Recommended:** GitHub API polling from API/worker over outbound HTTPS; no inbound webhooks.

- Every ~30–60 seconds per enabled repo (jittered), query updated issues filtered by labels and/or `since`. These are proposed defaults, configurable to avoid rate limits.
- Persist last successful high-water timestamp and GitHub object IDs/ETags; handle pagination and overlapping windows to account for clock skew.
- Filter out pull requests from the general Issues endpoint where necessary (the API can expose PRs through issue-shaped records).
- Periodically run a full reconciliation for all active tasks, PRs, and CI states even if no `updated_at` change was observed.
- Poll with authenticated GitHub App installation tokens; respect rate limits, secondary limits, conditional requests, and retry/backoff.
- Optionally reconcile Projects v2 status as a projection; do not require Projects access for basic operations.
- When a GitHub issue is closed/label removed, apply configured cancellation or block rules; never blindly kill work based on a transient API failure.

### 8.4 GitHub App authentication

Use a private GitHub App installed only on selected repositories when possible. The API stores the App ID, installation ID(s), and private key in a protected secret store/environment; use Octokit to mint short-lived installation tokens. GitHub installation tokens are time-limited (documented as one hour), so renew for each operation or let SDK manage refresh. Repositories and token permissions can be constrained further per installation-token request.

Suggested **permissions to evaluate against actual endpoints**:
- Metadata: read (implicit).
- Issues: read/write (sync labels, comments, issue state).
- Contents: read/write only if trusted runner pushes branches; ideally separate narrowly scoped path/operation.
- Pull requests: read/write (create/update PRs and retrieve review state).
- Actions/checks/statuses: read as needed to observe CI.
- Projects: only if optional Projects v2 integration is enabled and supported by App installation/access context.

The exact minimum permission set must be determined by the implemented GitHub endpoints. Credentials for the trusted GitHub service must **not** be copied into agent sandboxes. A sandbox may need read-only package or repository access; configure separate restricted credential handling for that instead.

### 8.5 Issue discovery and duplicate prevention

Task identity: stable GitHub repository node ID + issue node ID (or repo ID + issue number), not just issue number. Unique constraint prevents duplicate local task records. For a single issue, only one active execution unless explicitly modeling a separate independently named subtask. Use an idempotency key such as `repo_id:issue_node_id:workflow_version:request_nonce` for manual dispatch, and a deterministic discovery key for auto-pickup.

If the issue changes while an agent runs, record the new issue version and apply a policy: continue against original snapshot with a visible warning, or pause for replan if acceptance criteria change. Do not silently mutate a live prompt.

### 8.6 PR delivery contract

- Stable branch `factory/<project>/<issue>-<execution-short>`.
- PR title references issue number; body includes `Closes #<issue>` only if intended on merge, plus implementation summary, tests, risks, and run ID.
- Use draft PRs if validation/review is incomplete but useful code should be surfaced.
- Do not duplicate PRs on retry: search by `head` branch and persisted mapping before creating.
- Maintain GitHub label projection and optional Project v2 status; preserve manually edited metadata.
- After PR created: `pr_open` / `awaiting_human` status; only `completed` after merge + required checks policy is satisfied.
- A closed, unmerged PR is **not** completion; mark blocked, cancelled, or requiring rework based on policy.
- Force pushes are not allowed by default; any rework updates must be explicit and auditable.
- Enforce branch protection / rulesets for approval. Prompt instructions alone are not a merge authorization mechanism.
- **Re-test against the current base before merge (PROPOSED).** Validation (ADR 0003) tests the candidate SHA on the base it was cut from, and other PRs may merge first. The factory does not rebase (ADR 0008), so this is covered on GitHub: require branches to be up to date before merging, or use a merge queue, so required checks run on the merged result (§14.3). The execution page shows "base moved since validation" when the default branch head differs from the execution's recorded base SHA.

### 8.7 Optional planning pipeline (LATER)

Distinguish two independent product capabilities:
- **Delivery pipeline (MVP):** implement **approved existing issues** into PRs.
- **Planning pipeline (later):** inspect code, draft recommendations and GitHub issues, decompose epics, and propose priorities for human acceptance.

Draft planning output must not automatically trigger its own execution; require explicit issue approval/label to prevent recursive, runaway task creation.

### 8.8 Optional triage stage (PROPOSED, post-MVP)

A read-only agent stage that runs first, after a human has set `agent:ready`. It does not replace that approval: the label stays the trust gate. Triage decides how the issue should be built, and whether it can be built as one PR.

| Verdict | Effect |
|---|---|
| `accept` | Writes a **scope brief** that later stages treat as binding. It may narrow an oversized issue to a first slice that is useful on its own (`rescoped: true`, listing what was left out), or settle a vague issue on the reading the code makes obvious. Rates `risk` (`low` / `medium` / `high`) and `surfaces` (`ui`, `security`) for routing (§5.3). |
| `needs_info` | Posts the fewest questions that unblock the work, each answerable in a line, as one issue comment. The execution moves to `BLOCKED` (reason `awaiting_author`) and resumes at triage after a reply. |
| `reject` | Posts the reasoning and moves the execution to `BLOCKED` with a proposed cancellation. The operator confirms. The factory never removes `agent:ready` on its own. |

- Triage is isolated like the review stage (ADR 0003): its own sandbox, from which the runner takes only the schema-validated result (§9.2); anything it writes is discarded. Its instructions give it no reason to run commands, but that is an instruction, not enforcement (§14.4).
- It reads the issue snapshot (§8.5) as quoted data, with comments filtered as for every other stage (ADR 0009 §3: repository owners, members and collaborators only).
- Questions reach the author before any plan exists, so later stages never stall on a question only the author can answer. Later stages record what is still ambiguous as assumptions in their result. Assumptions that amount to product decisions (pricing, public copy, what is free versus paid) are raised as decision requests (§9.2), not settled silently.
- An author reply that arrives after `agent:ready` is content the approving human has not seen. When the author is not an owner, member or collaborator, the comment filter would drop their answer, so resuming needs one operator acknowledgement in the Inbox (§12.1 E). The acknowledgement shows the new comments and any body edits, and admits only the replies it shows into the triage prompt.
- The verdict, scope brief and any rescoping are posted on the issue, so the author sees what will be built. The issue body is never edited.

---

## 9. Orchestrator state machine and scheduler

### 9.1 Work lifecycle

A **GitHub Task** can have multiple **Executions** over time; an Execution has ordered **Stage Runs**, and each agent stage creates an **Agent Instance**. Keep these layers distinct.

Proposed execution states:

```text
DISCOVERED -> QUEUED -> CLAIMED -> PLANNING -> IMPLEMENTING
                          |             |             |
                          |             |             v
                          |             |        VALIDATING <--- bounded repair
                          |             |             |
                          |             +----------> REVIEWING
                          |                           |
                          v                           v
                       BLOCKED <---------> AWAITING_APPROVAL
                                                      |
                                                      v
                                               PREPARING_PR
                                                      |
                                                      v
                                               PR_OPEN / AWAITING_MERGE
                                                      |
                                                      v
                                             COMPLETED (merged + CI)

Any nonterminal state -> FAILED / CANCELLED (by explicit rule)
Uncertain host/runner state -> RECOVERY_REQUIRED, never fake success
```

Treat PR-open and awaiting-human as nonterminal. Keep stage state separate from execution state to avoid losing granularity.

### 9.2 Stage results

Every stage returns one of:
- `succeeded` — required artifacts are present and verified.
- `failed_retryable` — explicit failure with budget remaining and safe retry policy.
- `failed_terminal` — unrecoverable or disallowed operation.
- `blocked` — requires user input, credential fix, or external dependency.
- `cancelled` — requested stop, with process termination evidence.
- `lost` — process/runner unknown; require recovery reconciliation.

Success is defined by machine-verifiable outputs wherever possible. A headless agent returning exit code 0 is necessary but not sufficient for implementation success.

**Result envelope (PROPOSED).** Each agent stage's structured completion (§7.4) leads with a verdict from a closed enum for its stage kind, then a summary, then the detail:

| Stage kind | Verdicts |
|---|---|
| triage (§8.8) | `accept`, `needs_info`, `reject` |
| plan, implement | `done`, `blocked` |
| review | `pass`, `changes` |

- The orchestrator branches on the verdict and on verified artifacts, never on prose. A missing or unknown verdict is an invalid result (`failed_retryable` within budget) and never counts as a pass. A `done` verdict still needs a committed SHA and passing validation (§9.6).
- `summary` is 3–5 short bullets for the dashboard. The detail (acceptance-criteria evidence, assumptions, review findings as `path:line` plus the requested fix) is stored as an artifact for the next stage and for humans.
- Review results name the SHA they reviewed. Findings separate blocking `changes` from non-blocking notes, so a style preference doesn't cost a repair attempt.

**Decision requests (PROPOSED).** Every `blocked` result, from an agent or from the factory itself (attempts exhausted, conflict, refused transition), carries a decision request: the one question a human must answer, the options with what each leads to, and the artifacts to read. It is posted as a single comment on the PR, or on the issue when there is no PR, and listed in the Approvals inbox (§12.1 E). "Please review" is not a decision request. When attempts run out, the request lists the outstanding findings by stage.

### 9.3 Scheduling policy

- Poll for `agent:ready` and dashboard-queued tasks.
- Check project enabled, branch rules, dependencies, issue status, human approval preconditions.
- Apply global concurrency cap, project cap, per-harness/provider rate limits, and daily/weekly budget.
- Prioritize explicit user actions over auto-discovery if configured; fair round-robin across projects prevents one repo from starving others.
- Choose eligible task, write execution + job within one database transaction, atomically acquire ownership.
- The worker claims a **job lease** with an expiration timestamp and fencing token; retries of API calls must be idempotent.
- Only the lease holder with current fencing token can append authoritative terminal transition/result.
- On expiry, first reconcile real sandbox/process status; **do not immediately launch another copy**.
- Maintain an outbox for GitHub projections and notifications so DB commits and external API writes can be retried safely.

Postgres implementation may use `SELECT ... FOR UPDATE SKIP LOCKED` for claims. Use unique partial indexes to enforce “at most one active execution per task” and optimistic versioning for transitions. Database transactions cannot make external `sbx`/GitHub calls atomic; use sagas/idempotent compensating actions.

### 9.4 Retry policy

- Separate **transient infrastructure retry** (network timeout, GitHub 502, runner disconnect) from **agent repair loop** (failed tests/review findings).
- Have per-stage `maxAttempts`, global maximum cost/time, and explicit backoff.
- A coding retry should either resume the known sandbox/session with verified state or start a fresh attempt from an established Git ref/plan. Never assume session resume exists just because a model supports a conversation history.
- Give review or validation findings as structured inputs for bounded repair.
- After budget exhaustion, move to `blocked`/`failed` and surface a clear action to human.
- No unlimited auto-retry, no automatic switch to a more expensive provider without policy approval.

### 9.5 Dependency and parallelism model

MVP: execute *one task's stages sequentially* and run independent GitHub issues concurrently in separate sandboxes. This provides genuine multi-agent/multi-project operation without introducing merge coordination inside one issue.

Later: dependency DAGs with task fanout and independent subtasks, each with unique workspace and branch. A parent aggregator should gather results, resolve integration conflicts, and request review. Branches alone do not isolate filesystem edits in a shared directory, so parallel writers must have distinct sandboxes or worktrees.

### 9.6 Example transition invariants

1. `task.active_execution_id` points to **at most one** active execution.
2. A stage cannot start until its declared prerequisites succeeded or a documented override was approved.
3. A stage cannot be marked successful until required artifacts are stored and validated.
4. Only the runner currently owning a valid lease can finalize its active job.
5. Any `COMPLETED` execution must reference a merged PR (unless a separately declared no-code workflow explicitly permits otherwise).
6. Any destructive action has an authorized actor, reason, and audit event.
7. No sandbox is removed while unpushed/unfetched required commits remain or recovery is unresolved.
8. Every status change is paired with an append-only event, in the same DB transaction where feasible.

**Transition evidence (PROPOSED).** One transition function owns execution state, and each move names the evidence it requires. When evidence is missing, stale (for another SHA) or malformed, the move is refused: the refusal is an event with its reason, and the execution goes to `BLOCKED` with a decision request (§9.2) instead of being pushed forward some other way. Minimum evidence:

| Transition | Evidence required |
|---|---|
| `IMPLEMENTING → VALIDATING` | Verdict `done`; at least one commit on the expected branch on top of the recorded base SHA; the fetched candidate SHA recorded (ADR 0003). |
| `IMPLEMENTING → BLOCKED` | Verdict `blocked` with a decision request. |
| `VALIDATING → REVIEWING` | A passing validation report for exactly the candidate SHA, and no change under the project's `protectedPaths`. |
| `VALIDATING → IMPLEMENTING` | A failing report for the candidate SHA and attempts remaining; the failing output is the repair input. With no attempts left, `BLOCKED`. |
| `REVIEWING → AWAITING_APPROVAL` | A schema-valid result from every routed review stage (§5.3) for the candidate SHA, all `pass`. |
| `REVIEWING → IMPLEMENTING` | At least one `changes` verdict and attempts remaining; the findings are the repair input. With no attempts left, `BLOCKED`. |
| `PR_OPEN → COMPLETED` | PR merged and required checks satisfied (invariant 5). |

Validation commands are approved config (§14.2), but they run code the branch controls: package scripts, test-runner config, fixtures. `protectedPaths` (§5.2) lists repo files a branch may not modify or delete without a human decision, such as CI workflows or the test-runner config the validation profile depends on. It extends the runner's changed-path deny list (ADR 0009 §7), which stays as the last check before a push. A change under them goes to `BLOCKED` with a decision request rather than failing validation, which would only lead to repair attempts that cannot succeed. Tests are not protected by default, because features legitimately change them. Reviewers flag tests that were weakened, skipped or special-cased to pass.

---

## 10. Persistence and data model

### 10.1 Proposed tables

| Table | Primary responsibility / important fields |
|---|---|
| `projects` | ID, repo node ID, slug, default branch, enabled, workflow ID, caps, installation ID, sync cursor. |
| `github_tasks` | GitHub issue node ID, repo ID, number, title, body hash, labels snapshot, state, version, last_seen. |
| `workflows` | ID, version, path, checksum, parsed immutable definition. |
| `agent_definitions` | ID, source path/hash, version, summary/compatibility flags (Git remains canonical for content). |
| `executions` | Task FK, workflow snapshot, state, priority, attempt, active sandbox, timestamps, budget, idempotency key, state version. |
| `stage_runs` | Execution FK, stage ID, agent definition/hash, state, attempt, inputs/outputs, timers. |
| `agent_instances` | Stage FK, harness, model, kit/CLI versions, native session ID, PID/process metadata, state. |
| `sandbox_instances` | Sandbox name/ID, host, owner execution, state, create/stop/remove timestamps, resource policy snapshot. |
| `jobs` | Kind, payload, state, lease owner, expiry, fencing generation, heartbeat, retries, dedupe key. |
| `events` | Monotonic ID, source, run/agent reference, type, timestamp, safe payload, payload version. |
| `artifacts` | Kind, SHA256, content location, source commit/ref, metadata, timestamps. |
| `approvals` | Run/stage, requested action, actor, status, deadline, resolution note. |
| `github_links` | Execution/PR/branch/commit mappings and last GitHub projection. |
| `github_outbox` | Pending idempotent GitHub label/comment/PR updates with retry state. |
| `audit_log` | Actor, action, resource, before/after references, timestamp, IP/session where relevant. |
| `budgets_usage` | Model/provider cost or token records if available; hard caps and confidence markers. |

### 10.2 Critical constraints and indexes

- `UNIQUE(project_id, github_issue_node_id)` for tasks.
- `UNIQUE(execution.idempotency_key)` for requested runs.
- Partial unique index for one active execution per task (document exact active statuses).
- `UNIQUE(sandbox_instances.name)` and `UNIQUE(github_links.branch_ref)` scoped to repository.
- `UNIQUE(jobs.dedupe_key)` where logical operation should occur once.
- Job lease fields with monotonic fencing generation; never reuse a stale claim token.
- `events.id` monotonic cursor for replay; index `(execution_id,id)` and `(project_id,id)`.
- Immutable history for execution snapshots, stage attempts, GitHub outcome links.

### 10.3 Event model

```ts
type FactoryEvent = {
  schemaVersion: 1;
  id: string;                 // assigned by DB / event ingest
  createdAt: string;          // ISO timestamp
  source: "scheduler" | "runner" | "harness" | "github" | "operator";
  projectId: string;
  executionId?: string;
  stageRunId?: string;
  agentInstanceId?: string;
  type:
    | "task_queued" | "stage_started" | "stage_finished"
    | "agent_message" | "tool_started" | "tool_finished"
    | "process_started" | "process_exited" | "usage_reported"
    | "sandbox_created" | "sandbox_stopped" | "sandbox_removed"
    | "validation_finished" | "review_finished" | "commit_collected"
    | "pr_created" | "approval_requested" | "approval_resolved"
    | "github_status_changed" | "execution_blocked" | "execution_completed"
    | "execution_failed" | "execution_cancelled" | "recovery_required";
  payload: Record<string, unknown>; // validated, bounded, redacted
};
```

Maintain raw native event type and parser version in an internal structured field when safe. Native provider output can evolve; unknown events should be retained as diagnostic metadata or safely ignored with a warning, not crash an execution. UI displays normalized events, not entire raw stream blindly.

### 10.4 Artifacts and storage

Store small artifacts as database JSON/text and larger logs/diffs/test reports in a host-managed artifact directory (or S3-compatible storage later). Reference them with a checksum, content type, size, and retention policy. The artifact store must be backed up with the DB or regenerated from GitHub when possible.

Avoid storing provider API keys, GitHub tokens, unredacted environment variables, full `.env` contents, or high-volume sensitive tool output in events. Apply redaction on ingestion, then display. Make log retention configurable and provide an export/erase function.

---

## 11. HTTP APIs, runner protocol, and realtime streaming

### 11.1 API design rules

- Hono is the transport; orchestration services are framework-independent TypeScript.
- Version the API under `/api/v1` in the public browser route; internal application service may use `/v1` after proxy rewriting.
- Authenticate UI calls with secure same-site sessions and enforce CSRF protection for mutations.
- Authenticate runner calls separately: dedicated runner identity/token plus a per-job lease/fencing token; never trust a browser's runner headers.
- Validate all inputs with Zod, require bounded request sizes, log request ID, and avoid returning sensitive job prompts/credentials to unauthorized users.
- All mutations accept or derive an idempotency key. Long-running operations return a job/run ID, not an open request that waits for the agent.
- Use UTC in stored timestamps. UI renders local time and explicit time zone.

### 11.2 Proposed browser endpoints

| Method | Path | Behavior |
|---|---|---|
| `GET` | `/api/v1/health` | Authenticated health summary; separate unauthenticated minimal probe if required. |
| `GET` | `/api/v1/projects` | Registered repos and summarized activity. |
| `POST` | `/api/v1/projects` | Register a permitted GitHub repo; validate access/install. |
| `PATCH` | `/api/v1/projects/:id` | Enable, disable, or change safe settings. |
| `GET` | `/api/v1/projects/:id/tasks` | Synced issues and queue status. |
| `POST` | `/api/v1/tasks/:id/queue` | Explicitly queue a GitHub issue. |
| `GET` | `/api/v1/executions` | Filter by project, state, agent, issue. |
| `GET` | `/api/v1/executions/:id` | Detailed immutable config/status/stage/artifact view. |
| `POST` | `/api/v1/executions/:id/cancel` | Request cancellation (not an immediate success claim). |
| `POST` | `/api/v1/executions/:id/retry` | Authorized retry with explicit reason/attempt. |
| `GET` | `/api/v1/executions/:id/events` | Paginated event timeline. |
| `GET` | `/api/v1/events/stream` | Authenticated SSE, filterable, resumable by cursor. |
| `GET` | `/api/v1/agents` | Registry definitions, versions, compatibility. |
| `POST` | `/api/v1/agents/import` | Import and validate native definitions; preview before commit. |
| `GET` | `/api/v1/agents/:id/compatibility` | Harness compatibility diagnostics. |
| `GET` | `/api/v1/harnesses` | Installed agent runtimes and capability probes. |
| `GET` | `/api/v1/workflows` | Workflow registry and versions. |
| `GET` | `/api/v1/approvals` | Pending human decisions. |
| `POST` | `/api/v1/approvals/:id/decision` | Approve/deny, with actor and reason. |
| `GET` | `/api/v1/fleet` | Current agent instances, sandbox health, leases. |
| `GET` | `/api/v1/settings` | Global limits/security view, redacted. |

### 11.3 Proposed runner-only endpoints

| Method | Path | Behavior |
|---|---|---|
| `POST` | `/internal/runner/register` | Authenticate host runner, report capabilities/versions and identity. |
| `POST` | `/internal/runner/claim` | Atomic job claim with lease expiry/fencing generation. |
| `POST` | `/internal/runner/jobs/:id/heartbeat` | Extend currently held lease, report sandbox/process status. |
| `POST` | `/internal/runner/jobs/:id/events` | Batch append normalized events with per-batch sequence dedupe. |
| `POST` | `/internal/runner/jobs/:id/result` | Final stage result, exit info, artifact hashes, fencing token. |
| `POST` | `/internal/runner/observations` | Report startup reconciliation (owned sandbox inventory). |

The exact URL prefixes are **proposed application contracts**, not provided by Docker. For worker security, a host runner should receive only work it is authorized to execute. Make each response contain the minimal necessary ephemeral config, not all project credentials. Consider a per-run short-lived signing token in addition to the runner token.

### 11.4 SSE design

Serve `Content-Type: text/event-stream`, `Cache-Control: no-cache, no-transform`, and keep the response alive with periodic heartbeat comments. Use `id:` as a monotonic durable event ID and `event:` as the normalized type. Support `Last-Event-ID` or `?after=<cursor>` to replay missed events on reconnect. Backpressure requires bounded in-memory fan-out and a Postgres replay query; Postgres is durable storage, not an uncontrolled log of every raw token.

Example event framing:

```text
id: 18437
event: tool_started
data: {"executionId":"ex_123","agentInstanceId":"ag_42","tool":"shell","summary":"npm test"}

```

The Next.js reverse proxy must preserve streaming response bodies; do not buffer the entire event stream or cache it. Traefik should route to `web`; the web server proxies SSE on to the internal API. Avoid long blocking operations inside a Next.js request handler.

### 11.5 Runner-to-dashboard UX consistency

Events may be delivered more than once or out of order across retries: the UI sorts by monotonic event ID, ignores duplicates, and renders materialized execution state from the authoritative API. Events should trigger revalidation/updates; they should not be the only means of reconstructing correctness in the browser.

---

## 12. Dashboard: information architecture and UX requirements

The dashboard is a **control room for agent execution**, not another GitHub Issues UI.

### 12.1 Primary screens

**A. Overview / Factory Home**
- Health: API, DB, GitHub sync, runner, Docker Sandbox daemon.
- Fleet counts: active, idle, blocked, failed, awaiting human, completed today.
- Active projects, work queue sizes, PRs pending review, recent failures.
- Global resource consumption: active microVMs, CPU/memory/disk pressure if reliably measured, budgets.
- Recent activity feed with links to execution, task, sandbox, and PR.
- Paused/enabled factory switch gated by authorization.

**B. Agent Fleet (visualizer)**
- Cards or compact table for each **agent instance**, not just each reusable agent definition.
- Agent role/name, harness, model, project, GitHub issue number, stage, observed current action, start time, last activity, sandbox health, tool calls, tokens/cost when available.
- Filters by project, state, agent definition, harness/model, and age.
- Drill down to a single execution, GitHub issue/PR, logs, and artifact metadata.
- Avoid fabricated percent complete or hidden-thought summaries. A state badge and timeline are truthful; optional heuristics should be labeled approximate.

**C. Projects**
- GitHub repository, default branch, enabled/paused, configured workflow, concurrency cap.
- Synced issue queue and labels, pending/running/blocked tasks, latest PRs and CI.
- Optional Projects v2 status panels.
- Issue/task details can link out to GitHub instead of reproducing its editing experience.
- Per-project settings: allowed agents/models, validation profile, budgets, polling.

**D. Execution Detail**
- Header: repo, issue, execution ID, current state, owner runner, GitHub links, timestamps, resolved config versions.
- Stage rail: planning → implementation → validation → review → PR → merge.
- Live event timeline and aggregated tool calls; search/filter, expandable sanitized outputs.
- Logs and diagnostic stderr, artifact links, test results, git branch/commit SHA, review feedback.
- Explicit failure reason, retry history, and recovery evidence.
- Authorized actions: cancel, retry, approve, mark blocked, inspect sandbox, open GitHub PR.
- Show real resource/token metrics where supplied, otherwise `unavailable` rather than estimated without disclosure.

**E. Approvals / Inbox**
- Human-required approval tasks: plan acceptance (optional), scope changes, stuck work, risky validation, PR readiness, retry beyond policy.
- “Approve” action changes factory execution policy state; it **does not** override GitHub branch protection or merge the PR unless a separate, explicitly authorized feature exists.
- Every approval records actor, decision, reason, and timestamp.
- Decision requests (§9.2) show the question, the options and links to the artifacts. Answering one records the choice and starts the next action: rework with notes (ADR 0008), cancel, or accept as is. Triage acknowledgements (§8.8) show the author's new comments and body edits.

**F. Agent Registry / Workflow Builder**
- Browse `agents/*.md` with name, description, instructions, model/harness, supported capabilities.
- Edit Markdown and YAML frontmatter using a guarded editor, preview parsed schema, show native compilation compatibility.
- Import existing `.claude/agents`, `.opencode/agents`, `.cursor/agents`, `.github/agents`, `.codex/agents`; warn on unsupported fields.
- Versioned workflow editor with per-stage **agent assignment** (not merely model assignment), timeout, validation, and approval policies.
- Advanced settings allow changing the model and harness for an agent, or overriding them per project/workflow when permitted.

**G. Settings & Operations**
- GitHub App/installations, project allowlist, polling and API health, runner status, CLI kit inventory, limits/budgets, backup status.
- Configure which existing Traefik hostname is used, without provisioning a second proxy.
- “Test harness” feature runs a harmless probe inside a disposable sandbox and reports exact capabilities.

### 12.2 Dashboard visual interaction example

```text
Factory                                       [All projects v] [Operational]

 Active Agents  4/6     Queued Issues  18     Blocked  1     Awaiting PR Review  3

 ┌─ Builder 01 ──────────────────┐  ┌─ Builder 02 ─────────────────┐
 │ photo-share  #42              │  │ photo-share  #46             │
 │ Implementing                  │  │ Validating                   │
 │ Claude Code / configured model│  │ Codex / configured model     │
 │ Last tool: edit auth callback │  │ Last tool: npm test          │
 └───────────────────────────────┘  └──────────────────────────────┘

 ┌─ Reviewer 01 ─────────────────┐  ┌─ Builder 03 ─────────────────┐
 │ polypolicies  #18             │  │ polypolicies  #21            │
 │ Reviewing                      │  │ Blocked: requires input      │
 └───────────────────────────────┘  └──────────────────────────────┘

 Recent events        19:05  PR opened #84   19:03  Validation succeeded
 Pending actions      Review PR #84 / Resolve blocked issue #21
```

This is conceptual, not live data. Consider a dense fleet table for >10 agents and cards for small deployments. Mobile usability matters for observing a home server, but full terminal interaction is not needed for MVP.

### 12.3 Optional interactive terminal (LATER)

For direct remote debugging on the LAN, use SSH to the home server and inspect the sandbox or attach to a persistent tmux session. An in-browser terminal requires a separately authenticated, audited PTY gateway and command authorization; do not build it merely to show execution status. The native structured event timeline should be the primary observability interface.

---

## 13. Compose deployment using existing Traefik

### 13.1 Assumptions and host prerequisites

- A Linux home server with supported Docker + Docker Sandboxes/microVM prerequisites; validate CPU virtualization, kernel, storage, and installed `sbx` version before installation.
- Existing Traefik container already connected to a user-managed external Docker network (example name: `traefik`).
- User controls LAN DNS to resolve `factory.home.arpa` (replace with actual chosen domain).
- Traefik already has an appropriate HTTPS entrypoint and certificate/CA strategy for the internal hostname; a `.home.arpa` name will not simply receive a publicly trusted certificate through ordinary ACME HTTP validation.
- GitHub and model providers are reachable over permitted outbound network paths.
- Dedicated `factory` host service account with rights to run local `sbx`, but without broad sudo and without exposing privileged sockets to Compose containers.
- Secrets, including GitHub App private key and runner token, are provisioned outside version control.

### 13.2 Proposed root `compose.yaml`

**SKELETON — not tested application code.** This intentionally omits implementation-specific Dockerfiles, session secret wiring, API healthcheck, DB migration job, secret-manager integration, and production hardening. Use pinned versions/digests in a release.

```yaml
name: software-factory

services:
  web:
    build:
      context: .
      dockerfile: apps/web/Dockerfile
    restart: unless-stopped
    environment:
      INTERNAL_API_URL: http://api:3001
      APP_ORIGIN: https://${FACTORY_HOST}
    depends_on:
      - api
    networks:
      - frontend
      - traefik
    labels:
      - traefik.enable=true
      - traefik.docker.network=${TRAEFIK_NETWORK}
      - traefik.http.routers.factory.rule=Host(`${FACTORY_HOST}`)
      - traefik.http.routers.factory.entrypoints=${TRAEFIK_ENTRYPOINT}
      - traefik.http.routers.factory.tls=true
      - traefik.http.services.factory.loadbalancer.server.port=3000

  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile
    restart: unless-stopped
    environment:
      DATABASE_URL: ${DATABASE_URL}
      NODE_ENV: production
    ports:
      - "127.0.0.1:8787:3001"  # host runner ONLY; firewall/authorization too
    depends_on:
      db:
        condition: service_healthy
    networks:
      - frontend
      - database
      - egress

  db:
    image: postgres:17
    restart: unless-stopped
    environment:
      POSTGRES_DB: factory
      POSTGRES_USER: factory
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U factory -d factory"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - database

networks:
  traefik:
    external: true
    name: ${TRAEFIK_NETWORK}
  frontend:
    internal: true
  database:
    internal: true
  egress: {}    # API outbound to GitHub; no inbound Traefik route

volumes:
  postgres_data:
```

**Important caveats:**
- The API has both private and normal egress networks; with only an `internal: true` Docker network, GitHub API traffic may fail. Traefik never routes directly to the API or DB.
- `web` is on the existing Traefik network and a private frontend network, so its own outbound behavior must be audited.
- The example uses an environment variable for `DATABASE_URL`; in production, move secrets to an appropriate secret-management mechanism and never commit `.env`.
- Actual Compose network, TLS middleware, certificates, IP allowlist, and router priorities must match the existing Traefik instance.
- Apply authentication and authorization at the app even if Traefik has forward-auth middleware. Router labels do not automatically provide login protection.
- If the host already has other services on port `8787`, make runner loopback port configurable.
- Test whether `127.0.0.1` publication provides the expected isolation under the installed Docker/network stack; enforce host firewall rules as defense in depth.

### 13.3 Example `.env.example`

```dotenv
FACTORY_HOST=factory.home.arpa
TRAEFIK_NETWORK=traefik
TRAEFIK_ENTRYPOINT=websecure
DB_PASSWORD=CHANGE_ME
DATABASE_URL=postgresql://factory:CHANGE_ME@db:5432/factory
# APP_SESSION_SECRET=...       (provision securely)
# GITHUB_APP_ID=...            (provision securely)
# GITHUB_APP_PRIVATE_KEY=...   (prefer file/secret, not inline in .env)
# RUNNER_SHARED_TOKEN=...      (provision securely on API + host runner)
```

### 13.4 Host runner systemd service

```ini
[Unit]
Description=Software Factory Docker Sandbox Runner
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=simple
User=factory
Group=factory
WorkingDirectory=/opt/software-factory
EnvironmentFile=/etc/software-factory/runner.env
ExecStart=/usr/bin/node apps/runner/dist/index.js
Restart=always
RestartSec=5
NoNewPrivileges=true
UMask=0077

[Install]
WantedBy=multi-user.target
```

Install `sbx` for that service account and perform its separate Docker and model-provider headless authentication setup. Do not assume the human user's interactive `sbx` login automatically transfers to the service account. Use `systemctl status`/journal for operational troubleshooting. On startup the runner must inventory existing owned sandboxes before claiming new work.

### 13.5 Host directories

```text
/opt/software-factory/               # deployed application + config checkout
/etc/software-factory/runner.env      # protected service configuration
/var/lib/factory/repos/              # pristine factory-owned Git caches
/var/lib/factory/artifacts/          # bounded logs, reports, exports
/var/lib/factory/backups/            # encrypted/off-host backups preferred
```

Use strict file ownership and permissions. Prevent sandboxes from mounting `/etc/software-factory`, artifact backends, or the host runner's credentials. Any generated `sbxenv.yaml` with host-executed lifecycle settings must be stored outside writable sandbox mounts.

### 13.6 Bootstrap concept

The installation script should be idempotent and perform:

1. Verify OS/architecture, Docker, virtualization, `sbx` version, Traefik network, available ports, disk and memory.
2. Create host service account and protected directory structure.
3. Verify `sbx` authentication/agent kits with non-destructive probe commands; surface fixes rather than auto-accepting dangerous settings.
4. Generate `.env` from `.env.example` and assist with secret provisioning securely.
5. Validate YAML/agent registry/workflow schema and check GitHub App installations.
6. Build/start Compose; apply Drizzle migrations; verify DB/API/web health.
7. Install/enable runner systemd unit; perform runner handshake and capability probe.
8. Verify `factory.home.arpa` DNS, Traefik routing, TLS and app authentication from another LAN device.
9. Run optional disposable issue-to-PR smoke test on an expressly designated sandbox test repository.
10. Show backup/restore instructions and effective security settings.

No install script should globally expose Docker API ports or bypass host firewall configuration.

---

## 14. Security and authorization model

### 14.1 Threat model

Threats include malicious GitHub issue text/comments, prompt injection through repositories/dependencies, compromised package scripts, untrusted agent-authored code/tests, exposed API/model credentials, accidental GitHub branch or PR operations, a dashboard user with excessive privileges, and runaway cost/resource consumption. Docker microVM isolation is helpful but **not** sufficient to protect credentials deliberately placed in a sandbox or resources reachable through network policy.

### 14.2 Trust boundaries

**Trusted:** factory-owned config (after review), API/orchestrator, host runner, GitHub App private key, deployment secrets.

**Untrusted:** issue descriptions/comments, repository contents, dependencies, agent instructions imported from an unfamiliar repo, harness output, tool output, test/build processes, and agent-produced Git refs/names.

Make the runner incapable of executing arbitrary *operator-supplied* shell commands via HTTP. It should accept a restricted job enum and validated IDs/arguments; only factory-owned templates determine which executable runs. Project validation commands are approved config and should execute inside sandbox, not on the host.

### 14.3 Mandatory controls

1. **UI authentication:** signed, expiring sessions, secure cookies, CSRF defense, optional multi-user RBAC (`viewer`, `operator`, `admin`); LAN access alone is not authentication.
2. **Repository allowlist:** GitHub App installed on specified repos; task/issue cannot redirect execution to another repo.
3. **No broad agent GitHub token:** push, labels, and PR operations mediated by trusted runner/API; agent gets only necessary read access.
4. **GitHub branch protection:** require human reviews and configured checks; agent cannot merge/deploy. Require branches to be up to date before merging, or use a merge queue, so required checks run against the current base (§8.6).
5. **Per-stage capability enforcement:** immutable security policy maps to sandbox network/filesystem/tool restrictions. Unsupported deny semantics must block rather than be silently ignored.
6. **Network restrictions:** explicit model/provider/registry/required domains; deny access to host control plane, private LAN services, metadata IPs, and sensitive local endpoints unless approved.
7. **Resource/budget caps:** CPU, RAM, max runtime, max concurrent VMs, maximum attempts, provider rate limits and configurable spend ceilings.
8. **Prompt-injection resistance:** agent content cannot change workflows, grant permissions, authorize merges, or inspect secrets.
9. **No host shell injection:** fixed executables + argument arrays; sanitize refs and paths; verify branch/commit before trusted Git operations.
10. **Secret handling:** use documented `sbx` credential injection/host-side secret features where supported; never echo tokens into logs or prompts.
11. **Audit trail:** who queued, cancelled, approved, changed workflow, updated agent, imported file, or pushed PR and why.
12. **Redaction & retention:** redact logs at ingest and give short default retention for raw tool output.
13. **Server egress:** API needs GitHub access; DB does not; Traefik and UI exposure restricted to LAN.
14. **Credential rotation:** rotate GitHub App keys, runner token, session secret, and provider credentials without changing source code.

### 14.4 Read-only reviewer semantics

A “reviewer” agent declared read-only in Markdown is **not** actually read-only unless its harness and sandbox policy can enforce it. Put review in an isolated checkout with no trusted Git push credentials and restricted write/elevated tool access. Tests invoked during validation are also untrusted code and must stay inside the sandbox.

### 14.5 Authentication alternatives

- **Dashboard access:** existing Traefik SSO/forward-auth if available, *plus* application enforcement/identity; otherwise an app-local login/passkey/OIDC strategy. Do not require a third-party SaaS identity provider.
- **GitHub access:** private GitHub App, installation tokens, constrained repo permissions, short expiry.
- **Model access:** host-managed credentials or Docker Sandboxes secret provisioning; per-provider auth is separate from Docker authentication.
- **Runner-to-API:** restricted host loopback + dedicated bearer/mTLS-equivalent identity if desired + fenced job leases. Loopback alone is not sufficient authentication.

---

## 15. Failure recovery, reconciliation, cleanup, and backup

### 15.1 Why this is first-class

An unattended factory must survive host restarts, network loss, expired GitHub tokens, provider failures, runner crashes, partial Git pushes, stale job leases, and interrupted sandbox processes without duplicating work or deleting uncommitted changes. An execution should never be declared complete merely because a process disappeared.

### 15.2 Recovery algorithm on API restart

1. Load all nonterminal execution rows and current leases.
2. Pause dispatch until the runner registers and reports sandbox inventory (configurable startup grace period).
3. Reconcile GitHub state: existing branch/ref, PR, CI/checks, merge status, issue labels.
4. Reconcile the runner/sandbox observations with stored sandbox IDs and agent session IDs.
5. If process is alive and lease can be re-established without competing holder, continue monitoring.
6. If process exited but artifacts/commit are present, finish ingestion and continue the deterministic next stage.
7. If process existence is unknown, mark `RECOVERY_REQUIRED` and quarantine re-dispatch until probed or approved.
8. If sandbox is gone but verified commits/PR are durable, resume from last committed stage or re-run safely from a clean checkout.
9. If sandbox is gone and work was never committed/preserved, record potential lost work and request an explicit retry.
10. Emit a recovery timeline in events/audit log so the dashboard makes decisions transparent.

### 15.3 Runner startup inventory

On every service start, run `sbx ls`/supported inventory API, identify names with the factory-owned prefix and compare to DB. Never delete unknown third-party/user sandboxes. Report orphans for manual inspection, and only garbage-collect factory-owned sandboxes after verifying safe-to-delete criteria.

### 15.4 Cleanup gate

A sandbox may be removed only when:
- No active process is associated with it.
- Every required commit SHA has been verified and fetched/pushed to durable storage.
- Required artifacts/events are persisted.
- No job lease still expects it.
- The configured retention/cancellation policy permits deletion.

Expired sandbox disk pressure should trigger a dashboard alert and safe cleanup queue, not random removal of VMs. Retain failed-run sandboxes temporarily for debugging when disk permits.

### 15.5 Backups

**Back up:**
- PostgreSQL database (consistent logical dump or approved backup method).
- `config/agents/workflows/projects` Git repository, including deployed commit and locally uncommitted configuration if permitted.
- Artifact storage not reproducible from GitHub.
- Necessary secret *references* and secure re-provisioning instructions; encrypt actual secret backups separately.
- Deployment `.env` configuration under appropriate secure handling.

**Do not assume:** live sandbox microVMs are portable snapshots, or that a DB backup alone includes sandbox state. GitHub commits/branches/PRs are durable external artifacts; unpushed sandbox files are not.

**Restore sequence:**
1. Provision compatible host, Docker, `sbx`, Traefik network, DNS, TLS, model and GitHub credentials.
2. Check out the factory config at the backed-up version.
3. Restore Postgres and artifacts, apply version-compatible migrations only after validation.
4. Start Compose and host runner; keep dispatch paused.
5. Re-register GitHub project integrations and scan all active PRs/branches/issues.
6. Reconcile executions, marking missing sandboxes `RECOVERY_REQUIRED` instead of assuming they exist.
7. Resume only safe jobs, then manually release queued/recovery jobs as needed.
8. Run smoke tests and verify dashboard links and backup health.

Test restore to a different compatible host periodically. Store encrypted backups off the system or on independent storage; a backup on the same disk alone is not disaster recovery.

---

## 16. Repository and module structure

### 16.1 Recommended monorepo

Use pnpm workspaces (optionally Turborepo only if build orchestration clearly needs it). Keep a single, deployable factory repository whose config is plain files in Git.

```text
software-factory/
├── README.md
├── SOFTWARE_FACTORY_ARCHITECTURE.md       # this handoff document
├── package.json
├── pnpm-workspace.yaml
├── pnpm-lock.yaml
├── tsconfig.base.json
├── compose.yaml
├── .env.example
├── .gitignore
│
├── apps/
│   ├── web/                               # Next.js + Tailwind + shadcn/ui
│   │   ├── app/
│   │   │   ├── (dashboard)/overview/
│   │   │   ├── (dashboard)/fleet/
│   │   │   ├── (dashboard)/projects/
│   │   │   ├── (dashboard)/executions/[id]/
│   │   │   ├── (dashboard)/approvals/
│   │   │   ├── (dashboard)/agents/
│   │   │   └── api/                       # server-side reverse-proxy routes
│   │   ├── components/
│   │   └── Dockerfile
│   │
│   ├── api/                               # Hono + scheduler + GitHub sync
│   │   ├── src/
│   │   │   ├── app.ts
│   │   │   ├── routes/
│   │   │   ├── middleware/
│   │   │   ├── scheduler/
│   │   │   ├── github-sync/
│   │   │   └── sse/
│   │   └── Dockerfile
│   │
│   └── runner/                            # host-only Node.js process
│       ├── src/
│       │   ├── main.ts
│       │   ├── job-poller.ts
│       │   ├── sandbox-manager.ts
│       │   ├── process-manager.ts
│       │   ├── git-artifact-service.ts
│       │   └── startup-recovery.ts
│       └── package.json
│
├── packages/
│   ├── core/                              # deterministic workflow + state machine
│   ├── database/                          # Drizzle schema/migrations/queries
│   ├── github/                            # Octokit + REST/GraphQL + projections
│   ├── agent-registry/                    # Markdown parser, validator, compiler
│   ├── harnesses/
│   │   ├── common/
│   │   ├── claude/
│   │   ├── codex/
│   │   ├── opencode/
│   │   └── grok/
│   ├── sandbox/                           # sbx process/API adapter
│   ├── events/                            # normalized event schemas/parsers
│   ├── validation/                        # approved in-sandbox commands/checks
│   ├── config/                            # schema, load/merge/version snapshots
│   └── shared/                            # stable DTOs, types, Zod contracts
│
├── agents/                                # canonical agent definitions
│   ├── software-architect.md
│   ├── senior-developer.md
│   ├── qa-engineer.md
│   └── code-reviewer.md
│
├── workflows/                             # versioned stage graphs
│   ├── standard.yaml
│   ├── bugfix.yaml
│   └── documentation.yaml
│
├── config/
│   ├── projects.yaml
│   ├── defaults.yaml
│   ├── models.yaml
│   └── validation-profiles.yaml
│
├── infra/
│   ├── factory-runner.service
│   ├── traefik-labels.md                 # only integration guide, no new proxy
│   └── backups/
│
├── scripts/
│   ├── bootstrap.sh
│   ├── doctor.sh
│   ├── backup.sh
│   ├── restore.sh
│   ├── migrate.sh
│   └── smoke-test.sh
│
└── tests/
    ├── fixtures/                         # sample native agent definitions/events
    ├── integration/                      # mock GitHub/sbx and real DB
    ├── e2e/                              # opt-in live sandbox smoke tests
    └── recovery/                         # kill/restart/failure injection
```

### 16.2 Dependency boundaries

- `packages/core` depends only on pure types/ports, *not* Hono, Next.js, sbx CLI, GitHub SDK, or Drizzle schema implementation.
- `apps/api` implements ports via Postgres/GitHub services, handles authorization, starts scheduler and SSE.
- `apps/runner` implements sandbox/harness execution ports and sends structured results over authenticated local API.
- `packages/agent-registry` operates on files/text, compilers are provided by `packages/harnesses/*`.
- `apps/web` uses typed DTOs; does not import database driver or runner code.
- DB migration ownership is explicit (`packages/database`), and release startup checks schema version.

### 16.3 Build/dev commands (targets to implement)

```bash
pnpm install --frozen-lockfile
pnpm lint
pnpm typecheck
pnpm test
pnpm db:migrate
pnpm dev

# Deploy services once Dockerfiles/configuration exist:
docker compose --env-file .env up -d --build

# Host runner is separately built/installed under systemd:
pnpm --filter @factory/runner build
sudo systemctl enable --now factory-runner
```

These are **desired package scripts**: not evidence a repository containing these commands already exists.

---

## 17. Detailed implementation milestones

### Milestone 0 — Infrastructure and compatibility spike

**Build**:
- Validate server's Docker, Docker Sandbox, kernel/virtualization and `sbx` CLI version.
- Probe headless Claude authentication in an isolated disposable repo.
- Validate `sbx create --clone`, `sbx exec`, structured CLI output, commit, fetch, cleanup.
- Confirm Traefik network name, DNS, TLS and server account permissions.
- Define security policy and project repository allowlist.
- Record installed versions and any deviations in an implementation `COMPATIBILITY.md` or in this document's annex.

**Done when:** a runner-owned, unattended test task can produce a commit in a disposable sandbox and safely fetch it into a trusted host checkout without touching an interactive user's repo.

### Milestone 1 — End-to-end single issue-to-PR vertical slice

**Build**:
- Monorepo, Postgres/Drizzle, Hono API, runner token and systemd service.
- GitHub App installation auth, issue polling and one repo registry.
- One agent Markdown definition, one simple workflow, Claude adapter.
- Transactional queue, execution states, bounded logs, artifact collection.
- Safe branch push + PR creation + GitHub label projection.
- Basic Next.js execution list/detail with current state (polished fleet UI not yet required).

**Done when:** one explicitly labeled issue becomes a PR with stable execution ID, validated Git SHA, no duplicate dispatch, and no automatic merge.

### Milestone 2 — Visual control room and observability

**Build**:
- SSE durable event stream and replay.
- Fleet overview, projects screen, execution timeline, approval inbox.
- Runner/sbx health, tool events, token/provider usage when available, error classification.
- Cancellation/retry actions with audit trail and RBAC.
- Log redaction and retention.

**Done when:** operator can diagnose agent state and blockers without manually attaching to the sandbox.

### Milestone 3 — Multi-project + robust workflow engine

**Build**:
- Project-specific concurrency/fair scheduling, global limits and budgets.
- Planner/QA/reviewer agent definitions, stage dependencies and repair loop.
- Independent review environment, GitHub required-check tracking.
- Versioned workflow and config snapshot semantics.
- Strong heartbeat/leases/reconciliation on restarts; failure injections.

**Done when:** at least two repos can run concurrent isolated work without cross-repo credential/path leaks or double execution after restart.

### Milestone 4 — Harness compatibility, agent registry, and portability

**Build**:
- Codex adapter with structured parser; OpenCode adapter and probes; optional Grok custom kit.
- Native Markdown/TOML importer/exporter with diagnostics and schema validation.
- Agent editor + harness capability screen and per-project configuration.
- Bootstrap/doctor, secure secret provisioning, backups, and tested restore.
- CLI version compatibility matrix and sandbox kit pinning.

**Done when:** a new compatible host can deploy from Git/config + restored DB/artifacts, and a workflow can select different agent definitions backed by different verified harnesses.

### Milestone 5 — Optional expansions, in priority order

1. Read-only planning agent that proposes issues for human approval.
2. DAG fanout and parallel subtasks with integration coordinator.
3. Automatic agent/model selection based on cost/performance/quality signals.
4. Local/hosted model routing where supported by harness/sbx and policy.
5. Optional Projects v2 status projection and richer planning views.
6. Authenticated in-browser PTY debugging gateway (if compelling).
7. Remote runners/other sandbox providers behind the same adapter interfaces.
8. Multiple operators and fine-grained project-level RBAC.
9. Host resource scheduling and more extensive cost analytics.

---

## 18. Verification, quality gates, and acceptance criteria

### 18.1 Unit tests

- YAML/frontmatter parsing and strict agent schema validation.
- Native agent compilation, preservation of instruction body, warnings for unsupported fields.
- Harness resolution (supported kit, auth, model, fallback, capabilities) and fail-closed rules.
- Workflow DAG validation and stage transition invariant enforcement.
- Transition evidence (§9.6): each transition refuses when its evidence is missing, written for another SHA, or malformed; a missing or unknown verdict never counts as a pass.
- Stage routing (§5.3): `pathSurfaces` adds stages but never removes them; `default` applies without a triage result; at least one review stage always runs.
- GitHub repo/issue identifiers, branch and sandbox name sanitization.
- Retry budgets, concurrency admission, fairness, budget exhaustion.
- Native stream parsers with chunk splits, invalid JSON, unknown event types, huge lines, UTF-8 boundaries.
- Event redaction, SSE cursor/replay, unknown costs.

### 18.2 Database and API integration tests

- Two simultaneous scheduler processes cannot claim the same issue/job.
- Lease expiry does not automatically create another active sandbox if the first is still running.
- Stale fencing token cannot write terminal results.
- Duplicate event batches are idempotent.
- Retrying `create PR` after a simulated timeout finds/adopts existing PR instead of duplicating it.
- Invalid API session/CSRF/runner token receives rejection.
- GitHub App permission denial blocks precisely and surfaces actionable errors.
- SSE reconnect with Last-Event-ID replays exactly the missed event range.

### 18.3 Sandbox and Git end-to-end tests

- Create `sbx --clone`, run a harmless agent task, produce commit, fetch expected branch, push to disposable GitHub repo, create draft PR.
- No agent edits appear in unrelated host repo checkout.
- Parallel executions from same project produce independent branches without shared working tree changes.
- Agent cannot merge PR or mutate unrelated repos with its given credentials.
- Sandbox cannot access forbidden host LAN endpoint; approved provider/package endpoints work.
- Removing a sandbox before artifact persistence is blocked.
- Codex/Grok/OpenCode adapters fail gracefully when not installed/authenticated.
- A native subagent file merely present on disk is insufficient; the selected agent's resolved instructions must be visible in the execution snapshot and used in the actual command/session.

### 18.4 Failure-injection tests

Kill/restart each of: Next.js, Hono API, host runner, `sbx` sandbox, and entire host (where practical). Also simulate GitHub rate limiting, expired App tokens, provider timeout, invalid JSON events, disk full, malformed frontmatter, lost SSE connection, and PR creation partial success. Verify no duplicate active execution; state is recovered or `RECOVERY_REQUIRED`; no unpushed branch is silently removed.

### 18.5 UX acceptance tests

- Overview shows all active projects and agent instances in one view.
- Filtering by repo/agent/harness works, and blocked/error states are visible without navigating to terminal.
- Selecting an instance shows issue, stage, event history, Git SHA, test outcomes, PR link, last activity, and real failure reason.
- Operator can queue/cancel/retry/resolve approval with audit history.
- Agent Registry imports a Claude Markdown agent and accurately reports compilation to Codex TOML.
- Model/harness settings are clearly subordinate to the user-facing **Agent** concept.
- When usage/cost data is unavailable, UI says so; no fake token counts or progress bars.
- App is accessible from another LAN device through existing Traefik and not inadvertently reachable from the Internet.

### 18.6 Definition of done for MVP

A single compatible Linux host can deploy the stack behind existing Traefik; GitHub Issues label triggers one safe, isolated Claude execution; stdout is normalized into durable events; branch is committed and PR created using trusted credentials; operator sees status/live logs in dashboard; auth, cancellation, retry limits, and one restart/recovery scenario work; human controls merge. No Gas City, Plane, or new reverse proxy is installed.

---

## 19. Remote access to an existing interactive Claude sandbox

This requirement was part of the original design motivation and should not be lost even though the factory's primary workload is **headless execution**.

For LAN-only manual debugging, SSH to the home server, then use existing `sbx` CLI under the correct user account. Prefer `tmux` or an equivalent persistent terminal multiplexer to retain the interactive `sbx run` foreground session across SSH disconnects:

```bash
# From another machine on the LAN:
ssh user@home-server-lan-ip

# On host; see configured sandbox names:
sbx ls

# Attach to an existing named sandbox's agent session (per installed CLI):
sbx run --name my-project

# Or enter another command shell in the sandbox (not necessarily the same
# interactive Claude conversation):
sbx exec -it my-project bash
```

`sbx exec` creates a separate command process; it does **not** automatically reconnect to the same interactive Claude session. To preserve the original SSH-managed terminal, create and keep the `sbx run` process under tmux from the outset. Docker's optional Claude Remote Control may offer browser/mobile convenience, but uses an external Anthropic connection and therefore **is not the LAN-only default** for this project.

The factory dashboard should not need to adopt the user's existing interactive sandbox automatically. Treat imported/interactive sandboxes separately from owned headless jobs until an explicit import workflow and safety/recovery semantics are designed.

---

## 20. Open implementation choices and risk register

These do **not** invalidate the accepted high-level architecture. They are deliberately identified decisions an implementing agent should verify or propose clearly before baking behavior into code.

| Topic | Preferred starting direction | Main risk / validation |
|---|---|---|
| Linux host compatibility | Use existing home server | Ensure Docker Sandbox VM, daemon, OS architecture, disk, and resource headroom. |
| `sbx` version | Detect/pin documented supported version | CLI/kit/environment file flags can change. |
| Reproducible sandbox config | CLI adapter first; `sbxenv.yaml` after spike | Environment files are experimental and can run host lifecycle commands. |
| Native agent invocation | Prompt-bundle fallback always available | Different harnesses use native subagents differently. |
| Codex custom agent/TOML | Compile supported fields | Confirm native headless selection in installed Codex version. |
| Grok Build | Custom kit/adapter later | No verified Docker built-in `grok` kit. |
| UI authentication | Existing Traefik SSO if available + app auth | No implicit assumption about user's Traefik middleware. |
| API topology | Hono separated from Next.js | Can consolidate for prototype; avoid coupling scheduler to web requests. |
| DB queue | Postgres transactional leases | Reconcile stale leases before re-dispatch; ensure fencing. |
| Artifact storage | Filesystem + DB metadata | Backup and retention policy; optional object store later. |
| PR update behavior | Push exact vetted commit to unique branch | Credential leaks, duplicate PRs, malicious Git refs. |
| GitHub Projects v2 | Optional, discover/install after basic Issues work | App permissions and GraphQL complexity. |
| Runner host privileged role | Dedicated service account | `sbx` remains powerful host-controlled capability. |
| Model budgets | Hard time/attempt caps first, provider cost only if measured | Tokens/cost not consistently available across harnesses. |
| Automatic agent choice | Fixed configured agent by stage | Auto-routing could silently choose a less capable/unexpected harness. The deterministic risk/surface table (§5.3) is still fixed configuration, not automatic choice. |
| Multi-user control | One trusted operator MVP | RBAC and approval identity before wider LAN user access. |
| Parallel subtasks | Future DAG with worktree isolation | Merge conflicts and shared state multiply complexity. |

**Risk ranking**:
1. **High:** safe host/VM/credential boundary, preserving commits before cleanup, duplicate execution after crash, pushing wrong ref.
2. **High:** mismatch between native agent file support and headless stage invocation.
3. **Medium:** resource pressure from several microVMs/builds on home hardware.
4. **Medium:** GitHub token expiry/rate limits and eventual consistency.
5. **Medium:** upstream `sbx`/harness API churn; mitigate with probes, adapters, version pinning.
6. **Medium:** prompt injection and malicious scripts; mitigate with isolation, allowlists, no privileged agent credentials.
7. **Low-to-medium:** UI streaming/proxy buffering; test with existing Traefik stack.

---

## 21. Instructions for the next implementation agent

**Treat this as an architecture specification and agreed product vision, not an existing codebase.** Do not claim it is deployed, tested, or implemented until you have built and verified it.

1. **Preserve the non-negotiables**: existing Traefik, GitHub Issues/PRs, no Gas City/Plane/Caddy, custom orchestrator, Docker Sandboxes, agent-first Markdown definitions, LAN-only inbound access, real crash recovery, human merge approval.
2. **Start with an environment audit** (OS, Docker, `sbx` version, Traefik Docker network, GitHub App permissions), then write an implementation plan anchored to the verified CLI and docs.
3. **Create a minimal vertical slice** before building advanced dashboards or model routing. The success criterion is an approved GitHub issue turning into a PR from an isolated, headless sandbox with reliable event tracking.
4. **Do not silently change terminology**: *agent* is the specialist definition; *harness* is Claude Code/Codex/OpenCode/Grok Build; *model* is the inference endpoint; *runtime* is Docker Sandbox. Different agent **instances** may share one agent definition.
5. **Avoid new dependencies by default**. If recommending a new package/service, explain the concrete failure it solves and why existing Node/Postgres/Compose machinery is insufficient.
6. **Mark unsupported upstream capabilities as unavailable** until exercised by a probe; never assume markdown frontmatter fields are portable beyond the common core.
7. **Make security controls real**: permission constraints in an agent Markdown file are instructions, not enforcement. Use external GitHub branch protection and trusted control-plane auth.
8. **Use idempotent, durable orchestration**. Every external operation (sandbox creation, push, PR) needs a stable idempotency/reconciliation story.
9. **Write tests alongside adapter implementations**, especially parser fixtures, GitHub partial failures, and runner crashes.
10. **Keep the product portable**: configuration/version snapshots in Git, Postgres backup, artifact backup, explicit systemd runner provisioning, pinned image/kit versions, restore runbook.
11. **Prefer explicit accepted assumptions to repeated questions**. Make configuration tunable, expose errors, and proceed with safe defaults for the initial single-host implementation.
12. **Document deviations** from this specification as new Architecture Decision Records (ADRs) rather than silently replacing accepted decisions.

### Proposed first coding task for another agent

> Scaffold the pnpm monorepo and implement a Docker Sandbox/Claude headless compatibility probe plus a secure host runner interface. Add unit tests for `AgentDefinition` Markdown parsing and `HarnessAdapter` event normalization. Configure a minimal PostgreSQL schema for projects/tasks/executions/jobs/events. Do not yet implement autonomous PR merging or a complex dashboard. Validate every `sbx` command against the installed version and produce the first verified Git commit from a sandbox clone.

---

## 22. Authoritative references and verification checklist

These links are official product documentation used for the architecture review. The code/config in this document is **proposed factory application code** unless explicitly identified as documented upstream CLI usage. Verify CLI/SDK behavior against installed versions before implementation.

### Docker Sandboxes

- Docker Sandboxes overview: https://docs.docker.com/ai/sandboxes/
- Supported built-in agents: https://docs.docker.com/ai/sandboxes/agents/
- `sbx` command reference: https://docs.docker.com/reference/cli/sbx/
- `sbx create`: https://docs.docker.com/reference/cli/sbx/create/
- `sbx exec`: https://docs.docker.com/reference/cli/sbx/exec/
- Usage, CLI args, workspace modes: https://docs.docker.com/ai/sandboxes/usage/
- Git and clone-mode behavior: https://docs.docker.com/ai/sandboxes/workflows/git/
- Headless/CI automation: https://docs.docker.com/ai/sandboxes/workflows/automation/
- Sandbox architecture and credential proxy: https://docs.docker.com/ai/sandboxes/architecture
- Default network and workspace security: https://docs.docker.com/ai/sandboxes/security/defaults/
- Sandbox environment files (experimental): https://docs.docker.com/ai/sandboxes/configuration/environment-files
- Model selection (experimental): https://docs.docker.com/ai/sandboxes/configuration/models/
- Sandbox kits SDK: https://docs.docker.com/ai/sandboxes-api/cookbook/add-tools-with-kits/
- Release notes: https://docs.docker.com/ai/sandboxes/release-notes/

### Agent profiles and coding harnesses

- Claude Code custom subagents: https://code.claude.com/docs/en/sub-agents
- Anthropic's explanation of CLAUDE.md versus subagents: https://claude.com/resources/articles/steering-claude-code-skills-hooks-rules-subagents-and-more
- OpenCode agents and Markdown frontmatter: https://opencode.ai/docs/agents
- Cursor subagents: https://cursor.com/docs/subagents
- GitHub Copilot custom agents: https://docs.github.com/en/copilot/concepts/customizations/custom-agents
- Codex custom subagents/TOML: https://learn.chatgpt.com/docs/agent-configuration/subagents
- Codex configuration reference: https://learn.chatgpt.com/docs/config-file/config-reference
- xAI Grok Build headless CLI: https://docs.x.ai/build/overview

### GitHub, Traefik, and infrastructure

- GitHub Issues REST API: https://docs.github.com/en/rest/issues/issues
- GitHub labels REST API: https://docs.github.com/en/rest/issues/labels
- GitHub Projects v2 GraphQL reference: https://docs.github.com/en/graphql/reference/projects
- GitHub App installation authentication: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation
- GitHub App installation token generation: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app
- GitHub Actions workflow runs: https://docs.github.com/en/rest/actions/workflow-runs
- Traefik Docker provider: https://doc.traefik.io/traefik/providers/docker/

### Version gate at build time

Record the actual versions of Node, pnpm, Docker Engine, Docker Sandboxes, Claude Code, Codex, OpenCode, Grok Build (if used), Traefik, PostgreSQL, and Next.js in release metadata. A runner probe should fail clearly when an installed harness no longer emits the expected structured event shape or an unsupported version lacks a required feature. Build documentation should not assume the upstream interfaces remain frozen.

---

## 23. Final acceptance statement

**The finished product is a portable, self-hosted AI software factory that: (a) runs entirely on the user's LAN for its control plane; (b) uses existing Traefik; (c) uses GitHub for repositories, issues, optional boards, PRs, and CI; (d) runs reusable Markdown-defined specialist agents through selected harnesses in isolated Docker Sandboxes; (e) has a deterministic TypeScript scheduler/runner with durable state and event history; (f) visualizes all agents and projects in a custom Next.js dashboard; (g) survives realistic restarts/failures; and (h) never bypasses the operator's explicit approval for merging/deploying.**

If any implementation proposal cannot meet those properties, document the tradeoff and seek an explicit architectural decision before changing the contract.
