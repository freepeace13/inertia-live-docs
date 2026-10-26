# Phase 1 audit

Read-only review of the code at client 0.1.1 and Laravel 0.1.0, plus the demo app as a real consumer. It lists what is awkward, what is a defect and what could be simplified, with the breaking-change impact of each. It decides nothing: the 1.0 scope (D-011), the `protocol` field (D-012) and the response factory (D-015) are still open. Not done yet: a fresh-app install walkthrough (needs a clean Laravel project) and running the demo.

Impact key: **none** = additive or internal, **minor** = breaks a narrow use, **breaking** = changes a documented API.

## Defects found

Both are fixed (2026-10-27): defect 1 in the client repo (stub completed and typed over `LiveClient`'s public members, with a `useLive()` assertion in the server test), defect 2 in `v1/installation.md`.

1. **Vue SSR crashes in `useLive()`.** The inert client that `InertiaLive` provides on the server (`packages/vue/src/plugin.ts`, `serverClient()`) has no `stale` or `onStale`, but `useLive()` reads `client.stale` and calls `client.onStale(...)`. During SSR that throws `client.onStale is not a function`. The `as unknown as LiveClient` cast hides the gap from the compiler, and no test renders on the server. Fix: add the two members, drop the cast (type the stub as `Pick` of `LiveClient` or share one inert client from core), and add an SSR test. Impact: none. Confirm by reproducing before fixing.
2. **`pusher-js` is missing from the install command.** `installation.md` imports `pusher-js` in the Echo example, but the `npm install` lines list only the adapter and `laravel-echo`. Impact: docs only.

## Setup and install path

3. **Install is four manual steps with no command.** `composer require`, an `npm install` of the adapter and Echo (plus `pusher-js`), Echo configuration (about ten lines), and wiring the plugin or provider. Broadcasting and channel authorization come first if the app has none. Candidates: `php artisan inertia-live:install` (checks broadcasting and cache store, publishes config, prints the client snippet); a documented one-page quickstart checked against the demo. Impact: none.
4. **The Echo block is the real boilerplate.** The adapter wiring is already one line (`app.use(InertiaLive, { echo })` or `<InertiaLiveProvider echo={echo}>`). Check whether Echo's own Vue and React helpers can create the instance before writing one ourselves. Not verified.
5. **Cursor store requirement is easy to miss.** Cursors need a cache store with atomic `increment`; `file` with concurrent workers is wrong. Nothing checks this at boot. Candidate: a boot-time warning when the resolved store is `file` (or `array` outside tests). Impact: none.
6. **Config is small and mostly sane.** Seven keys (`enabled`, `channel_prefix`, `cursor_store`, `cursor_ttl`, `max_signals_per_second`, `replay.suppress`, `replay.final_signal`, `debug`) all have defaults, so zero-config already works. `debug` defaults to `APP_DEBUG`; consider whether it should log through a dedicated channel. Impact: none.

## PHP API

7. **The topic and its props are written twice.** In the demo, `#[LiveTopic('documents.{documentUuid}', props: ['comments'])]` sits on the event and `->live("documents.{$document->uuid}", only: ['document', 'comments'])` sits in the controller. The topic string is repeated and the `only` list must be kept in sync by hand. This is the case for a response factory (D-015). Candidates B and C remove the duplication; A does not.
8. **`->live()` has no IDE or static-analysis hint.** It is a macro, so editors and PHPStan do not know `Response::live()`. Candidate: ship a `@method` stub or an ide-helper file. Impact: none.
9. **`only` must be non-empty and is a bare array.** The empty case throws at runtime. Candidate C removes the case; otherwise keep the throw. Impact: none for C, minor if `only` becomes optional.
10. **`EmitsLiveChanges` overrides `handle()`.** It wraps Spatie's `HandlesEvents::handle(StoredEvent)` to run after handlers return. That works, but it depends on Spatie's internals and conflicts with any other trait or class that overrides `handle()`. Candidate: also offer a listener or Projectionist hook that does not need a trait. Impact: none if additive; the trait stays.
11. **`LiveFake` duplicates `LiveManager`'s methods.** `authorize`, `publicTopic`, `isPublic` and `hasAuthorizerFor` are copied across and the facade docblock lists them. A new manager method must be added in three places. Candidate: make the fake extend or decorate the manager. Impact: none.
12. **There is no way to assert a page's live bindings.** `Live::fake()` asserts what was broadcast, but a controller test cannot easily say "this response is bound to topic X for props Y" (the `_live` prop). Candidate: an assertion helper that works with Inertia's `assertInertia`. Impact: none.
13. **Most classes are `final` and only `CursorRepository` is a contract.** That keeps the surface small. For 1.0 decide the stable list explicitly: facade methods, `LiveTopic`, `EmitsLiveChanges` (including whether protected `liveChanged()` is public API), `->live()`, config keys, `CursorRepository`, `FlushesChanges`. Everything else becomes `@internal`. Impact: minor if anything is later made internal.

## JavaScript API

14. **`core` exports internals.** `ConnectionTracker` and `CursorStore` are exported from the entry but are implementation details of `LiveClient`. Freezing them at 1.0 constrains refactors. Candidate: remove them from the public entry before 1.0. Impact: minor (anyone importing them).
15. **Adapter options are copy-pasted.** `InertiaLiveOptions` (Vue) and `InertiaLiveProviderProps` (React) repeat `echo`, `debounceMs`, `connection`, `maxWaitMs`, `reload`, `onError` with the same comments, and `LiveClientOptions` repeats them again. Candidate: one shared options type in `core`. Impact: none.
16. **The two testing fakes are the same file.** `vue/src/testing.ts` and `react/src/testing.ts` differ only in the options type they satisfy. Fine as is; if a third adapter appears, move the shared part into `core/testing`. Impact: none.
17. **React creates its client in an effect.** The first render has a `null` client and `useLive()` reports `connecting` until the effect runs. That is correct for StrictMode and SSR, but document it as the intended behavior. Impact: none.
18. **No `protocol` in the signal.** `ChangeSignal` is `{ topic, version, props }` and `Binding` has no version marker. Adding `protocol` later is easier before 1.0 (D-012). Impact: breaking at the wire level if it becomes required, so add it as optional first, or bump the minor on both sides.

## Testing story

19. **JS:** `createFakeLive()` in `core/testing`, `vue` and `react` is solid: it covers emit, reloads, joined and left channels, socket id, subscription and connection state. Gap: no SSR test (see defect 1) and no end-to-end test against a browser.
20. **PHP:** `Live::fake()` plus `assertChanged`, `assertNothingChangedFor` and `assertChangedTimes` cover the projector side. The response side is untested by consumers (item 12). The package's own suite has 16 test files across authorization, consistency, cursors, replay and config.
21. **No cross-repo contract test.** Nothing proves that a signal the Laravel side emits is one the client accepts. Phase 3 plans shared fixtures; this audit confirms the need.

## Suggested candidates for the 1.0 scope (not decided)

Likely worth doing: defects 1 and 2; the response factory (D-015); `protocol` (D-012); un-exporting `core` internals (14); `@internal` marking and the stable-API list (13); a binding assertion (12); an install command or checked quickstart (3).
Cheap and optional: shared options type (15), IDE stub (8), cursor-store warning (5), `LiveFake` delegation (11).
Deliberately left alone unless asked: the `EmitsLiveChanges` hook design (10) beyond adding an alternative.
