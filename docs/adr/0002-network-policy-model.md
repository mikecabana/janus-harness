# ADR 0002 — Network policy is machine-level, not per `SandboxSpec`

**Status:** Accepted 2026-11-13
**Affects:** §7.2 (network bullet), §7.3 (`SandboxSpec.allowedHosts`), §14.3 item 6

## Context

The spec's `SandboxSpec` carries `allowedHosts: string[]`, implying each sandbox is created with its own allowlist.

Local `sbx` does not work that way:

- `sbx create` accepts `--deny-network` only. `--allow-network` is for cloud sandboxes.
- Allow rules come from a machine-wide **preset** (`allow-all`, `balanced`, `deny-all`) plus rules added with `sbx policy`, either global or scoped to one sandbox by name. Kits add the rules their agent needs.
- Under Balanced and Locked Down, a request no rule matches is blocked and becomes a pending approval.
- The Balanced baseline allows AI provider APIs, package registries, OS repositories, code hosts and container registries.
- With a `github` secret stored, the host proxy injects it into requests to `github.com` and `api.github.com`.

So GitHub is reachable from a sandbox under Balanced, and the spec's rule "no broad agent GitHub token" (§14.3 item 3) holds only if no `github` secret exists for the account that runs the factory's sandboxes.

## Decision

1. **Preset.** The `factory` account uses the **Balanced** preset.
2. **No GitHub secret.** The factory account never stores an `sbx` secret for the `github` service. `scripts/doctor.sh` and the runner's startup check fail if one is present.
3. **Who pushes.** Unchanged from the spec: the host runner pushes and the API opens the PR. The agent commits locally and holds no GitHub credential.
4. **`SandboxSpec`.** `allowedHosts` is removed. It is replaced by:
   - `denyHosts: string[]`, passed as `--deny-network` at creation;
   - `extraAllowHosts: string[]`, taken from the project's trusted config and applied as sandbox-scoped allow rules after creation.
5. **Blocked requests.** They are reported as events and shown in the dashboard (M2.7). The factory never approves a pending request by itself. Allowing a host is a change to the project's config.
6. **Factory policy** may set a stricter preset later without code changes; the adapter reads the active preset and reports it.

## Consequences

- `npm install` and similar work without per-project setup.
- Agents can read public GitHub content anonymously. They cannot read private repositories or push anywhere.
- A project with private GitHub dependencies cannot install them inside the sandbox. A read-only token scoped to one sandbox is deferred (see `docs/plan/05-later.md`).
- Egress under Balanced is wider than a hand-written allowlist. This is accepted for a single-operator home deployment and is revisable through item 6.

## To verify (M0.3 probes 9 and 10)

- The syntax and behaviour of a sandbox-scoped allow rule. Fallback: global rules built from the union of enabled projects' `extraAllowHosts`.
- What a blocked request looks like in `sbx policy log --json` and `sbx policy approval ls`.
- That a sandbox cannot reach the runner's loopback API port or LAN addresses under Balanced.

## Not chosen

- **Locked Down preset with a curated allowlist per project.** Tighter, but every project needs its registries listed before it can build, and blocked-request triage becomes routine.
- **Letting the agent push with a stored GitHub token.** Less factory code, but GitHub tokens cannot be limited to one branch, so branch protection would be the only guard against a prompt-injected agent.
