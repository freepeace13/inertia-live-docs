# Compatibility

The versions Inertia Live supports, what CI actually checks, and how a version leaves the list.

> Accepted as [D-024](../decision-log.md) on 2026-10-29, from the CI workflows and dependency ranges of that date.

## Supported versions

| Dependency | Supported | Notes |
| --- | --- | --- |
| PHP | 8.3, 8.4, 8.5 | |
| Laravel | 12, 13 | |
| `inertiajs/inertia-laravel` | ^2.0 or ^3.0 | |
| `spatie/laravel-event-sourcing` | ^7.14 | The first line that supports Laravel 13. |
| Broadcaster | Reverb, Pusher, Ably (any Echo-compatible driver) | Connection tracking and the reconnect reload need a Pusher-protocol connection (Reverb, Pusher); other drivers pass their own `connection`. |
| `@inertiajs/vue3`, `@inertiajs/react` | ^2.0 or ^3.0 | Inertia 3 requires React 19. |
| React | 18 or 19 | React 18 only with Inertia 2. |
| Vue | 3.4 or later | |
| `laravel-echo` | ^2.0 | |
| Node (building the client) | 22 or later | `engines.node` is `>=22` (on `main`, unreleased). |

Server and client are independent: any supported client works with any supported server, as long as the [wire protocol](protocol.md) version matches (before 1.0, as long as the minors match).

## What CI checks

| Area | Covered | Not covered |
| --- | --- | --- |
| Server | PHP 8.3, 8.4, 8.5 x Laravel 12, 13 x Inertia 2, 3; a `prefer-lowest` run on PHP 8.3 | |
| Client, Inertia and React | Inertia ^2 with React 19 and 18; Inertia ^3 with React 19; the lowest Inertia 2 (`~2.0.0`) with React 18 | |
| Client, Node | Node 22 for core, Vue and React | Node 20 (end of life; dropped) |
| Vue | The newest 3.x, and Vue 3.4.0 in the `lowest` cell | |
| `laravel-echo` | Faked in tests (`createFakeLive()`), at the newest 2.x and at 2.0.0 in the `lowest` cell | A real Echo against Reverb |
| Package shape | `publint`, `@arethetypeswrong/cli` (ESM-only profile) and a tarball contents check on all three packages | |
| Reverb round trip | The demo app's CI runs a real Reverb round trip (two Echo clients, the cursors endpoint, sender exclusion, authorization refusal) and a two-tab Playwright run for Vue and React | The run uses the newest dependencies, not the lowest |

The supported list above is wider than what CI proves in one place: the end-to-end run uses the newest `laravel-echo` and Inertia, not the lowest. The `lowest` client cell (Inertia 2.0, React 18, Vue 3.4.0, `laravel-echo` 2.0.0) is on `main`.

## Changes this required

1. **Node 22 or later.** Node 20 reached end of life in April 2026. `engines.node` is `>=22` and the Node 20 CI cell is gone.
2. **State the Inertia and React pairing in the docs.** Inertia 3 with React 18 is unsupported because Inertia 3 requires React 19. `installation.md` says so.
3. **A lowest-versions client cell** in CI, so the supported range is tested at its edge.

## How a version leaves the list

- A dependency version is dropped when its upstream security support ends, not before. Laravel 12 and Inertia 2 are the next candidates (the original plan was to drop Inertia 2 once its security support ended in March 2027, and Laravel 12 follows its own security window).
- Dropping a version is not a breaking change under [semver](../release-policy.md#versioning). It happens in a minor release, announced in the changelog one minor earlier when possible, and the CI matrix and this page change in the same release.
- Adding a supported version (for example Laravel 14) is a minor release and needs a green CI cell first.
- Before 1.0, a version can be dropped in any minor.
