# Testing

## PHP: `Live::fake()`

`Live::fake()` replaces the flusher with a recorder, so changes are captured instead of broadcast. Nothing touches the cursor store, rate limiter or broadcaster.

```php
use InertiaLive\Facades\Live;

it('signals the document topic once', function () {
    Live::fake();

    $this->post(route('documents.rename', $doc), ['title' => 'Q4 plan']);

    Live::assertChanged("documents.{$doc->uuid}", props: ['document']);
    Live::assertNothingChangedFor('documents.other-uuid');
    Live::assertChangedTimes("documents.{$doc->uuid}", 1);
});
```

| Assertion | Passes when |
| --- | --- |
| `assertChanged($topic, ?array $props = null)` | At least one flushed change for the topic exists and, if `$props` is given, one of them includes all of those props |
| `assertNothingChangedFor($topic)` | No flushed change for the topic |
| `assertChangedTimes($topic, $times)` | Exactly `$times` flushed changes for the topic. Proves coalescing: one request with many events should be `1` |

A change is "flushed" when the request, job or command ends (or when the flusher is called). In feature tests that is the end of the simulated request.

`Live::authorize()`, `Live::publicTopic()`, `Live::hasAuthorizerFor()` and `Live::canSubscribe()` still delegate to the real manager under the fake. `Live::fake()` can be called repeatedly and does not rebind `LiveManager`, so resolving `ChangeFlusher` or `LiveManager` afterwards still returns the real classes.

### Asserting a page's bindings

Controller tests can check which topic and props a page keeps live. `assertLive()` and `assertNotLive()` are available on the `assertInertia()` callback argument and directly on the test response (since 1.0, D-011 item 4).

```php
use Inertia\Testing\AssertableInertia;

it('keeps the document page live', function () {
    $this->get(route('documents.show', $doc))
        ->assertInertia(fn (AssertableInertia $page) => $page
            ->component('Documents/Show')
            ->assertLive("documents.{$doc->uuid}", ['document', 'activity']));

    // or, without assertInertia:
    $this->get(route('documents.show', $doc))->assertNotLive('folders.other');
});
```

| Assertion | Passes when |
| --- | --- |
| `assertLive($topic, ?array $props = null)` | The page has a binding for the topic and, if `$props` is given, the binding lists exactly those props (order does not matter) |
| `assertNotLive(?string $topic = null)` | The page has no binding for the topic, or with no argument no bindings at all |

They read the `_live` prop, so they work for pages built with `InertiaLive::render()` and with `->live()`, and for an Inertia JSON request as well as a full page load.

### Testing without the fake

For lower-level tests, resolve `ChangeBuffer`, `TopicResolver`, `ChangeFlusher` or `CursorRepository` from the container. The package's own suite uses Pest with Orchestra Testbench, SQLite in memory and Spatie's migrations; fixtures live in `tests/Fixtures` of the Laravel repository.

## Client: `createFakeLive()`

Every adapter has a `createFakeLive()` that returns a fake Echo plus helpers to drive it.

| Import | Extra member for wiring |
| --- | --- |
| `@freepeace13/inertia-live-core/testing` | `echo`, `reload` for `new LiveClient({...})` |
| `@freepeace13/inertia-live-vue/testing` | `options` for `app.use(InertiaLive, fake.options)` |
| `@freepeace13/inertia-live-react/testing` | `providerProps` for `<InertiaLiveProvider {...fake.providerProps}>` |

The Vue and React versions also accept `{ debounceMs }`. All accept `{ channelPrefix }` (default `'live'`).

### Fake API

| Member | Description |
| --- | --- |
| `echo` | Fake Echo implementing `private`, `channel`, `leave` and a Pusher-like connection |
| `reload` | Reloader that records the call |
| `reloads` | `string[][]`: every `only` list passed to the reloader, in order |
| `joined` | `Set<string>` of channels currently joined |
| `left` | Channels left, in order |
| `emit(topic, version, props = [], protocol?)` | Deliver a change signal as if broadcast on `{prefix}.{topic}`. Pass `protocol` to simulate a server that sends the field; omit it to simulate one that does not |
| `setSocketId(id)` | What `echo.socketId()` reports, for sender-exclusion tests |
| `fetchCursors`, `cursorRequests` | The fake cursors endpoint, already wired into `options` / `providerProps`, and every `{ url, topics }` request made to it |
| `setServerCursor(topic, cursor)` | The cursor the fake server reports for a topic (default `0`); `undefined` leaves the topic out of the answer |
| `failCursors(failing = true)` | Make the cursors endpoint fail, to test the fallback to re-reading `_live` |
| `confirmSubscription(topic)` | Simulate the server confirming the subscription (triggers the missed-signal check) |
| `setConnectionState(state)` | Simulate `connected`, `connecting`, `unavailable`, `disconnected`, `failed` |

### Example (core)

```ts
import { LiveClient } from '@freepeace13/inertia-live-core'
import { createFakeLive } from '@freepeace13/inertia-live-core/testing'

const fake = createFakeLive()
const client = new LiveClient({ echo: fake.echo, reload: fake.reload, debounceMs: 0 })

client.sync({
  bindings: [{ topic: 'documents.a', channel: 'live.documents.a', props: ['document'], cursor: 0 }],
})

fake.emit('documents.a', 1)
expect(fake.reloads).toEqual([['document']])

fake.emit('documents.a', 1) // stale: version <= cursor
expect(fake.reloads).toHaveLength(1)
```

Use `debounceMs: 0` for immediate reloads, or wait past the debounce window (150 ms by default) before asserting.

### Testing reconnects

```ts
fake.setConnectionState('unavailable')
fake.setConnectionState('connected') // triggers one refresh() of all bound props
```

## Running the repository's own tests

```bash
# server: https://github.com/freepeace13/inertia-live-laravel
composer install && composer test   # also: composer lint, composer analyse

# client: this repository, from the root
pnpm install && pnpm test           # also: pnpm typecheck, pnpm lint
```
