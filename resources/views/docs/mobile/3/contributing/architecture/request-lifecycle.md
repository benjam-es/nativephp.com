---
title: Request Lifecycle
order: 700
---

## Overview

When your app runs on a device, there is no network server. PHP runs embedded directly on the device, and requests flow
through the native runtime into Laravel's HTTP kernel. Understanding this flow is important when debugging performance
issues or contributing to the bootstrap layer.

## Request Flow

1. **WebView** triggers a request (navigation, form submission, JS fetch)
2. **Platform bootstrap** (`bootstrap/ios/native.php` or `bootstrap/android/native.php`) initializes the PHP
   environment, loads the Composer autoloader, and boots the Laravel application
3. **Laravel kernel** processes the request through middleware, routing, and controllers as usual
4. **Edge middleware** (`RenderEdgeComponents`) runs after the response is generated, serializing the component tree
   and sending it to the native layer via `Edge.Set`
5. **Response** is returned to the WebView with HTTP headers and content

Each request goes through this full cycle. The bootstrap files handle cookie parsing, query parameter extraction,
and POST body handling before Laravel sees the request.

## Platform Differences

### iOS

- Uses a custom `Request` class (`Native\Mobile\Support\Ios\Request`) for request capture
- Custom cookie parsing via `parse_str(str_replace('; ', '&', ...))`
- Manual `$_GET` and `$_POST` population from server variables
- `PhpUrlGenerator` handles the `php://` protocol scheme for URL generation
- Timing breakdown logged to Xcode console via `error_log()`

### Android

- Uses Laravel's standard `Illuminate\Http\Request::capture()`
- OPcache status monitoring -- reports enabled/disabled state and cached script count
- Explicit kernel bootstrap step (`$kernel->bootstrap()`) separate from request handling
- Enhanced error logging with full exception traces for debugging
- Timing breakdown logged to logcat via `error_log()`

### Both Platforms

- `X-PHP-Timing` headers are added to every response with performance breakdown (autoload, bootstrap, handle, total)
- Performance timing starts before autoloading and tracks each phase
- Run entirely on-device with no network server involved
- Request data is passed through the native PHP extension

## Native API Facades

NativePHP exposes 16 facades that follow a consistent PendingBuilder pattern for calling into native code:

```
Facade -> Class -> PendingBuilder -> nativephp_call()
```

For example, `Camera::getPhoto()` creates a `PendingPhotoCapture` instance. Builder methods like `id()`, `event()`,
and `remember()` return `$this` for fluent chaining. The terminal method (e.g., `start()`) calls `nativephp_call()`.

If you don't explicitly call the terminal method, `__destruct()` calls it automatically:

```php
Camera::getPhoto(); // Auto-executed on destruct
```

Or use the builder fluently:

```php
Camera::getPhoto()
    ->id('avatar-upload')
    ->event(AvatarPhotoTaken::class)
    ->remember()
    ->start();
```

The full list of facades: Biometrics, Browser, Camera, Device, Dialog, File, Geolocation, Haptics, Microphone,
MobileWallet, Network, PushNotifications, Scanner, SecureStorage, Share, System.

## Contributing: Adding a New Native API

<aside>

If you're adding a significant new device capability (Bluetooth, NFC, ML, maps, payments), it belongs in a
[plugin](/docs/mobile/3/plugins/creating-plugins), not the core package. Plugins can be developed, versioned, and
distributed independently.

</aside>

Core API additions are best for small, universally-needed features that tightly integrate with the existing bridge.
Here's the process:

### Step 1: Create the API Class

Create a class in `src/` that acts as the entry point. Each method returns a PendingBuilder:

```php
// src/Clipboard.php
namespace Native\Mobile;

class Clipboard
{
    public function copy(string $text): PendingClipboardCopy
    {
        return new PendingClipboardCopy(['text' => $text]);
    }
}
```

### Step 2: Create the Facade

Add a facade in `src/Facades/` with the standard accessor:

```php
// src/Facades/Clipboard.php
namespace Native\Mobile\Facades;

use Illuminate\Support\Facades\Facade;

class Clipboard extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return \Native\Mobile\Clipboard::class;
    }
}
```

### Step 3: Create the PendingBuilder

Builder methods return `$this`. The terminal method calls `nativephp_call()`. The `__destruct()` method auto-executes
if the terminal method was never called:

```php
// src/PendingClipboardCopy.php
namespace Native\Mobile;

class PendingClipboardCopy
{
    protected bool $started = false;

    public function __construct(
        protected array $options = []
    ) {}

    public function start(): bool
    {
        if ($this->started) {
            return false;
        }

        $this->started = true;

        if (function_exists('nativephp_call')) {
            $result = nativephp_call('Clipboard.Copy', json_encode($this->options));

            if ($result) {
                $decoded = json_decode($result, true);
                return isset($decoded['status']) && $decoded['status'] === 'success';
            }
        }

        return false;
    }

    public function __destruct()
    {
        if (! $this->started) {
            $this->start();
        }
    }
}
```

### Step 4: Create Events

For async operations, create event classes in `src/Events/` grouped by API name. Event constructors receive their
payload as individual arguments -- this is how `DispatchEventFromAppController` instantiates them:

```php
// src/Events/Clipboard/TextPasted.php
namespace Native\Mobile\Events\Clipboard;

class TextPasted
{
    public function __construct(
        public string $text
    ) {}
}
```

### Step 5: Add Tests

Write tests in `tests/` to cover the API class, facade, and PendingBuilder. Use the `Tests\TestCase` base class:

```php
it('creates a pending clipboard copy with options', function () {
    $pending = new PendingClipboardCopy(['text' => 'Hello']);

    expect($pending)->toBeInstanceOf(PendingClipboardCopy::class);
});
```
