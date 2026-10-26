# Decision log

Numbered decisions about how Inertia Live is released and what 1.0 means. Design decisions about the library itself (for example, per-topic signal versions) live in [Design decisions](design-decisions.md). Add new entries at the bottom with the next number; to change a decision, add a new entry and mark the old one superseded. The [roadmap tracker](https://claude.ai/artifact/PjKwna6Tty3ZnZFKneBjhT) mirrors this log; this file wins on conflict.

## D-001. Pre-1.0 versioning: minor is breaking
Status: decided, 2026-10-27.
The API is still moving. A minor bump may break; a patch is a fix or docs only. Consumers pin with `^0.N`.

## D-002. Client packages release in lockstep
Status: decided, 2026-10-27.
`core`, `vue` and `react` are thin and tightly bound (`vue` and `react` depend on `core` via `workspace:^`). All three share one version and one tag. A docs-only fix republishes all three; revisit with changesets after 1.0 if that hurts.

## D-003. Client and server minors match until 1.0
Status: decided, 2026-10-27.
npm `0.N.x` pairs with Composer `0.N.x`. A breaking protocol change bumps the minor on both sides in the same release window, and each changelog links to the other. After 1.0 a protocol version replaces minor matching (D-012).

## D-004. Version branches, cut lazily, main-first backports
Status: decided, 2026-10-27.
`release/0.N` (`release/1.x` after 1.0), cut from the last tag when the next minor ships. Fixes land on `main` first and are cherry-picked with `-x` via a `backport/<line>` label. Only patch releases on a release branch. No upfront branch overhead, and a fix can never exist only on an old branch.

## D-005. Support window: current plus one previous minor
Status: decided, 2026-10-27.
Critical fixes only. Older lines are archived. After 1.0: the current major, plus the previous major for critical fixes for six months.

## D-006. npm dist-tag rules
Status: decided, 2026-10-27.
`latest` only for the highest stable version; a patch on an older line publishes as `release-<line>`; prereleases publish as `next`. The publish workflow compares the tag with the highest published version.

## D-007. Human gate on every release
Status: decided, 2026-10-27.
Merging the release PR plus one approval on a GitHub Environment ships it. Manual tags and publishes are blocked by a hook. CI creates the tag and publishes in one job, because tags pushed with `GITHUB_TOKEN` do not trigger other workflows.

## D-008. Shared tooling in its own repo
Status: decided, 2026-10-27.
`inertia-live-tooling` holds the release script, reusable workflows and a Claude Code plugin. Each repo carries `.release.json`, a thin caller workflow pinned to a tag and a `.claude/settings.json` enabling the plugin. Copying skills and workflows into every repo drifts.

## D-009. The docs repo is the source of truth
Status: decided, 2026-10-27.
`inertia-live-docs` holds `release-policy.md`, `roadmap.md` and this log. Every repo has a `CLAUDE.md` pointing here. The tracker mirrors progress and decisions. Agents update the docs first, then the tracker, in the same change.

## D-010. No AI attribution
Status: decided, 2026-10-27.
No AI or Claude attribution in commits, pull requests, issues, changelogs, docs or code comments.

## D-011. What goes into 1.0
Status: open, 2026-10-27.
The owner wants API enhancements and refactoring so the packages are easy to set up, consume and test, but cannot name the scope yet. Phase 1 audits produce the list; the scope decision follows. The contract (phase 2) and quality gates (phase 3) wait on it; tooling and hygiene work can proceed in parallel.

## D-012. Explicit protocol version in signals
Status: open, 2026-10-27.
Minor matching stops being enough after 1.0, and adding a version field later is awkward. Recommended: add `protocol: 1` to signals before 1.0. It touches both repos and the spec. Decide during phase 1.

## D-013. Docs are versioned by major in folders
Status: decided, 2026-10-27.
Guides and reference live in `v1/`, `v2/` and so on; the root keeps only project docs and the index. `v2/` is created as a copy of `v1/` when the next major starts. Moving the existing pages into `v1/` would break links that are already published in npm READMEs, so one-line stubs remain at the old root paths until after the next release. Open consequence: pre-1.0 users read `v1/`, which will describe 1.0 once it ships; decide at 1.0 whether to keep a frozen copy of the 0.x docs.

## D-014. Generated specs and plans are not committed
Status: decided, 2026-10-27.
`/docs/` is gitignored in all four repos. Design documents produced during agent-assisted work stay local. Anything that must survive goes in this decision log, the roadmap, a repo README or `CLAUDE.md`. Consequence: the tooling design doc is local only; its rules are recorded in the release policy and the tooling README.

## D-015. A response factory to replace `Inertia::render(...)->live(...)`
Status: open, 2026-10-27.
Today a live page needs `Inertia::render($component, $props)->live($topic, only: [...])`, and `only` must be kept in sync with the props by hand. The owner wants a Laravel-side response factory as part of v1, such as `InertiaLive::render($component, $props, $id)` or a better API. Candidates are recorded in the roadmap (phase 1). Constraints: it must return an `Inertia\Response` so existing code keeps working, `->live()` stays as the low-level escape hatch, and it must work on Inertia 2 and 3. Not decided.

## D-016. The demo app is deferred until the packages are publishable
Status: decided, 2026-10-27.
`inertia-live-demo` should showcase Inertia Live with both the Vue and React frontends, built around a real-world scenario where live updates are clearly useful (the scenario is not chosen yet). It waits until the 1.0 contract and API are settled, so it is built once against the final API. Consequence: the work that assumes a demo (the Playwright end-to-end run, the five-minute quickstart, checking the response factory in a real app, the release-candidate dogfood) depends on it, so the demo must exist before phase 3's end-to-end task and phase 6 can finish. The repository is not checked out locally; `../inertia-live-demo-local-old` is an earlier local copy, not the current repo.

## D-017. The tracker is updated with every change, at one URL
Status: decided, 2026-10-27.
Every change to progress, tasks, decisions or notes is mirrored in the roadmap tracker artifact in the same piece of work. There is one tracker; updates go to the same artifact (database writes for state, a republish with `url` for page changes), never a new one. Each repo's `CLAUDE.md` carries the procedure.

## D-018. The demo repo exists; it is a starting point, the scenario is still open
Status: decided, 2026-10-27. Supersedes only the "not checked out locally" part of D-016.
`freepeace13/inertia-live-demo` is cloned locally at `inertia-live/inertia-live-demo`. It is a Laravel 13 app (Inertia 3, Spatie Event Sourcing, Reverb) with a documents-and-comments example in Vue (`/documents`) and React (`/react/documents`). That example is not the chosen real-world scenario; the owner wants the demo to show real-world use cases and will rework it later, so the scenario stays open (D-016). It also depends on `inertia-live-laravel` at `dev-main` and on `file:../inertia-live/...` client links, which no longer resolve after the folders were regrouped. The old local copy `inertia-live-demo-local-old` was deleted by the owner.
