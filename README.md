# Inertia Live: Documentation

Live [Inertia](https://inertiajs.com) pages driven by [Spatie Event Sourcing](https://github.com/spatie/laravel-event-sourcing) projections. You declare on the server which projection changes affect which page props; the package broadcasts a small "topic changed" signal and the client reloads only the affected props through the page's own controller.

## Versions

Each major version has its own folder. Pick the one that matches the packages you have installed.

| Docs | Packages | State |
| --- | --- | --- |
| [**v1**](v1/README.md) | `@freepeace13/inertia-live-*` 0.1.x today, 1.x after release | Current. Describes 0.1.x until 1.0 ships |

When 2.0 development starts, `v2/` is created as a copy of `v1/`. `v1/` then receives fixes only for supported 1.x lines (see the [release policy](release-policy.md)).

## Project docs

Unversioned. They apply to the whole project and are the source of truth for the release program.

| Doc | What it covers |
| --- | --- |
| [Release policy](release-policy.md) | Versioning, version branches, support window, dist-tags, docs versioning, release flow |
| [Roadmap](roadmap.md) | The road to 1.0: phases, definition of done, open questions |
| [Decision log](decision-log.md) | Numbered release and 1.0 decisions |

## Repositories

| Repository | Role |
| --- | --- |
| [`inertia-live`](https://github.com/freepeace13/inertia-live) | Client packages: `core`, `vue`, `react` |
| [`inertia-live-laravel`](https://github.com/freepeace13/inertia-live-laravel) | Server package (Composer) |
| [`inertia-live-tooling`](https://github.com/freepeace13/inertia-live-tooling) | Release script, reusable workflows, Claude plugin |
| [`inertia-live-demo`](https://github.com/freepeace13/inertia-live-demo) | Laravel 13 demo with Vue and React frontends |
| **`inertia-live-docs`** | This documentation |

Old links such as `installation.md` at the repository root are one-line stubs pointing into `v1/`; they will be removed after the next release.
