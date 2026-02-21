---
title: Plugin System
order: 300
---

## Overview

Plugins are the **primary mechanism** for introducing and expanding native functionality. Large new device capabilities
(Bluetooth, NFC, ML, maps, payments, etc.) belong in plugins rather than the core package. This keeps the core lean and
lets features evolve independently.

A plugin is a Composer package with `type: nativephp-plugin` and a `nativephp.json` manifest that declares its bridge
functions, events, permissions, dependencies, and lifecycle hooks.

See the user-facing [Plugin docs](/docs/mobile/3/plugins/introduction) for the full guide on building plugins.

## Security Model

Plugins must be explicitly allowlisted in `App\Providers\NativeServiceProvider::plugins()`. This is a deliberate
security choice -- it prevents transitive Composer dependencies from silently registering native code into your app.

The flow works like this:

1. `PluginDiscovery` scans `vendor/composer/installed.json` for packages of type `nativephp-plugin`
2. It checks each package against the allowlist from `NativeServiceProvider::plugins()`
3. Only allowlisted plugins are loaded and returned by `discover()`
4. If no `NativeServiceProvider` exists or the `plugins()` method is missing, **all plugins are blocked**

```php
// App\Providers\NativeServiceProvider
public function plugins(): array
{
    return [
        \NativePhp\FirebasePlugin\FirebasePluginServiceProvider::class,
        \Vendor\MyPlugin\MyPluginServiceProvider::class,
    ];
}
```

The allowlist uses the plugin's Laravel service provider class name (from the package's
`extra.laravel.providers` in `composer.json`), not the Composer package name.

## Compilation Pipeline

When you run `native:run` or `native:install`, the plugin compilers process each registered plugin through four stages:

1. **Copy native source** -- `.kt` and `.swift` files are copied from the plugin package into the native project
2. **Generate bridge registration** -- Stub-based code generation creates the registration code that maps bridge
   function names to their native implementations
3. **Merge manifests** -- Permissions, dependencies, repositories, and Android components (activities, services,
   receivers, providers) are merged into the native project's configuration files
4. **Inject dependencies** -- Gradle dependencies, CocoaPods, and Swift Package Manager entries are added to build files

`AndroidPluginCompiler` and `IOSPluginCompiler` in `src/Plugins/Compilers/` handle their respective platforms. Both
compilers use the [file modification patterns](/docs/mobile/3/contributing/appendix/file-modification-patterns) extensively -- particularly stub-based
generation (Pattern 2) and build configuration injection (Pattern 6).

## Conflict Detection

Before compilation begins, both compilers call `PluginRegistry::detectConflicts()` to check for:

- **Namespace collisions** -- Two plugins using the same `namespace` in their manifest
- **Bridge function name conflicts** -- Two plugins registering the same bridge function name

If conflicts are found, a `PluginConflictException` is thrown with a message identifying the conflicting plugins and
suggesting how to resolve it:

```
Plugin conflicts detected:
Bridge function 'Camera.GetPhoto' is registered by both vendor/plugin-a and vendor/plugin-b

Unregister one of the conflicting plugins with: php artisan native:plugin:register <plugin> --remove
```

This check runs before any files are modified, so a conflict never leaves the native project in a broken state.

## Lifecycle Hooks

Plugins can run custom code at four points during the build process:

| Hook | When It Runs |
|------|-------------|
| `pre_compile` | Before plugin compilation starts |
| `post_compile` | After all plugins are compiled |
| `copy_assets` | During the asset copying phase |
| `post_build` | After the native build completes |

Hook commands extend `NativePluginHookCommand`, which provides helpers for common operations:

- **Platform detection** -- `platform()`, `isIos()`, `isAndroid()`
- **Path resolution** -- `buildPath()`, `pluginPath()`, `appId()`
- **File copying** -- `copyToAndroidAssets()`, `copyToAndroidRes()`, `copyToIosBundle()`, `copyToIosAssets()`
- **Downloads** -- `downloadFile()`, `downloadIfMissing()`, `downloadAndUnzip()`

See [Lifecycle Hooks](/docs/mobile/3/plugins/lifecycle-hooks) for the full reference.

## Contributing: Extending the Plugin System

### Adding a Manifest Field

The manifest schema is defined in `PluginManifest`. To add a new field:

1. Add the property to `PluginManifest` and update the constructor
2. Update the parsing and normalization logic (handle both old and new format if needed)
3. Update `PluginValidateCommand` to validate the new field
4. Update the relevant compiler(s) to consume it during compilation

### Plugin Compilers

`AndroidPluginCompiler` and `IOSPluginCompiler` translate manifest declarations into native project changes. They use
the [file modification patterns](/docs/mobile/3/contributing/appendix/file-modification-patterns) extensively -- especially stub-based generation for
bridge registration and build configuration injection for dependencies.

When modifying a compiler, keep the idempotency rule in mind: running a build twice must produce identical results.
Always check with `str_contains()` before injecting content into build files.

### Discovery and Registry

`PluginDiscovery` scans installed Composer packages and filters by the allowlist. `PluginRegistry` wraps discovery and
adds conflict detection, dependency aggregation, and convenience methods for querying plugins.

Changes to discovery affect how plugins are found. Changes to the registry affect validation and how the rest of the
system queries plugin data. Both classes cache their results -- call `clearCache()` or `refresh()` when testing changes
that modify the installed plugin set.
