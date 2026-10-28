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
Status: decided, 2026-10-28 (was open, 2026-10-27).
The owner wants API enhancements and refactoring so the packages are easy to set up, consume and test. Phase 1 audits (phase-1-audit.md) produced the list. **In scope for 1.0:**
1. The response factory with live props (D-019).
2. An explicit `protocol` field in signals (D-012). The shape and rollout are still decided under D-012; the recommendation is `protocol: 1`, optional on the client first.
3. A checked install path: the install docs fixes, a five-minute quickstart checked against the demo, and an `inertia-live:install` Artisan command that publishes and migrates Spatie's migrations and prints the client snippet (audit items 3, 22, 23).
4. A binding assertion for controller tests that works with `assertInertia` (audit item 12).
5. The stable-API list and `@internal` marking (audit items 13, 14), including removing `ConnectionTracker` and `CursorStore` from the `core` public entry.
6. Skipping the redundant `_live` catch-up reload on subscribe when the binding's cursor equals the client's (audit item 28). Touches client-core behavior and the spec. **Amended by D-020:** replaced by a lighter cursors-only read, because skipping the read removes the missed-signal check.
**Optional, only if time allows:** a shared options type (15), an IDE stub for `->live()` (8), a cursor-store boot warning (5), `LiveFake` delegation (11).
**Out of scope:** an `EmitsLiveChanges` hook redesign (10), beyond possibly adding an alternative later, and a shared topic helper to remove the repeated topic string (7). Both can follow 1.0.
Consequences: the contract (phase 2) and quality gates (phase 3) can start. Items 2, 3 and 6 change the protocol or the install path, so each goes through the `protocol-change` skill and moves client, server and spec together. Items 1 to 6 need implementation tasks; D-012 is still open on the field's shape.

## D-012. Explicit protocol version in signals
Status: decided, 2026-10-28 (was open, 2026-10-27).
Minor matching stops being enough after 1.0, and adding a version field later is awkward. Decision: signals carry `protocol: 1`, introduced as optional. The server emits it; a client that receives a signal without it treats it as protocol 1, so a new client works with an older server and the reverse. Making the field mandatory is a later step, decided with the versioned spec in phase 2. Where else the field appears (for example in the `_live` bindings) and what a client does with a higher value than it supports are settled in the protocol spec task, not here.
Consequences: a protocol change, so it goes through the `protocol-change` skill and moves client, server and spec together (D-011 item 2). Because the field is optional, it does not by itself force a minor bump on both sides.

## D-013. Docs are versioned by major in folders
Status: decided, 2026-10-27.
Guides and reference live in `v1/`, `v2/` and so on; the root keeps only project docs and the index. `v2/` is created as a copy of `v1/` when the next major starts. Moving the existing pages into `v1/` would break links that are already published in npm READMEs, so one-line stubs remain at the old root paths until after the next release. Open consequence: pre-1.0 users read `v1/`, which will describe 1.0 once it ships; decide at 1.0 whether to keep a frozen copy of the 0.x docs.

## D-014. Generated specs and plans are not committed
Status: decided, 2026-10-27.
`/docs/` is gitignored in all four repos. Design documents produced during agent-assisted work stay local. Anything that must survive goes in this decision log, the roadmap, a repo README or `CLAUDE.md`. Consequence: the tooling design doc is local only; its rules are recorded in the release policy and the tooling README.

## D-015. A response factory to replace `Inertia::render(...)->live(...)`
Status: decided in D-019, 2026-10-28 (was open, 2026-10-27).
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

## D-019. The response factory uses live props (candidate C)
Status: decided, 2026-10-28. Resolves D-015.
The Laravel adapter gets `InertiaLive::render($component, $props)` that returns an `Inertia\Response`. A prop becomes live by wrapping it: `'document' => live("documents.{$doc->uuid}", fn () => ...)`. The factory derives `only` and the `_live` bindings from the wrapped props, so no `only` list is written and the empty-`only` error cannot happen. Props that are not wrapped are not live. There is no `id` argument and no "default `only` to every prop" rule, which closes those open questions. `->live()` stays as the low-level escape hatch. A global `inertia_live()` helper (candidate D) may follow as sugar and is not part of this decision. Candidates A and B were not chosen: A keeps the hand-written `only`, B needs a topic-template lookup that hides the topic.
Mechanism: the factory unwraps live props itself before calling `Inertia::render`, passing the inner closure or value and recording topic to props, so it does not depend on how Inertia 2 or 3 resolves custom prop types. This is the intended design and must be proved by a spike on Inertia 2 and 3, including partial reloads, before the implementation task starts.
Consequences: additive, no wire-protocol change, no breaking change for 0.x users. The topic string is still written both on the event's `#[LiveTopic]` and in the controller (phase-1-audit.md item 7); this decision does not remove that, and a shared topic helper can be considered later. Touches the PHP API, `Live::fake()` tests (add a binding assertion, audit item 12), `v1/bindings.md`, the quickstart and the demo. Implementation waits for the spike and for D-011 to put it in the 1.0 scope.

## D-020. Protocol details: higher versions, placement and the subscribe-time read
Status: decided, 2026-10-28. Fills the details D-012 left to the spec, and amends item 6 of D-011.
1. **A signal with a `protocol` higher than the client supports:** the client reloads every prop bound to that topic and calls `onError` once (per topic, not per signal). It does not ignore the signal and does not stop listening.
2. **Where `protocol` appears:** on the signal only. `_live` does not carry it.
3. **The subscribe-time `_live` read (D-011 item 6):** the plan to skip the read when the cursor matches does not work, because the read exists to compare the server's current cursor with the render-time cursor the client holds. It is replaced by a lighter cursors-only read: a small package endpoint that returns the current cursors for the given topics, without running the page's controller. The client calls it when a channel's subscription is confirmed, and reloads the bound props of any binding whose cursor is ahead. The endpoint must apply the same authorization as the channel, because a cursor reveals that a topic changed. Until it ships, the existing `_live` re-read is the behavior.
Still open: making `protocol` mandatory (decided with the contract tests), and the endpoint's route, how the client learns its URL and its response shape (protocol spec task).
Consequences: item 3 adds a small HTTP surface to the protocol, so it goes through the `protocol-change` skill and moves client, server and spec together. Items 1 and 2 add no wire surface beyond D-012's field.

## D-021. The PHP namespace is `InertiaLive`
Status: decided, 2026-10-28.
The root namespace of the Laravel package changes from `Freepeace13\InertiaLive` to `InertiaLive` (`InertiaLive\Attributes\LiveTopic`, `InertiaLive\Facades\Live`, `InertiaLive\Concerns\EmitsLiveChanges` and so on). The owner's reason: the package has no real users yet, so the rename costs nothing now and a vendor-free namespace reads better in application code. Unchanged: the Composer package `freepeace13/inertia-live-laravel`, the npm scope `@freepeace13/inertia-live-*`, and the GitHub repository names.
Consequences: breaking for anyone who installed 0.1.0 and imports package classes, accepted because there are no known users (0.1.0 is on Packagist, so it is installable). No wire-protocol impact. Every `use` line in docs, the demo and tests changes with it. The Laravel changelog records it as breaking under `## Unreleased`; no version is bumped or tagged here. The demo's lock file pins the old package, so the demo needs a `composer update` once the change is on `dev-main`. Namespace and class names listed in the stable-API work (`t-freeze`) use the new namespace.

## D-022. The cursors endpoint
Status: decided, 2026-10-29. Specifies item 3 of D-020.
The package adds a small endpoint that returns current cursors, replacing the subscribe-time `_live` re-read. As specified in `v1/protocol.md` ("The cursors endpoint"):
- **Discovery:** `_live.cursors` is a same-origin path; if absent, the client keeps using the `_live` re-read.
- **Request:** `GET {_live.cursors}?topics[]=...`, sent with `fetch`, batched per burst, at most 50 topics (more is a `422`).
- **Response:** `{ "cursors": { "<topic>": <int> } }` with `Cache-Control: no-store`. Topics the caller may not subscribe to are omitted, and the client then reloads that binding's props. A non-200 response or network error falls back to the `_live` re-read.
- **Authorization:** the same callbacks as the private channel, run for the signed-in user; public topics need no check; a request without a user reads public topics only. `LiveManager` keeps the registered callbacks and exposes a public "may this user subscribe" method.
- **Route and config:** route `inertia-live.cursors` at `inertia-live/cursors`; `inertia-live.cursors.path` and `inertia-live.cursors.middleware`, default middleware `['web', 'throttle:60,1']`.
- **Client:** `LiveClient` takes an optional `fetchCursors` (default: global `fetch`).
Consequences: adds public surface (route name, `_live.cursors`, request and response shapes, two config keys, the `LiveManager` method) to the stable-API list (`t-freeze`). A signed-token alternative was rejected: it ties the check to the page render rather than channel access and adds key handling and expiry. Implementation is `t-impl-catchup-skip` and goes through the `protocol-change` skill. The throttle default is the owner's accepted starting value and may change in implementation.

## D-023. The stable API list
Status: decided, 2026-10-29.
`v1/stable-api.md` lists what 1.0 promises, in three tiers: stable (no breaking change in 1.x), experimental (may change in a minor, noted in the changelog) and internal (no promise, `@internal` or not exported). The list was built from an inventory of the code and docs and covers PHP, JavaScript and the wire. Calls made: `CursorRepository` and the protected `liveChanged()` hook are experimental; `SendTrailingSignal` is internal with a rule not to rename it or change its constructor while queued jobs may be in flight; `FlushesChanges` is internal; `withSocketId` and `SOCKET_ID_HEADER` are experimental (not internal) because custom adapters need them.
Consequences: before 1.0 the code must catch up with the list: `@internal` and `@experimental` markers, tests for the stable items that have none (config `cursor_store`, `createInertiaReloader`, `withSocketId`), re-exporting `LiveStatus`, `EchoLike`, `Reloader` and `ReloadCancelled` from `vue` and `react`, docs for `withSocketId`, consistent wording for template, pattern and topic, and removing `ConnectionTracker` and `CursorStore` from the `core` entry (D-011 item 5). The planned items from D-011 to D-022 join the stable tier when they ship. The deprecation policy (`t-deprecation`) defines how a stable item is removed.

## D-024. The support matrix
Status: decided, 2026-10-29.
`v1/compatibility.md` is the support matrix: PHP 8.3 to 8.5, Laravel 12 and 13, Inertia 2 and 3, Spatie event sourcing ^7.14, React 18 and 19 (18 only with Inertia 2, because Inertia 3 requires React 19), Vue 3.4 or later, `laravel-echo` ^2.0, and Node 22 or later at 1.0. Node 20 reached end of life in April 2026 and the React tests already need Node 22. A dependency version is dropped when its upstream security support ends, in a minor, announced a minor earlier where possible; adding one needs a green CI cell first.
Consequences: before 1.0 `engines.node` becomes `>=22` and the Node 20 CI cell goes; the docs state the Inertia and React pairing; phase 3 adds a lowest-versions client cell, since Vue 3.4 and the lowest Echo 2 are claimed but not tested.

## D-025. The deprecation policy
Status: decided, 2026-10-29. The owner accepted the draft with "carry on", including the compatibility promise.
Stated in `release-policy.md`, from 1.0: stable items are removed only in a major, after being deprecated earlier in that major, and keep working until then; a deprecation has a changelog entry, a docs mark and a one-time runtime notice; experimental items may change in a minor with a changelog entry and no deprecation period; internal items carry no notice; the wire protocol keeps one version per major, with optional fields allowed in a minor; a 1.x client works with any 1.y server and the reverse, so every new feature must degrade to the older behavior or wait for a major; dropping a dependency version is not a breaking change.
Consequences: the compatibility promise constrains design, for example the cursors endpoint falls back to the `_live` re-read when a server sends no `_live.cursors` (D-022). Whether a 2.x client also speaks protocol 1 is decided when 2.0 is planned.

