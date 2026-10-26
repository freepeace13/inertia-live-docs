# Roadmap to 1.0

The packages are published at 0.1.x (client `0.1.1`, Laravel `0.1.0`). 1.0 is a promise about what will not break, so it is gated on a defined contract and on the developer experience, not on a date. Live progress is tracked in the [roadmap tracker](https://claude.ai/artifact/PjKwna6Tty3ZnZFKneBjhT) (private; owner access). This file is the source of truth: if the tracker disagrees, this file wins. Decisions are in the [decision log](decision-log.md), rules in the [release policy](release-policy.md).

## Goals

The packages should be **easy to set up, easy to consume and easy to test**. The API and implementation need enhancement and refactoring toward that. The exact scope is not yet defined; phase 1 produces it (D-011).

## Phases

### 0. Foundations
Tooling repo (release script, tests, design doc), policy docs, `CLAUDE.md` in every repo, the tracker. Generated specs and plans are gitignored in every repo (D-014) and the docs are versioned by major in `v1/` (D-013). Remaining: push the tooling repo, tooling CI, branch protection, confirm Packagist, add `.release.json` and `## Unreleased` to the client and Laravel changelogs.

### 1. DX and API review
Audit before deciding. Walk the install path from a fresh Laravel app and record every manual step. Review the PHP API (`Live::`, `#[LiveTopic]`, `EmitsLiveChanges`, `->live()`, config keys, `CursorRepository`), the JS exports of `core`/`vue`/`react` and their `/testing` entries, and the testing story (`Live::fake()`, `createFakeLive()`). Investigate, without committing to them: an install command or auto-discovery, one-line Vue/React setup, zero-config defaults, and a Laravel response factory that replaces `Inertia::render(...)->live(...)` (D-015, candidates below). Compile the refactoring candidates with the breaking-change impact of each. Decide the 1.0 scope (D-011), whether signals carry an explicit `protocol` version (D-012), and the response factory API (D-015).

#### Response factory candidates (D-015)

All return an `Inertia\Response`; `->live()` stays as the escape hatch.

- **A. Factory with a topic argument:** `InertiaLive::render('Documents/Show', $props, topic: "documents.{$id}", only: ['document'])`. Smallest change; `only` still has to be listed by hand.
- **B. Topic template plus id:** `InertiaLive::render('Documents/Show', $props, id: $doc->uuid)`, where the topic comes from a `#[LiveTopic]` template or a model hook. Shortest call site; needs a way to say which template the page uses.
- **C. Live props:** props declare their own topic, and the factory derives `only` and `_live` from them, e.g. `'document' => live("documents.{$id}", fn () => ...)`. No `only` list to keep in sync, and the empty-`only` error cannot happen. Most ergonomic; needs care to work across Inertia 2 and 3 prop resolution.
- **D. Helper only:** a global `inertia_live()` mirroring `inertia()`. Sugar over any of the above.

Open questions: what `id` means in `render(component, props, id)`; whether to default `only` to every prop when none is given; how this appears in `Live::fake()` tests.

### 2. Contract
Turn `SPEC.md` into a versioned protocol spec (signal payload, the `_live` read, cursor and version semantics, `X-Socket-ID`). List the public API promised stable for JS and PHP. Fix the support matrix: Inertia 2/3, React 18/19, Vue 3.4+, Node 20+, PHP 8.3+, Laravel 12/13, Echo 2. Write the deprecation policy. Mark anything not frozen as experimental.

### 3. Quality gates
Shared contract-test fixtures consumed by both repos; Playwright end-to-end against the demo; `publint`, `@arethetypeswrong/cli` and an `npm pack` contents check in CI; a Laravel CI matrix (PHP 8.3/8.4, Laravel 12/13, Inertia 2/3, `prefer-lowest`); SSR tests for the adapters.

### 4. Hygiene and docs
`keywords` on the three packages; `SECURITY.md`, `CONTRIBUTING.md`, issue and PR templates, `CODEOWNERS`; Dependabot or Renovate aware of the pnpm catalog; `.gitattributes` `export-ignore` in the Laravel repo; remove the root stubs that point into `v1/` after the next release; 2FA and trusted publishing on npm; an upgrade guide from 0.x; a five-minute quickstart.

### 5. Release automation and agents
Reusable workflows in the tooling repo: prepare-release, publish-on-merge, backport, release-window pruning. Thin caller workflows in each repo. The Claude plugin: `/release`, `/backport` and `/protocol-change` skills, `release-reviewer` and `protocol-reviewer` agents, and the hook that blocks manual tag and publish. Each repo's `.claude/settings.json` enables the plugin.

### 6. Rehearsal and 1.0
`1.0.0-rc.1` on `next`; run it in a real app for about two weeks; packaging dry run (`npm pack`, install the tarballs in a fresh app, `npm publish --dry-run`); `rc.2` if needed; `1.0.0`; cut `release/0.N` for the current and previous minor.

### Demo app (repo exists; scenario open, rework later, D-016, D-018)
The `inertia-live-demo` app should showcase both the Vue and React frontends with a real-world scenario where live updates are clearly useful. The repo exists with a documents-and-comments example in Vue and React, but the real-world scenario is not chosen (D-018). It is reworked after the API is settled (end of phase 2) so it uses the final API, and it must exist before the end-to-end task in phase 3 and the release-candidate dogfood in phase 6 can finish. The quickstart and the response factory docs are checked against it.

## Order

Phase 0, then phase 1 (it decides what everything else checks), then phases 2, 3 and 5 in parallel where they do not depend on the scope decision (tooling and hygiene can start early), then phase 4, then phase 6.

## 1.0 definition of done

- The protocol spec is versioned and contract tests pass in both repos.
- The API review is done, the 1.0 scope is decided and implemented, and the support matrix is green in CI.
- Package-shape checks pass; community files and registry settings are in place.
- The upgrade guide and quickstart exist and the quickstart works against the demo.
- At least one release candidate has run in a real app without issues.

## Open questions
- Which real-world scenario or use cases the demo app should show (D-016, D-018).

- 1.0 scope (D-011).
- Explicit protocol version field in signals (D-012).
- Response factory API (D-015).
- Whether to keep a frozen copy of the 0.x docs at 1.0 (D-013).
- Is `freepeace13/inertia-live-laravel` registered on Packagist with the auto-update webhook?
