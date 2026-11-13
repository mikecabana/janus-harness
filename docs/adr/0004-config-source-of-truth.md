# ADR 0004 — Files are the source of truth for configuration

**Status:** Accepted 2026-11-13
**Affects:** §5.2, §10.1 (`projects`, `workflows`, `agent_definitions`), §11.2 (`POST /projects`, `PATCH /projects/:id`, `POST /agents/import`), §12.1 F

## Context

The spec describes projects, workflows and agents twice: as YAML and Markdown files in Git (§5), and as database tables with mutation endpoints and a dashboard editor (§10.1, §11.2, §12.1 F). It says "Git remains canonical for content" for agent definitions only, and does not say which side wins elsewhere.

Two writable copies would need conflict rules, and a config change made in the database would not be reviewable in Git.

## Decision

1. **Files win.** `config/*.yaml`, `workflows/*.yaml` and `agents/*.md` are the only source of truth. The directory is set by `FACTORY_CONFIG_DIR` and defaults to the repository root, so it can become a separate repository later.
2. **The database holds an index.** `projects`, `workflows` and `agent_definitions` are written only by the config sync, with file checksums and a version that increases when a checksum changes.
3. **Snapshots are immutable.** Each execution stores its fully resolved configuration (§5.1). Later file changes never affect it.
4. **Runtime state stays in the database.** A project's enabled or paused flag and its sync cursor are operational state, not configuration, and are changed through the API.
5. **Until Milestone 4** the dashboard shows configuration read-only. `PATCH /projects/:id` accepts enable and pause only. `POST /projects` and `POST /agents/import` are not available.
6. **From Milestone 4** (M4.5, M4.6) dashboard edits validate the change, write the file, and create a Git commit attributed to the operator, with an audit entry. The database is still updated only by the sync.

## Consequences

- Every configuration change is a reviewable diff with history.
- Registering a project before Milestone 4 means editing `config/projects.yaml`.
- Invalid files are rejected by the sync with an error naming the file and field; the previous valid version stays in effect and the dashboard shows the error.

## Not chosen

**Database as the source of truth with files as import and export.** Dashboard editing from the start, but more UI and migration work early and no Git review of configuration.
