# Inertia Live: Documentation

Documentation for **Inertia Live**: live [Inertia](https://inertiajs.com) pages driven by [Spatie Event Sourcing](https://github.com/spatie/laravel-event-sourcing) projections. You declare on the server which projection changes affect which page props. The package broadcasts a small "topic changed" signal, and the client reloads only the affected props through the page's own controller.

## Start here

- New to Inertia Live? Read [Installation](installation.md), then follow the guides in order below.
- Want to see it running? Clone the [demo app](https://github.com/freepeace13/inertia-live-demo).
- Looking for something specific? Jump to [Troubleshooting](troubleshooting.md) or [Configuration](configuration.md).

## Guides

Read these in order for a complete setup.

| # | Guide | What it covers |
| --- | --- | --- |
| 1 | [Installation](installation.md) | Requirements, packages, broadcasting and Echo setup |
| 2 | [Topics](topics.md) | `#[LiveTopic]`, placeholders, explicit `liveChanged()` |
| 3 | [Projectors](projectors.md) | `EmitsLiveChanges`, the change buffer, flushing, queues and replays |
| 4 | [Page bindings](bindings.md) | `->live()` on Inertia responses and the `_live` prop |
| 5 | [Authorization](authorization.md) | `Live::authorize()`, private and public topics, fail-closed rules |
| 6 | Client: [Vue 3](vue.md) or [React](react.md) | Adapter setup and `useLive()` |
| 7 | [Testing](testing.md) | `Live::fake()` for PHP, `createFakeLive()` for Vitest |

## Reference

| Doc | What it covers |
| --- | --- |
| [Configuration](configuration.md) | Every `config/inertia-live.php` key |
| [Client core](client-core.md) | `LiveClient`, Echo and connection types, writing a custom adapter |
| [Consistency](consistency.md) | Versions, cursors, ordering, debouncing and reconnects |
| [Design decisions](design-decisions.md) | Why versions are per-topic sequences, and what that costs |
| [Troubleshooting](troubleshooting.md) | Symptoms, causes and fixes |
| [Spec](SPEC.md) | Original design, goals and milestones |

## How the pieces fit

```
stored event ──► projector ──► ChangeBuffer ──► (after commit) ChangeFlusher ──► Echo / Reverb
  #[LiveTopic]   EmitsLiveChanges   coalesce          sequence + rate limit         private-live.{topic}
                                                                                         │
page props ◄── router.reload({ only }) ◄── debounce ◄── LiveClient (drops stale versions)
```

The socket never carries model data. Policies, hidden attributes and per-user fields keep working because the data still comes from your controller.

## Repositories

| Repository | Role |
| --- | --- |
| [`inertia-live`](https://github.com/freepeace13/inertia-live) | Client packages: `core`, `vue`, `react` |
| [`inertia-live-laravel`](https://github.com/freepeace13/inertia-live-laravel) | Server package (Composer) |
| [`inertia-live-demo`](https://github.com/freepeace13/inertia-live-demo) | Laravel 13 demo with Vue and React frontends |
| **`inertia-live-docs`** | This documentation |
