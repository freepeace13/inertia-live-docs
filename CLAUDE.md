# inertia-live-docs

Documentation for Inertia Live, and the **source of truth** for the release program across all repositories. Plain Markdown, flat layout, no build step.

## What lives here

- **Versioned guides and reference** in a folder per major: `v1/` (installation, topics, projectors, bindings, authorization, vue, react, testing, configuration, client-core, consistency, troubleshooting, design-decisions, SPEC; `v1/README.md` is its index). Edit docs for the line you are changing. When the next major starts, create `v2/` as a copy of `v1/` (D-013).
- **Unversioned project docs at the root:** `README.md` (index and version table), `release-policy.md` (versioning, branches, support window, dist-tags, docs versioning, flow), `roadmap.md` (phases to 1.0, definition of done, open questions), `decision-log.md` (numbered decisions, D-001 and up).
- **Root stubs:** one-line files at the old root paths (`installation.md` and so on) that point into `v1/`. They only keep already-published links alive. Do not edit them or add new ones; delete them after the next release.
- `/docs/` is gitignored: generated specs and plans stay local (D-014).

A private progress tracker mirrors the roadmap and decision log: https://claude.ai/artifact/PjKwna6Tty3ZnZFKneBjhT (owner access; checklist state lives in its database). **Order of updates:** change these docs first, then the tracker, in the same piece of work. If they disagree, these docs win.

Sibling repos (local checkouts next to this one): `../inertia-live` (client packages), `../inertia-live-laravel` (server adapter), `../inertia-live-tooling` (release script, workflows, Claude plugin).

## Rules for this repo

- A change to behavior, config, API or the wire protocol in either package updates the matching doc here in the same piece of work. The protocol spec is `SPEC.md` plus `consistency.md`; keep both repos' behavior and these docs consistent.
- Decisions: append to `decision-log.md` with the next number; to change one, add a new entry and mark the old one superseded. Do not rewrite history.
- Keep each version's `README.md` guide and reference tables in sync when adding or renaming a file in that folder. Use relative links between docs; link to project docs from a version folder with `../`.
- Link to a versioned page from other repos as `https://github.com/freepeace13/inertia-live-docs/blob/main/v1/<file>.md`, never the root path.
- Write plainly. Prefer short sentences and concrete examples that are tested against the demo or the packages.
- **No AI attribution** anywhere: no `Co-Authored-By` trailers, no "Generated with" footers, no mentions in commits, PRs or docs.

## Status

The packages are at 0.1.x (client 0.1.1, Laravel 0.1.0). The 1.0 scope is not yet defined (D-011); see `roadmap.md` before documenting anything as stable.

## Keeping the tracker in sync

Tracker: https://claude.ai/artifact/PjKwna6Tty3ZnZFKneBjhT. Update it whenever you finish, start, block or add work, or record a decision or note, in the same piece of work. Never create a second tracker artifact.

- **State** (phases, tasks, decisions, notes) lives in the artifact's database. Use the `ArtifactData` tool with the tracker URL; pin writes to documents you have read (`if_version`). Task ids are stable slugs (`t-...`); statuses are `todo`, `doing`, `done`, `blocked`.
- **Page text or layout:** read the artifact (`Artifact` with `action: "read"` and the URL) to get its current HTML, change it, and publish with `url` set so the link stays the same.
- **Order:** docs repo first (`decision-log.md`, `roadmap.md`, `release-policy.md`), then the tracker. The docs win on conflict.
