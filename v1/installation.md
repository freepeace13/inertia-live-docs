# Installation

## Requirements

| Dependency | Supported |
| --- | --- |
| PHP | 8.3, 8.4, 8.5 |
| Laravel | 12, 13 |
| `inertiajs/inertia-laravel` | ^2.0 or ^3.0 |
| `spatie/laravel-event-sourcing` | ^7.14 |
| Broadcaster | Reverb, Pusher or Ably (any Echo-compatible driver) |
| Client | Vue 3.4+ or React 18/19, `laravel-echo` ^2.0, `@inertiajs/vue3` or `@inertiajs/react` ^2 or ^3 (Inertia 3 needs React 19) |

CI runs the server against PHP 8.3 to 8.5, Laravel 12 and 13, and Inertia 2 and 3, and the client against Inertia 2 and 3 and React 18 and 19.

## Server

```bash
composer require freepeace13/inertia-live-laravel
```

The service provider is auto-discovered. It merges the default config, registers the `->live()` macro on `Inertia\Response`, the cursors route, the test assertions and the `inertia-live:install` command, and hooks the flush points (request end, queue jobs, replays).

Then run the install command (since 1.0, [D-011](../decision-log.md)):

```bash
php artisan inertia-live:install
```

It publishes and runs Spatie's event sourcing migrations (skipped when the `stored_events` table exists), publishes the config (skipped when it exists), checks the setup and prints the client snippet for your framework. The checks warn about a broadcaster that delivers nothing (`log` or `null`), a missing channel authorization route and a cursor cache without atomic increments (`file`). They never fail the command; they tell you what to fix.

| Option | Meaning |
| --- | --- |
| `--client=vue` or `--client=react` | Print only that client's setup (default: both) |
| `--skip-migrations` | Do not publish or run the migrations |
| `--force` | Overwrite an already published config file |

The steps below are what the command does, for doing it by hand.

Publish and run Spatie's event sourcing migrations if the app does not have them yet. Without the `stored_events` table the first dispatched event fails:

```bash
php artisan vendor:publish --provider="Spatie\EventSourcing\EventSourcingServiceProvider" --tag=event-sourcing-migrations
php artisan migrate
```

Optionally publish the config:

```bash
php artisan vendor:publish --tag=inertia-live-config
```

See [Configuration](configuration.md) for the keys.

### Broadcasting

Inertia Live uses Laravel's normal broadcasting. If broadcasting is not set up yet:

```bash
php artisan install:broadcasting   # installs Reverb and Echo scaffolding
```

You need:

- A working broadcaster (`BROADCAST_CONNECTION=reverb`, or `pusher`/`ably`) with the server running.
- Channel authorization routes enabled, so `private-live.*` channels can be authorized. `Live::authorize()` registers its callbacks with `Broadcast::channel()`, so they end up in your normal channel authorization flow.
- A cache store that supports atomic `increment` for cursors (Redis, database or Memcached; not `file` with concurrent workers). See [Consistency](consistency.md#cursor-storage).
- A queue is **not** required for the signal: `LiveChangeBroadcast` implements `ShouldBroadcastNow`.

## Client

```bash
npm install @freepeace13/inertia-live-vue laravel-echo pusher-js     # Vue 3
npm install @freepeace13/inertia-live-react laravel-echo pusher-js   # React
npm install @freepeace13/inertia-live-core laravel-echo pusher-js    # another framework
```

`pusher-js` is the client for Reverb and Pusher (both speak the Pusher protocol); other Echo drivers need their own client library instead. Each adapter pulls in `@freepeace13/inertia-live-core`. Install only the adapter you use, with its Inertia peer (`@inertiajs/vue3` + `vue`, or `@inertiajs/react` + `react`).
If you set up broadcasting with `php artisan install:broadcasting`, Laravel's own Echo helper already created the instance, so pass it on and skip the hand-written block below:

```ts
import { echo } from '@laravel/echo-vue' // or '@laravel/echo-react'

app.use(InertiaLive, { echo: echo() }) // Vue; React: <InertiaLiveProvider echo={echo()}>
```

Without that helper, configure Echo once yourself and pass the instance to the adapter:

```ts
// resources/js/echo.ts
import Echo from 'laravel-echo'
import Pusher from 'pusher-js'

window.Pusher = Pusher

export const echo = new Echo({
  broadcaster: 'reverb',
  key: import.meta.env.VITE_REVERB_APP_KEY,
  wsHost: import.meta.env.VITE_REVERB_HOST,
  wsPort: import.meta.env.VITE_REVERB_PORT,
  forceTLS: import.meta.env.VITE_REVERB_SCHEME === 'https',
  enabledTransports: ['ws', 'wss'],
})
```

Then wire up the adapter: [Vue 3](vue.md) or [React](react.md).

### Package entry points

| Import | Contents |
| --- | --- |
| `@freepeace13/inertia-live-core` | `LiveClient`, `ReloadCancelled` and types (`FetchCursors` among them) |
| `@freepeace13/inertia-live-core/testing` | Framework-agnostic `createFakeLive()` |
| `@freepeace13/inertia-live-vue` | `InertiaLive` plugin, `useLive()`, `inertiaReloader`, and the types `LiveStatus`, `EchoLike`, `Reloader` and `ReloadCancelled` |
| `@freepeace13/inertia-live-vue/testing` | Vue `createFakeLive()` |
| `@freepeace13/inertia-live-react` | `InertiaLiveProvider`, `useLive()`, `inertiaReloader`, and the same re-exports |
| `@freepeace13/inertia-live-react/testing` | React `createFakeLive()` |

## Wiring checklist

1. Event carries `#[LiveTopic]` ([Topics](topics.md)).
2. Projector uses `EmitsLiveChanges` ([Projectors](projectors.md)).
3. Topic pattern has a `Live::authorize()` callback ([Authorization](authorization.md)).
4. Controller marks its live props with `live()` and renders through `InertiaLive::render()`, or calls `->live(...)` ([Page bindings](bindings.md)).
5. Client plugin or provider is installed ([Vue](vue.md) / [React](react.md)).
