---
title: Service Provider
order: 600
---

## Overview

`NativeServiceProvider` extends Spatie's `PackageServiceProvider` and bootstraps everything the package needs. It's the
single entry point where commands, singletons, middleware, components, and runtime configuration all come together.

## What It Registers

### Commands

19 artisan commands are registered in `configurePackage()`, covering the full lifecycle from installation through
packaging and plugin management:

```php
$package
    ->name('nativephp-mobile')
    ->hasConfigFile('nativephp')
    ->hasViews()
    ->hasRoute('api')
    ->hasCommands([
        PackageCommand::class,
        RunCommand::class,
        WatchCommand::class,
        InstallCommand::class,
        // ... 15 more
    ]);
```

### Plugin Singletons

Four plugin-related singletons are registered in `packageRegistered()`:

- `PluginDiscovery` -- Scans `vendor/composer/installed.json` for packages of type `nativephp-plugin`
- `PluginRegistry` -- Manages the set of registered plugins and checks for conflicts
- `AndroidPluginCompiler` -- Compiles Kotlin source and merges manifests into the Android project
- `IOSPluginCompiler` -- Compiles Swift source and merges manifests into the Xcode project

All four are bound as singletons so they maintain state across a single build run.

### Edge Components

Auto-discovered from `src/Edge/Components/` in `packageBooted()`. The `registerNativeComponents()` method recursively
scans the directory, converts class names to kebab-case, and registers them as Blade components with a `native-` prefix:

- `BottomNav` becomes `<native:bottom-nav>`
- `TopBarAction` becomes `<native:top-bar-action>`

No manual registration is needed when adding new Edge components.

### Middleware

`RenderEdgeComponents` is pushed globally onto the HTTP kernel. This middleware runs after every response, serializing
the Edge component tree to JSON and sending it to the native layer via `Edge.Set`.

### Blade Directives

Four conditional directives for platform-specific rendering:

- `@mobile` / `@web` -- Render content only on mobile or only on web
- `@ios` / `@android` -- Render content only on a specific mobile platform

These use the `System` facade under the hood (`System::isMobile()`, `System::isIos()`, `System::isAndroid()`).

### Filesystem Disks

Two disks are registered, but **only when running inside a native app** (`config('nativephp-internal.running')`):

- `mobile_public` -- A local disk pointed at `storage/app/public` with a URL prefix of `/_assets/storage`
- `temp` -- A local disk pointed at the platform's temp directory

### Vite Hot File

Platform-specific hot file paths (`ios-hot`, `android-hot`) are configured so both platforms can run simultaneously
with their own Vite dev servers. This only activates when running inside the native app.

### iOS URL Generator

On iOS (Darwin), the default URL generator is swapped with `PhpUrlGenerator` to handle the `php://` protocol scheme
used by the iOS runtime. This swap only happens when the app is running natively.

## Contributing: Adding Configuration Options

If your contribution introduces a new configuration value:

1. **Add it to `config/nativephp.php`** with a sensible default and `env()` support:

```php
'my_option' => env('NATIVEPHP_MY_OPTION', 'default-value'),
```

2. **Consume it in the relevant trait** using `config('nativephp.my_option')`. Read config values during execution,
   not in constructors or class properties.

3. **Apply to native projects if needed.** If the config affects native project files, use the appropriate
   [file modification pattern](/docs/mobile/3/contributing/appendix/file-modification-patterns) to inject the value during builds.
