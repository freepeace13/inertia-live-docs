# Upgrading from 0.x to 1.0

What changes when you move from the 0.1.x packages to 1.0. Server and client are independent from 1.0: a 1.x client works with any 1.y server and the reverse ([Deprecation](../release-policy.md#deprecation)). Upgrade both from 0.1 in the same window, because 0.1 and 1.0 do not promise to work together.

## Server (`freepeace13/inertia-live-laravel`)

1. **Namespace.** The PHP root namespace is `InertiaLive` instead of `Freepeace13\InertiaLive`. Replace it in `use` statements, for example `InertiaLive\Attributes\LiveTopic`, `InertiaLive\Concerns\EmitsLiveChanges`, `InertiaLive\Facades\Live`. The Composer package name is unchanged.
2. **Requirements.** PHP 8.3 or later, Laravel 12 or 13, Inertia 2 or 3, `spatie/laravel-event-sourcing` ^7.14. See [Compatibility](compatibility.md).
3. **Optional: the response factory.** `Inertia::render(...)->live($topic, only: [...])` still works. To drop the `only` list, mark each prop with `live($topic, $value)` and render through `InertiaLive::render()` ([Page bindings](bindings.md)).
4. **New: the cursors endpoint.** The package registers `GET /inertia-live/cursors` and `_live` advertises it. Authorizers registered with `Live::authorize()` are called there with the placeholder values as strings, so a callback that type-hints a bound model is denied at that endpoint and the client reloads the binding's props instead. Take string parameters. If your app caches routes or restricts routes, check that `inertia-live/cursors` is reachable with the session cookie. Config keys `cursors.path` and `cursors.middleware`; the default middleware is `['web', 'throttle:60,1']`.
5. **New: `protocol: 1`** on every signal. Nothing to do.
6. **New: `php artisan inertia-live:install`** and the `assertLive()` and `assertNotLive()` test assertions ([Testing](testing.md)).
7. **Experimental and internal.** `CursorRepository`, `LiveChangeBroadcast`, `Live::isPublic()`, `Live::hasAuthorizerFor()` and `liveChanged()` are experimental; `LiveManager`, `LiveBindings`, `ChangeBuffer` and the other classes in [Stable API](stable-api.md) are internal. Code that relies on them may need changes in a minor.

## Client (`@freepeace13/inertia-live-*`)

1. **Node 22 or later** to build the app.
2. **`ConnectionTracker` and `CursorStore` are no longer exported** from `@freepeace13/inertia-live-core`. They were internal. If you used them, copy the small logic you need or open an issue.
3. **Type imports.** `LiveStatus`, `EchoLike`, `Reloader` and `ReloadCancelled` can now be imported from the Vue or React package you already use.
4. **New option `fetchCursors`** on `LiveClient`, the Vue plugin and the React provider. The default `fetch` is right for a normal app. Supply your own for tests or a non-browser setup.
5. **Echo.** If you set up broadcasting with `php artisan install:broadcasting`, you can pass `echo()` from `@laravel/echo-vue` or `@laravel/echo-react` and delete the hand-written Echo instance ([Installation](installation.md#client)).
6. **Tests.** `createFakeLive()` gains `fetchCursors`, `cursorRequests`, `setServerCursor()` and `failCursors()`, and `emit()` takes an optional `protocol` argument.
7. **Experimental.** `createInertiaReloader`, `inertiaReloader`, `withSocketId`, `SOCKET_ID_HEADER`, `EchoChannelLike`, `ConnectionLike` and `resetPause()` are experimental.

## Behavior changes to know

- After a channel's subscription is confirmed, the client asks the cursors endpoint instead of re-reading `_live` through your controller: one cheap request per burst of new subscriptions instead of a partial render. If the endpoint is unreachable it falls back to the old re-read, so nothing breaks.
- A signal with a `protocol` newer than the client supports reloads every prop bound to that topic and calls `onError` once.
