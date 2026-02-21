---
title: Native Bridge
order: 100
---

## Overview

The bridge is the core communication layer between PHP and the native platform (Swift/Kotlin). It handles two directions
of communication: PHP calling into native code, and native code dispatching events back to PHP.

## PHP to Native

PHP calls native functions synchronously via the custom PHP extension:

```php
nativephp_call('Camera.GetPhoto', json_encode(['id' => $id, 'event' => PhotoTaken::class]));
```

The first argument is the bridge function name (dot-notation), and the second is a JSON-encoded payload. The native side
executes the registered function and returns a JSON response.

You can check whether a bridge function is registered before calling it:

```php
if (nativephp_can('Camera.GetPhoto')) {
    nativephp_call('Camera.GetPhoto', json_encode($payload));
}
```

### HTTP Alternative

There's also an HTTP endpoint via `NativeCallController` at `/_native/api/call`. The JavaScript bridge uses this
endpoint, accepting `method` and `params` in the request body.

The controller validates the method name, checks registration with `nativephp_can()`, and proxies the call to
`nativephp_call()`. It returns structured JSON responses with `status`, `data`, and error codes when things go wrong:

```php
// NativeCallController validates and proxies to the native extension
$result = nativephp_call($method, json_encode($params));
$decoded = json_decode($result, true);

return response()->json([
    'status' => 'success',
    'data' => $decoded,
]);
```

This is the same mechanism that the JS bridge (`native.js`) uses under the hood. PHP facades call `nativephp_call()`
directly, bypassing the HTTP layer entirely.

## Native to PHP

When native operations complete (e.g., the user takes a photo), the native app dispatches an event by sending an HTTP
POST to `/_native/api/events` with the event class name and payload:

```json
{
    "event": "Native\\Mobile\\Events\\Camera\\PhotoTaken",
    "payload": ["photo-id-123", "/path/to/photo.jpg"]
}
```

`DispatchEventFromAppController` handles this by instantiating the event class with the payload spread into its
constructor:

```php
$event = new $event(...$payload);
event($event);
```

Event constructors receive their payload as **individual arguments**, not as an array. This means the order of the
payload array must match the constructor parameter order. There is no payload validation beyond PHP's own type checking
-- if the constructor signature doesn't match the payload, you'll get a standard PHP error.

Events are standard Laravel event classes dispatched through Laravel's event system. You can listen using all the usual
mechanisms -- listeners, subscribers, or the `#[OnNative]` attribute in Livewire:

```php
use Native\Mobile\Attributes\OnNative;
use Native\Mobile\Events\Camera\PhotoTaken;

#[OnNative(PhotoTaken::class)]
public function handlePhoto(string $id, string $path)
{
    // Handle the photo
}
```

## Contributing: Adding a Bridge Function

New bridge functions are typically added through the plugin system rather than the core package. Plugins define their
bridge functions in their `nativephp.json` manifest and provide the native implementations in Swift and Kotlin.

See the [Plugin docs](/docs/mobile/3/plugins/creating-plugins) for the full guide on building plugins with custom bridge
functions.

If you're adding a bridge function to the core package, the native implementations live in the Xcode and Android Studio
project templates (`resources/xcode/` and `resources/androidstudio/`). The PHP side uses the
[PendingBuilder pattern](request-lifecycle#contributing-adding-a-new-native-api) described in the Request Lifecycle page.
