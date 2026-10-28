# Stable API

What 1.0 promises, and what it does not. Everything not listed as stable is either experimental or internal.

> Accepted as [D-023](../decision-log.md) on 2026-10-29, from an inventory of the code and docs. The items decided in D-011 to D-022 are built on `main` and listed below with their tier; they are stable from the 1.0 release. Nothing is frozen until 1.0 ships.

## Tiers

| Tier | Promise |
| --- | --- |
| **Stable** | No breaking change in 1.x. A removal or incompatible change waits for 2.0 and is deprecated first (the deprecation policy is a separate phase 2 task). |
| **Experimental** | May change or go away in a 1.x minor. Documented as experimental, and listed in the changelog when it changes. |
| **Internal** | No promise. Not documented as API. In PHP it carries `@internal`; in JavaScript it is not exported, or is tagged `@internal`. |

A bug fix is not a breaking change. Wire behavior is covered by [Wire protocol](protocol.md), which has its own version.

## PHP (namespace `InertiaLive`)

### Stable

| Item | Notes |
| --- | --- |
| `InertiaLive\InertiaLiveServiceProvider` | As the auto-discovery entry only. |
| `#[LiveTopic(template, props, ...)]` | The attribute, its two public properties, and repeatability. |
| `EmitsLiveChanges` trait | The trait name and its documented behavior: signals after the handler returns. |
| `Live` facade: `authorize()`, `publicTopic()`, `canSubscribe()`, `fake()` | `canSubscribe($user, $topic)` answers whether a user may subscribe to a topic. |
| `Live::fake()` assertions: `assertChanged()`, `assertNothingChangedFor()`, `assertChangedTimes()` | They are methods of `LiveFake`, which is `@internal` as a class; the methods are the promise. |
| `Inertia\Response::live($topic, $only)` macro | Stays as the low-level path next to the response factory. |
| Config file `config/inertia-live.php`, publish tag `inertia-live-config` | |
| Config keys `enabled`, `channel_prefix`, `cursor_store`, `max_signals_per_second`, `replay.suppress`, `replay.final_signal`, `cursors.path`, `cursors.middleware` | `channel_prefix` is part of the wire (the channel name). |
| Broadcast event name `live.changed` and the payload | Wire; see [Wire protocol](protocol.md). |
| `InertiaLive::render($component, $props)` and the function `InertiaLive\live($topic, $value)` (D-019) | The factory and the prop wrapper. |
| Test assertions `assertLive($topic, ?$props)` and `assertNotLive(?$topic)` on `AssertableInertia` and the test response (D-011 item 4) | |
| The artisan command `inertia-live:install` and its options `--client`, `--skip-migrations`, `--force` (D-011 item 3) | The command class is internal; the signature is the surface. |
| The cursors endpoint (D-022): route name `inertia-live.cursors`, `_live.cursors`, config keys `cursors.path` and `cursors.middleware` | The request and response shapes are in [Wire protocol](protocol.md). |

### Experimental

| Item | Why |
| --- | --- |
| `Live::isPublic()`, `Live::hasAuthorizerFor()` | Query helpers the docs barely use. |
| `liveChanged($topic, $props)` protected hook on `EmitsLiveChanges` | Manual use for events without the attribute. Subclass-facing. |
| `CursorRepository` interface (binding your own cursor store) | An extension point, but its contract (atomic increment, never reuse a number) is easy to get wrong. See [Decided calls](#decided-calls). |
| `LiveChangeBroadcast` class (listening to the broadcast event) | The event name and payload are stable; the class and its constructor are not. |
| Config keys `cursor_ttl`, `debug` | `cursor_ttl` has consistency effects and no test; `debug` defaults to `APP_DEBUG`. |
| Log messages ("topic has no authorizer; signal not sent", "failed to flush a change") | Operators may alert on them. Wording can change. |

### Internal (mark `@internal`)

`LiveManager` (use the facade), `LiveBindings`, `LiveProp`, `Http\CursorsController`, `Console\InstallCommand`, `Testing\LiveAssertions`, `Change`, `ResolvedTopic`, `TopicResolver`, `ChangeBuffer`, `ReplayBuffer`, `ChangeFlusher`, `Contracts\FlushesChanges`, `Cursor\CacheCursorRepository`, `Testing\LiveFake` (reached through `Live::fake()`), `LiveResponseMacro::register()` and `bind()`, the cache keys (`inertia-live:cursor:*`, `inertia-live:signals:*`, `inertia-live:trailing:*`).

`Jobs\SendTrailingSignal` is internal, but it is a queued job: its class name and constructor are serialized into queues. Do not rename it or change its constructor within a release line that may have jobs in flight (see [Decided calls](#decided-calls)).

## JavaScript

All three packages are ESM only.

### Stable

| Package | Items |
| --- | --- |
| `core` | `LiveClient` and `LiveClientOptions` (with `fetchCursors` and `onError`); `ReloadCancelled`; types `Binding`, `LiveProp` (with `cursors`), `ChangeSignal` (with `protocol`), `LiveStatus`, `Reloader`, `EchoLike`, `FetchCursors`. The members of `LiveClient` documented in [Client core](client-core.md) are stable except `resetPause()`. |
| `core/testing` | `createFakeLive()` with `echo`, `reloads`, `reload`, `emit` (including its `protocol` argument), `fetchCursors`, `cursorRequests`, `setServerCursor`, `failCursors`. |
| `vue` | `InertiaLive` plugin and `InertiaLiveOptions`; `useLive()` and `UseLive`. |
| `vue/testing` | `createFakeLive()` with `options` (and the `debounceMs` and `channelPrefix` arguments). |
| `react` | `InertiaLiveProvider` and `InertiaLiveProviderProps`; `useLive()` and `UseLive`. |
| `react/testing` | `createFakeLive()` with `providerProps` (and the `debounceMs` and `channelPrefix` arguments). |
| `vue`, `react` | The `fetchCursors` option on the plugin and the provider (D-022). |

### Experimental

| Item | Why |
| --- | --- |
| `createInertiaReloader()` and `ReloadRouter` (core); `inertiaReloader` (vue, react) | A seam used by the two adapters; the reload semantics may be tuned. |
| `SOCKET_ID_HEADER` and `withSocketId()` | Needed by anyone writing a custom adapter, so they must be exported, but they are untested and undocumented today. See pre-freeze work. |
| `EchoChannelLike` and `ConnectionLike` types | Minimal structural shapes that may grow. |
| `LiveClient.resetPause()` | Plumbing for adapters, documented as such. |
| `FakeLive` members `joined`, `left`, `setSocketId`, `confirmSubscription`, `setConnectionState` | They follow the client's internals and will change with `fetchCursors`. |

### Removed from the public entry (D-011 item 5)

`ConnectionTracker` and `CursorStore` have left the `core` entry (done on `main`, unreleased). They are internal and tagged `@internal`.

## Wire

Channel naming, the signal, the `_live` prop and `X-Socket-ID` are stable as described in [Wire protocol](protocol.md), protocol version 1.

## Before freezing

These are gaps the inventory found that should be closed or accepted before 1.0 labels something stable:

- **Tests for stable items that have none:** done. `cursor_store` was already tested; `createInertiaReloader` and `withSocketId` have tests on `main`.
- **`@internal` and `@experimental` markers** in PHP docblocks and TSDoc, so the tiers are visible in editors. Done on `main` for the client and PHP.
- **Docs:** `withSocketId` and `SOCKET_ID_HEADER` for custom adapter authors (done, in [Client core](client-core.md)); the rest: `Live::isPublic` or remove it from the facade docblock; the `LiveManager` and `Live` facade docblock lists the manager and fake methods by hand and should be generated or trimmed.
- **Re-exports from `vue` and `react`:** done on `main`: `LiveStatus`, `EchoLike`, `Reloader` and `ReloadCancelled` are importable from both adapters.
- **One options type:** `LiveClientOptions`, `InertiaLiveOptions` and `InertiaLiveProviderProps` repeat the same fields; a new option must be added in three places. Optional in D-011.
- **One word for one idea:** a **topic** is a concrete name such as `documents.9f1c2e`. A **pattern** is a topic with `{placeholders}`, used by `Live::authorize()`, `Live::publicTopic()` and the `#[LiveTopic]` attribute (whose parameter is named `template`; the name is kept because renaming it breaks named arguments). The docs use these two words only.

## Decided calls

- **`CursorRepository`** is experimental: a wrong custom cursor store loses signals.
- **`liveChanged()`** is experimental.
- **`SendTrailingSignal`** is internal, with the rule not to rename it or change its constructor while jobs may be in flight.
- **`FlushesChanges`** is internal: it is the seam `Live::fake()` replaces and is not meant to be implemented by users.
