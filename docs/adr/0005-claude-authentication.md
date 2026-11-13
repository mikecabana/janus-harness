# ADR 0005 — Claude subscription by default, API key optional

**Status:** Accepted 2026-11-13
**Affects:** §6.5 step 4, §14.5 (model access), §20 (model budgets)

## Context

The spec says model access uses "host-managed credentials or Docker Sandboxes secret provisioning" and leaves the mode open. `sbx` supports two modes for Claude Code, and in both the credential stays on the host:

- **Subscription OAuth.** A host-side sign-in; the proxy handles the token.
- **API key.** `sbx secret set anthropic`; the proxy injects it.

## Decision

1. The Claude harness adapter treats the auth mode as configuration: `subscription` (default) or `api-key`.
2. The harness probe reports which mode is active and fails clearly when the configured mode is not usable.
3. Budget controls that work in both modes are the primary ones: wall-clock time, idle time, attempts, and concurrency.
4. In `api-key` mode the adapter also passes `--max-budget-usd` per stage and records reported cost.
5. In `subscription` mode the dashboard shows cost as "unavailable" unless the stream reports it. Nothing is estimated.

## Consequences

- The factory's runs share the subscription's rate limits with the operator's interactive use. A rate-limit response is classified as a transient provider failure with backoff, not as an agent failure.
- The `factory` UNIX account needs one interactive Claude sign-in before unattended runs work.
- Switching to an API key is a config change and a stored secret, with no code change.

## To verify (M0.4 probes 1 and 6)

- Subscription OAuth works unattended under the `factory` account across sandbox restarts and token refresh.
- Whether usage or cost appears in the stream in subscription mode, and whether `--max-budget-usd` has any effect there.

## Not chosen

- **API key only.** Real spend caps and cost figures, but a separate bill from the plan already in use.
- **Subscription only.** Least work, but no path to hard spend caps.
