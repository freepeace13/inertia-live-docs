# Quickstart

From `composer require` to a page that updates by itself, in about five minutes. It assumes a Laravel 12 or 13 app with Inertia (Vue or React) and Reverb or Pusher broadcasting already set up. For the details behind each step, follow the links.

The server half of this page was run in a fresh Laravel app on 2026-10-29. The finished code is in the [demo app](https://github.com/freepeace13/inertia-live-demo).

## 1. Install

```bash
composer require freepeace13/inertia-live-laravel
php artisan inertia-live:install --client=vue    # or --client=react
```

The command runs Spatie's migrations, publishes the config, checks broadcasting and prints the client snippet. See [Installation](installation.md).

## 2. Mark the event

An event says which topic it changes and which page props it affects ([Topics](topics.md)):

```php
use InertiaLive\Attributes\LiveTopic;
use Spatie\EventSourcing\StoredEvents\ShouldBeStored;

#[LiveTopic('documents.{documentUuid}', props: ['document'])]
final class DocumentRenamed extends ShouldBeStored
{
    public function __construct(
        public readonly string $documentUuid,
        public readonly string $title,
    ) {}
}
```

## 3. Let the projector send signals

Add the trait to the projector that handles the event ([Projectors](projectors.md)):

```php
use InertiaLive\Concerns\EmitsLiveChanges;
use Spatie\EventSourcing\EventHandlers\Projectors\Projector;

final class DocumentProjector extends Projector
{
    use EmitsLiveChanges;

    public function onDocumentRenamed(DocumentRenamed $event): void
    {
        Document::whereKey($event->documentUuid)->update(['title' => $event->title]);
    }
}
```

## 4. Authorize the topic

A topic with no authorizer is never broadcast ([Authorization](authorization.md)). In `AppServiceProvider::boot()`:

```php
use InertiaLive\Facades\Live;

Live::authorize('documents.{uuid}', fn ($user, string $uuid) => Document::whereKey($uuid)->exists());
```

## 5. Bind the page

Mark the props that depend on the topic and render through `InertiaLive::render()` ([Page bindings](bindings.md)):

```php
use InertiaLive\InertiaLive;

use function InertiaLive\live;

public function show(Document $document): Response
{
    return InertiaLive::render('Documents/Show', [
        'document' => live("documents.{$document->uuid}", $document->only('uuid', 'title')),
    ]);
}
```

## 6. Connect the client

Paste what the install command printed. With Vue, after `.use(plugin)`:

```ts
import { echo } from '@laravel/echo-vue'
import { InertiaLive } from '@freepeace13/inertia-live-vue'

app.use(InertiaLive, { echo: echo() })
```

With React, in a persistent layout: `<InertiaLiveProvider echo={echo()}>{children}</InertiaLiveProvider>`. See [Vue 3](vue.md) and [React](react.md).

## 7. See it work

Start the app, Reverb (`php artisan reverb:start`) and Vite, open the page in two browser tabs, and rename the document in one. The other tab updates without a reload, and only the `document` prop is fetched.

## 8. Test it

The signal is sent when the request ends, so change the data through a request ([Testing](testing.md)):

```php
use App\Domain\Documents\DocumentCreated;
use App\Models\Document;
use Inertia\Testing\AssertableInertia;
use InertiaLive\Facades\Live;

it('keeps the document page live and signals a rename', function () {
    Live::authorize('documents.{uuid}', fn ($user, string $uuid) => Document::whereKey($uuid)->exists());
    Live::fake();
    event(new DocumentCreated('d1', 'First'));

    $this->put('/documents/d1', ['title' => 'Renamed'])->assertRedirect();

    Live::assertChanged('documents.d1', props: ['document']);

    $this->get('/documents/d1')->assertInertia(fn (AssertableInertia $page) => $page
        ->component('Documents/Show')
        ->assertLive('documents.d1', ['document']));
});
```

An event dispatched directly in a test, outside any request, is not flushed until the request, job or command that carries it ends, so `Live::assertChanged()` needs a request in between.

## If something does not update

Start with [Troubleshooting](troubleshooting.md). The usual causes are a broadcaster set to `log`, a missing authorizer, or the cursor cache set to `file`; `php artisan inertia-live:install` reports all three.
