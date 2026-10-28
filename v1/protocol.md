# Wire protocol (protocol 1)

The normative description of what the server sends and what the client must do with it. [SPEC.md](SPEC.md) explains the design; this file is the contract. If they disagree, this file wins for wire behavior.

The server and the client are separate packages and only meet at the points below: the broadcast channel, the signal, the `_live` page prop and the `X-Socket-ID` header. Anything not described here is an implementation detail and may change in a minor release.

> **Status.** This describes protocol 1 as it will be at 1.0. The 0.1.x packages follow it except where a section says **Planned for 1.0** or **Unreleased**. The contract tests are written: both repos check the shared fixture [`fixtures/protocol-1.json`](fixtures/protocol-1.json) (signal payloads, the `_live` prop, the cursors response and client behavior cases). The canonical copy is here; each repo carries an identical copy, and its test fails if it differs from this one when the docs repo is checked out alongside.

## Versioning

The protocol has one integer version. It is `1`.

- The server includes `protocol: 1` in every signal ([D-012](../decision-log.md)). Implemented on `main`, unreleased; 0.1.x servers do not send it.
- A client that receives a signal without `protocol` treats it as protocol 1. A server that never sends it is therefore a valid protocol 1 server. The field is optional at first; making it required is decided later.
- Receivers ignore fields they do not know. Adding an optional field is not a protocol change; removing or re-meaning a field, or changing a rule below, is, and it bumps the protocol version.
- A signal whose `protocol` is higher than the client supports is not ignored: the client reloads every prop bound to that topic and calls `onError` once for that topic ([D-020](../decision-log.md)). It stays subscribed.
- `protocol` appears on the signal only. `_live` does not carry it.

Until 1.0, a breaking wire change bumps the minor version on both packages (Composer `0.N.x` pairs with npm `0.N.x`). From 1.0, the protocol version is what tells the two apart.

## Topics and channels

A **topic** is a string such as `documents.9f1c2e`. Each topic maps to exactly one channel:

```
{channel_prefix}.{topic}        # default prefix: live  →  live.documents.9f1c2e
```

- A private topic (the default) uses a private channel. On the wire Echo adds `private-`, so the channel is `private-live.documents.9f1c2e`. The server authorizes the subscription through Laravel's channel authorization. A topic with no authorizer is not broadcast.
- A public topic (opt-in, per topic) uses a public channel with no authorization.
- Topics are built from an event's properties; placeholder values must be scalars and are cast to strings. Use non-guessable ids.

## The signal

The server broadcasts one event per changed topic, named `live.changed` (Echo listens for `.live.changed`, with the leading dot, because the server sets the name with `broadcastAs()`).

```json
{ "topic": "documents.9f1c2e", "version": 1789234567123456, "props": ["document", "activity"], "protocol": 1 }
```

| Field | Type | Meaning |
| --- | --- | --- |
| `topic` | string | The topic that changed. |
| `version` | integer | A per-topic sequence number, see [Versions and cursors](#versions-and-cursors). |
| `props` | list of strings | Prop keys the change affects. Empty means "unknown": every prop bound to the topic is affected. |
| `protocol` | integer, optional | Always `1`. Unreleased; absent on 0.1.x servers. |

A signal never carries model data. The page's data always comes back through its own controller.

The server must not send a signal before the change is committed and the projector handler has returned, and it excludes the sender's socket (see [X-Socket-ID](#x-socket-id)). Rules for coalescing, rate limiting and replays are in [Consistency](consistency.md).

## The `_live` page prop

A controller that binds topics adds the Inertia prop `_live`:

```json
{
  "bindings": [
    {
      "topic": "documents.9f1c2e",
      "channel": "live.documents.9f1c2e",
      "props": ["document", "activity"],
      "cursor": 1789234567123456,
      "public": false
    }
  ]
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `topic` | string | The bound topic. |
| `channel` | string | `{channel_prefix}.{topic}`. The client subscribes to this name. |
| `props` | list of strings, not empty | Page props that depend on the topic. |
| `cursor` | integer | The topic's latest issued version when the response was built. `0` if none was issued yet. |
| `public` | boolean | `true`: subscribe with `echo.channel`; otherwise `echo.private`. |

`_live` may also carry `cursors`, a string: the URL of the cursors endpoint (see [The cursors endpoint](#the-cursors-endpoint)). Unreleased (on `main`); absent on 0.1.x servers.

Rules:

- `_live` is present in every response for the page, including partial reloads (it is an Inertia "always" prop). The cursor is read when the response is built, after the page's other props.
- A page with no binding has no `_live` prop; the client then holds no subscriptions.
- A binding with an empty `props` list is invalid: the server throws.

## Versions and cursors

- A **version** is a number issued by the server for a topic every time it flushes a change. Each version for a topic is strictly greater than every one before it, across projectors, queue workers and processes.
- Versions are opaque. They are not event ids, are not comparable across topics, and are around `1.8e15` (clock-seeded), which is a safe integer in PHP and JavaScript. Clients must compare them numerically and must not parse them.
- A **cursor** is the latest version the receiver has seen for a topic. The server's copy is what `_live.bindings[].cursor` carries; each client keeps its own.
- The client **drops** a signal whose `version` is at or below its cursor for the topic and **accepts** one above it, advancing the cursor. It never moves a cursor backwards.
- On a fresh `_live`, the client raises its cursor to the binding's `cursor`. When it leaves a channel it forgets that topic's cursor.

Why a per-topic sequence and what it costs: [Design decisions](design-decisions.md#1-signal-versions-are-a-per-topic-sequence-not-event-ids).

## What the client does

1. **Subscribe.** For each binding in `_live`, join `channel` (private or public as `public` says) and listen for `.live.changed`. Leave channels that are no longer bound. Applying a new `_live` is idempotent.
2. **Receive.** Drop the signal if it is stale. Otherwise the affected props are `signal.props` intersected with the binding's `props`, or all of the binding's `props` when `signal.props` is empty. If none are affected, the cursor advances and nothing reloads.
3. **Reload.** Affected props join a pending set. After a quiet window (`debounceMs`, default 150 ms, never longer than `maxWaitMs` after the first signal) the client reloads that set once through the page's own route with a partial reload (`only`). Reloads never overlap: signals that arrive meanwhile queue one follow-up.
4. **Recover.** After the connection returns from `reconnecting` or `offline`, reload every bound prop once. A failed reload is retried with backoff up to five times, then the client reports `stale`.
5. **Close the subscribe gap.** A change can commit between the page render and the subscription. When a channel's subscription is confirmed, the client checks the server's current cursor for that topic. If it is ahead of the cursor the client holds, the props bound to the topic reload.
   - **Today (0.1.x):** the check is a re-read of `_live` alone through the page's route, which costs one extra request each time a new channel is subscribed.
   - **Unreleased ([D-020](../decision-log.md), [D-022](../decision-log.md)):** a small package endpoint that returns the current cursors for the given topics without running the page's controller. It applies the same authorization as the channel, because a cursor reveals that a topic changed. See [The cursors endpoint](#the-cursors-endpoint). A server that does not send `_live.cursors` is still valid: the client falls back to the `_live` re-read.

## The cursors endpoint

> Implemented on `main` ([D-022](../decision-log.md)), unreleased, and not in the 0.1.x packages. It specifies the endpoint named in [D-020](../decision-log.md), item 3.

**Purpose.** When a channel's subscription is confirmed, the client needs the server's current cursor for the topic to know whether a signal slipped in between render and subscribe. The endpoint answers that without running the page's controller.

**Discovery.** The server adds `cursors` to `_live`: a same-origin path such as `"/inertia-live/cursors"`. The client uses it as given and never builds the URL itself. If `_live.cursors` is absent, the client uses the `_live` re-read described above, so a new client works with an older server.

**Request.**

```
GET {_live.cursors}?topics[]=documents.9f1c2e&topics[]=folders.77a1
Accept: application/json
```

- A `GET`, so it is idempotent and needs no CSRF token. Sent with `fetch` (same origin, session cookies), not an Inertia visit, because it is not a page.
- The client batches every channel confirmed in the same burst into one request.
- At most 50 topics per request, and `topics` must be a list of 1 to 50 strings; anything else is a `422`.

**Response.** `200` with `Cache-Control: no-store, private` (more specific than D-022's `no-store`; the response depends on the signed-in user):

```json
{ "cursors": { "documents.9f1c2e": 1789234567123456 } }
```

- `cursors` maps each topic the caller may see to its current cursor, with the same meaning as `_live.bindings[].cursor` (`0` if none was issued).
- A topic the caller may not subscribe to, or that has no authorizer, is **omitted**. The client treats an omitted topic as "could not verify" and reloads that binding's props, which is safe because the reload goes through the page's own controller and policies.
- Any non-2xx response, or a network error, makes the client fall back to the `_live` re-read. A `429` from the throttle therefore moves load to the heavier fallback instead of shedding it.

**Authorization.** The endpoint runs the same callbacks that authorize the private channel for the signed-in user, so access to a cursor means access to the channel. Public topics need no check. The server keeps the callbacks registered with `Live::authorize()` and calls them directly rather than reaching into Laravel's broadcaster, which has no public check method. A request without a user can only read public topics. The callbacks are called with the topic's placeholder values as strings, so a callback that type-hints a bound model (which Laravel resolves for channel authorization) fails there; a callback that throws counts as a denial, and the client then reloads that binding's props. Write authorizers that take string parameters.

**Route and configuration.** The package registers one named route, `inertia-live.cursors`, at `inertia-live/cursors`. Path and middleware are configurable under `inertia-live.cursors.path` and `inertia-live.cursors.middleware`; the default middleware is `['web', 'throttle:60,1']`. A route cache and Octane work as for any package route.

**Client.** `LiveClient` gets an optional `fetchCursors` option (default: the global `fetch`), so tests and non-browser setups can supply their own, and the fake helpers stub it.

**Cost.** One cache read per topic, plus one authorizer call per topic (which may query a database), and no controller run, instead of a full Inertia partial render, once per burst of new subscriptions. With the default throttle of 60 requests a minute and up to 50 topics a request, one user can cause up to 3000 authorizer calls a minute, so keep authorizers cheap. Guests are throttled by IP address.

**What this adds to the stable API.** The route name and URL discovery field `_live.cursors`, the request and response shapes above, the two config keys, and a public `LiveManager` method that answers "may this user subscribe to this topic" (used by the endpoint and by `Live::fake()`).

## X-Socket-ID

Echo exposes the connection's socket id. The client adapters add it as the `X-Socket-ID` request header on every Inertia visit. The server excludes that socket when it broadcasts, so the user who made a change does not get a signal for it; they already receive fresh props from their own response.

A custom client that does not send the header will receive signals for its own changes. That is harmless but causes a redundant reload.

## Compatibility

| Client | Server | Result |
| --- | --- | --- |
| protocol 1, no `protocol` field | protocol 1 with `protocol: 1` | Works: unknown fields are ignored. |
| reads `protocol` | server without it | Works: a missing field means protocol 1. |
| any | different 0.N minor | Not supported before 1.0; pair the minors. |

## Open questions

- **Making `protocol` required.** To be decided with the contract tests.
