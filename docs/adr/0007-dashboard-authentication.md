# ADR 0007 — App-local single operator, sessions owned by the API

**Status:** Accepted 2026-11-13
**Affects:** §3.2 (`web` responsibilities), §11.1, §14.3 item 1, §14.5

## Context

The spec lists options for dashboard access (existing Traefik SSO plus app enforcement, or app-local login, passkey or OIDC) without choosing (§14.5, §20). It also places "authentication/session management" in `web` (§3.2) while requiring the API to authenticate UI calls (§11.1), which leaves it unclear which process validates a session.

## Decision

1. **One operator account**, local to the app. Its password hash is set at bootstrap, using Node's built-in `crypto` (argon2id where the installed Node provides it, otherwise scrypt) so that no native add-on is needed.
2. **The API owns sessions.** Login, logout, session validation and CSRF checks are API endpoints and middleware. Sessions are rows in Postgres; the cookie is signed, expiring, `HttpOnly`, `Secure` and `SameSite=Strict`.
3. **The web app does not validate sessions.** It forwards same-origin `/api/*` requests to the API and redirects to the login page when the API answers 401.
4. **Roles exist from the start.** `operators.role` is one of `viewer`, `operator`, `admin`, and every mutating endpoint declares the role it needs. The single account is `admin`.
5. Login attempts are rate limited, and every login and logout is audited.

## Consequences

- One place decides who is signed in, so the web proxy cannot drift from the API.
- No dependency on an identity provider or on the operator's Traefik middleware.
- Adding more operators later is a data change plus a management screen, not a redesign.

## Not chosen

- **Traefik forward-auth** with trusted identity headers. Worth it only where an SSO is already running; deferred (see `docs/plan/05-later.md`).
- **Both paths in the MVP.** More to build and test for one user.
