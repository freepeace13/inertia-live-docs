# Phase 1 audit

Read-only review of the code at client 0.1.1 and Laravel 0.1.0, plus the demo app as a real consumer. It lists what is awkward, what is a defect and what could be simplified, with the breaking-change impact of each. It decides nothing itself. The 1.0 scope (D-011) and the `protocol` field (D-012) were decided on 2026-10-28; so was the response factory as live props (D-015, D-019). The fresh-app install walkthrough and a live Reverb round trip were done on 2026-10-28 (section below). Not done yet: running the demo.

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

## Fresh-app install walkthrough (2026-10-28)

Done in a scratch app: `composer create-project laravel/vue-starter-kit` (resolved Laravel 12.70, Inertia 2.0.28, Vue starter with npm), then the published packages (Laravel `0.1.0`, client `0.1.1`), following `v1/installation.md`. Verified: server install, discovery, Spatie projector registration, `->live()` response and `_live` prop, a flushed change through `Live::fake()`, and a client build with the plugin wired. Also verified with a live Reverb server (below). Not verified: Laravel 13, Inertia 3, React, a browser.

Steps a user had to do, in order: 1. `composer require`. 2. Publish Spatie's migrations and migrate. 3. `php artisan install:broadcasting --reverb`. 4. `npm install` the adapter, `laravel-echo` and `pusher-js`. 5. Edit `app.ts` (two lines). 6. Write the event, projector, route and `Live::authorize()`. Steps 2 and 5 are where the docs and reality differ.

22. **Spatie's migrations are not mentioned.** `composer require` pulls in `spatie/laravel-event-sourcing` but its `stored_events` and `snapshots` migrations are not published or run. The first dispatched event fails with `no such table: stored_events`. `installation.md` never says to publish them. Candidates: add the step to the docs now; an install command (item 3) could publish and migrate. Impact: docs only.
23. **The documented Echo block is not the path a new app takes.** The starter kit's `install:broadcasting` already adds `configureEcho({ broadcaster: 'reverb' })` from `@laravel/echo-vue`, which also exports `echo()` returning the configured `Echo` instance. The wiring is then `.use(InertiaLive, { echo: echo() })`. Writing `resources/js/echo.ts` by hand, as the docs show, would create a second Echo instance next to it. This settles item 4: Echo's own helper does create the instance, and the adapter needs no Echo boilerplate of ours. Candidates: lead the docs with the `@laravel/echo-vue` and `@laravel/echo-react` path and keep the manual block for apps that have no helper. Impact: docs only. Check that `echo()` is safe to call before the first connection and under SSR.
24. **The docs assume Pest.** The starter kit ships PHPUnit, so `it(...)` in `testing.md` is undefined there. Candidate: show the PHPUnit form beside the Pest one. Impact: docs only.
25. **`Live::fake()` gives no hint when the event was dispatched outside a request.** Changes flush at the end of a request, job or command, so `event(new X)` in a test body gives "none was flushed" with no pointer to why. The docs say it, but the message does not. Candidate: mention the flush rule in the assertion message, or expose a `Live::flush()` for tests. Impact: none.
26. **Item 5 is lower risk than first thought.** The starter kit defaults `CACHE_STORE=database`, which supports atomic `increment`, so a default app is fine. The risk stays for apps that chose `file`.
27. **Counted effort.** Six manual steps and about 25 lines of app code (event, projector, route, authorizer) before a page is live, with the client edit two lines. The install command (item 3) would remove steps 2 to 4 and print step 5.
28. **Reverb round trip works (Reverb server, the published `LiveClient` from core in Node, a real session and `/broadcasting/auth`).** Results: the private channel authorized (200); a POST that fired the event delivered a signal and one reload of `["note"]`; two rapid POSTs coalesced into one reload; a topic whose authorizer returns false got a 403 and zero reloads; an event on a topic with no authorizer was stored but not broadcast and logged `Inertia Live topic has no authorizer; signal not sent.` (fails closed). Observation, not a defect: every subscribe causes one extra reload of `["_live"]` (the catch-up read in client-core step 7), so each page view costs a second request even when nothing changed. Correction (2026-10-28): the first suggestion, skipping it when the cursor matches the one the client holds, does not work: at subscribe time the client only holds the render-time cursor, and the read exists to compare that with the server's current one. Decided in D-020: replace it with a lighter cursors-only read from a small package endpoint. See `v1/protocol.md`. Not tested: reconnect catch-up, a queued projector, replay, the rate limit.

## Response factory spike (2026-10-28, D-019)

Throwaway prototype in the scratch app, not in the package: a `LiveProp` value object, a `live($topic, $value)` helper and an `InertiaLive::render()` that unwraps live props, calls `Inertia::render()` with the inner values, then calls `->live($topic, only: [...])` once per topic. Run on Inertia 2.0.28 and 3.5.2 (Laravel 12.70, PHP 8.4).

29. **The unwrap-in-factory design works on both Inertia versions.** Passed on both: props sharing a topic bind as one binding (`t.one` lists `a`, `b`) and a second topic binds separately; unwrapped props and plain closures are left alone; a partial reload for one prop returns only that prop and still carries `_live`; a page with no live props gets no `_live`. Wrapped Inertia props keep their own meaning on Inertia 3: `optional` is omitted on the first visit, `defer` is listed under `deferredProps`, `always` is present. Because the factory never asks Inertia to resolve a custom type, no version-specific code was needed.
30. **Limits found, to settle in the implementation task.** A prop can have only one topic with `live($topic, $value)`; a prop that depends on two topics needs `live(['a', 'b'], ...)` or the `->live()` escape hatch. `live()` as a global function name may clash; the package should namespace it (`InertiaLive\live`) or use a static `Live::prop()`. Not tested: SSR, `Live::fake()` with the factory, nested props, Laravel 13, the binding assertion (item 12). Key order of `_live` differs between Inertia 2 and 3 (harmless).

## Suggested candidates for the 1.0 scope (not decided)

Likely worth doing: defects 1 and 2; the response factory (D-015); `protocol` (D-012); un-exporting `core` internals (14); `@internal` marking and the stable-API list (13); a binding assertion (12); an install command or checked quickstart (3, 22, 23).
Cheap and optional: shared options type (15), IDE stub (8), cursor-store warning (5), `LiveFake` delegation (11).
Deliberately left alone unless asked: the `EmitsLiveChanges` hook design (10) beyond adding an alternative.
