# Page bindings

A **live binding** says: "these props on this page depend on this topic." Declare it in the controller by marking the props with `live()` and rendering through `InertiaLive::render()` (since 1.0, [D-019](../decision-log.md)), or with the lower-level `->live()` macro on the Inertia response.

```php
use InertiaLive\InertiaLive;

use function InertiaLive\live;

return InertiaLive::render('Documents/Show', [
    'document' => live("documents.{$doc->uuid}", DocumentResource::make($doc)),
    'activity' => live("documents.{$doc->uuid}", fn () => $doc->activity()->latest()->limit(20)->get()),
    'title' => 'Documents', // not live
]);
```

## `InertiaLive::render()` and `live()`

```php
InertiaLive::render(string $component, array $props = []): Inertia\Response
live(string $topic, mixed $value): InertiaLive\LiveProp
```

- A prop wrapped in `live()` depends on that topic. The props that share a topic become one binding, so there is no `only` list to keep in step with the props.
- The factory unwraps the values before Inertia sees them, so closures, `Inertia::optional()`, `Inertia::defer()` and plain values all work inside `live()`, and partial reloads behave as usual.
- Props that are not wrapped are not live. A page with no live prop has no `_live` prop.
- A prop belongs to one topic. For a prop that depends on two topics, use `->live()` below.
- `live()` is a namespaced function (`use function InertiaLive\live;`) so it cannot clash with other helpers. Whether the channel is public follows `Live::publicTopic()`, as for `->live()`.

## `->live()`

The low-level path, which stays supported next to the factory. The same page:

```php
return Inertia::render('Documents/Show', [
    'document' => DocumentResource::make($doc),
    'activity' => fn () => $doc->activity()->latest()->limit(20)->get(),
])->live("documents.{$doc->uuid}", only: ['document', 'activity']);
```

```php
live(string $topic, array $only): Inertia\Response
```

| Argument | Meaning |
| --- | --- |
| `$topic` | The resolved topic name (not the template) |
| `$only` | Prop keys that depend on the topic. An empty list throws: the client would have nothing to reload |

Call it more than once to bind several topics:

```php
return Inertia::render('Documents/Show', [...])
    ->live("documents.{$doc->uuid}", only: ['document'])
    ->live("folders.{$doc->folderUuid}", only: ['siblings']);
```

## The `_live` prop

Either way, one prop, `_live`, is added to the response:

```json
{
  "_live": {
    "bindings": [
      {
        "topic": "documents.9f1c…",
        "channel": "live.documents.9f1c…",
        "props": ["document", "activity"],
        "cursor": 4126,
        "public": false
      }
    ],
    "cursors": "/inertia-live/cursors"
  }
}
```

| Field | Meaning |
| --- | --- |
| `topic` | Topic name |
| `channel` | `{channel_prefix}.{topic}`, the name passed to Echo |
| `props` | Props to reload when the topic changes |
| `cursor` | The topic's latest sequence number at render time; signals at or below it are already reflected |
| `public` | Whether to use a public channel |

Details worth knowing:

- `_live` is wrapped in `Inertia::always()`, so it is present on partial reloads even when `only` lists other props. That is how cursors stay fresh after each live reload.
- It is resolved lazily on every render, so each render reads the current cursor from the `CursorRepository`.
- The client treats a changed `_live` object as a navigation: it joins new channels, leaves removed ones and raises cursors.
- Pages with no live props have no `_live` prop and open no subscriptions.

## Partial reloads and lazy props

The client reloads with `router.reload({ only: [...] })`, so only the listed props are re-evaluated. Closure props (`fn () => ...`) in your controller are only run when requested, which makes live reloads cheap for expensive props. `router.reload` keeps scroll position and component state by default, so a live update does not disturb the user.

Props listed in `only` must be real prop keys of that page.
