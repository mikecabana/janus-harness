# Milestone 4 — More harnesses, the agent registry, and portability

**Goal:** agents backed by different harnesses in one workflow, config editable from the dashboard, and a deployment that can be rebuilt on another host.

**Exit condition (§17 Milestone 4):** a new compatible host can be deployed from Git and config plus a restored database and artifacts, and a workflow can select different agent definitions backed by different verified harnesses.

**Order:** M4.1 first (it defines the model registry the adapters use). M4.2–M4.4 each stand alone. M4.5 → M4.6. M4.7 → M4.8.

Every harness in this milestone starts with its own probe, as Claude did in M0.4. No adapter is written against documentation alone.

---

## M4.1 — Model registry and agent resolution

**Build**

- `config/models.yaml`: aliases to exact upstream model IDs per harness, versioned.
- The full resolution algorithm in §6.5: allowed harnesses and models from policy; preferred harness or an explicitly allowed fallback; verification of the installed CLI, kit, version, auth and required features; exact model resolution that **fails** when the model is unavailable and never substitutes another.
- `GET /api/v1/harnesses` returning live probe results.
- The resolved harness, model, kit and CLI versions recorded in the execution snapshot.

**Tests:** resolution table tests, including every fail-closed case (§18.1).

**Spec:** §6.4, §6.5.

---

## M4.2 — Codex adapter

**Build**

- Probes: `codex exec --json` framing, write and sandbox flags, session resume, whether a custom agent can be selected headlessly, auth under the factory account.
- `packages/harnesses/codex`: launch, normaliser, resume if supported, and the TOML compilation in §6.3 limited to fields the installed schema accepts.
- Recorded fixtures under `tests/fixtures/codex/<version>/`.
- Passes the shared harness contract suite from M1.9.

**Done when:** an execution completes with the implement stage on Codex, and the adapter reports "not installed" or "not authenticated" cleanly when that is the case (§18.3).

**Spec:** §6.3, §6.6, §7.5.

---

## M4.3 — OpenCode adapter

**Build:** as M4.2 for OpenCode: probe the installed version's headless interface and agent format, implement the adapter, record fixtures, pass the contract suite.

**Spec:** §6.1, §6.6, §7.5.

---

## M4.4 — Grok Build (optional, gated)

**Build**

- A custom kit packaging the Grok CLI, since Docker ships none; credential injection through the `xai` service; network rules declared by the kit.
- An adapter using prompt-bundle injection only. No native agent file is assumed.

This package is built only if the operator wants Grok. It can be dropped without affecting the milestone's exit condition.

**Spec:** §6.1, §6.6, §7.5.

---

## M4.5 — Importer and exporter

**Build**

- Import from `.claude/agents`, `.opencode/agents`, `.cursor/agents`, `.github/agents` and `.codex/agents` following the eight steps in §6.2: hardened parsing with size limits, normalisation that keeps the source metadata, warnings for unsupported or ambiguous fields, instructions preserved exactly, name-conflict detection, a compatibility report (full, partial, unsupported).
- Export only validated native configurations. A requested deny is never emitted as an allow.
- `POST /api/v1/agents/import` with a preview step before anything is written.
- `GET /api/v1/agents/:id/compatibility`.

**Tests:** fixtures of real native files per format; round-trip preservation of the body; the §18.5 case "import a Claude Markdown agent and report its Codex TOML compilation accurately".

**Spec:** §6.1, §6.2, §11.2.

---

## M4.6 — Registry and workflow editing in the dashboard

**Build**

- Agent registry pages (§12.1 F): browse, view instructions, model and harness, capabilities, and per-harness compatibility.
- A guarded editor for agent Markdown and frontmatter with a parsed-schema preview. Saving validates, writes the file in the config directory, and creates a Git commit attributed to the operator ([ADR 0004](../adr/0004-config-source-of-truth.md)).
- The same write path for workflows (per-stage agent assignment, timeouts, approval policy) and for projects (register, change safe settings), which fills in `POST /projects` and the wider `PATCH /projects/:id` from §11.2.
- Harness capability screen and a "Test harness" action that runs the probe in a disposable sandbox (§12.1 G).
- Model and harness shown as advanced properties of an agent, never as the primary choice (§18.5).

**Tests:** an invalid edit is rejected and writes nothing; a valid edit produces exactly one commit and an audit entry; a running execution is unaffected by an edit.

**Spec:** §5.2, §11.2, §12.1 F and G, ADR 0004.

---

## M4.7 — Bootstrap and doctor

**Build**

- `scripts/bootstrap.sh` following the ten steps in §13.6: idempotent, never exposes a Docker API port, never changes firewall rules, never auto-accepts a dangerous setting.
- `scripts/doctor.sh` extended from M0.2 to cover the full deployment: Compose health, migrations, runner handshake, harness probes, DNS and TLS for the dashboard hostname.
- Secret provisioning guidance and rotation steps for the GitHub App key, runner token, session secret and provider credentials (§14.3 item 14).
- A version matrix written into release metadata (§22), and kit version pinning.

**Done when:** bootstrap run twice in a row makes no changes the second time.

**Spec:** §13.6, §14.3 item 14, §22.

---

## M4.8 — Backup and restore

**Build**

- `scripts/backup.sh`: a consistent Postgres dump, the config directory at its commit, non-reproducible artifacts, and a manifest of secret references (never the secrets). Encryption of the archive.
- `scripts/restore.sh` following the sequence in §15.5: restore with dispatch paused, reconcile, mark missing sandboxes `RECOVERY_REQUIRED`, release work manually.
- Backup status in Settings (§12.1 G).
- A restore runbook, and one recorded test restore onto a different compatible host or a clean VM.

**Done when:** the Milestone 4 exit condition is demonstrated: a clean host restored from backup, and one workflow run that uses two harnesses.

**Spec:** §15.5, §12.1 G.
