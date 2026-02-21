---
title: Development Workflow
order: 400
---

## Setup

Clone the repository and install dependencies:

```bash
git clone git@github.com:nativephp/mobile-air.git
cd mobile-air
composer install
```

The package requires PHP 8.3+ and is tested against Laravel 10, 11, and 12.

## Quality Assurance

Run all checks with a single command:

```bash
composer qa
```

This runs formatting, static analysis, and tests in sequence. You should run this before every pull request.

### Individual Commands

| Command | What it Does |
|---------|-------------|
| `composer test` | Run the Pest test suite |
| `composer test-coverage` | Run tests with code coverage report |
| `composer analyse` | Run PHPStan via Larastan for static analysis |
| `composer format` | Run Laravel Pint for code formatting (PSR-12) |
| `composer qa` | Run all three: format, analyse, test |

### Testing Details

- **Framework:** Pest PHP v2 with Orchestra Testbench
- **Base class:** `Tests\TestCase` (sets up `NativeServiceProvider` and default config)
- **Suites:** `Unit` and `Feature` in the `tests/` directory
- **Fixtures:** Sample plugin manifests and test data live in `tests/Fixtures/`
- **Coverage:** The `src/Commands/` directory is excluded from coverage metrics, as commands primarily orchestrate
  calls to traits and other classes

## Code Style

NativePHP uses [Laravel Pint](https://laravel.com/docs/pint) with the default PSR-12 preset. Run `composer format`
to auto-fix formatting issues before committing.

Pint will handle:
- Spacing, indentation, and brace placement
- Import sorting and grouping
- Trailing commas and other stylistic choices

## Project Structure

```
src/
├── NativeServiceProvider.php    # Package bootstrap
├── Commands/                    # 19 artisan commands (native:* prefix)
├── Edge/                        # Native UI component system
│   ├── Components/              # EdgeComponent subclasses
│   │   └── Navigation/          # BottomNav, TopBar, SideNav, etc.
│   ├── Edge.php                 # Static component tree manager
│   └── NativeTagPrecompiler.php # <native:*> tag processing
├── Events/                      # Events dispatched from native layer
├── Facades/                     # 16 native API facades
├── Http/
│   ├── Controllers/             # Bridge controllers (events, calls)
│   └── Middleware/               # RenderEdgeComponents
├── Plugins/
│   ├── Compilers/               # AndroidPluginCompiler, IOSPluginCompiler
│   ├── Commands/                # NativePluginHookCommand base class
│   ├── PluginDiscovery.php      # Scans vendor for plugin packages
│   ├── PluginRegistry.php       # Manages registered plugins
│   └── PluginManifest.php       # Manifest parser and validator
├── Support/
│   ├── Stub.php                 # Template engine for code generation
│   └── Ios/PhpUrlGenerator.php  # iOS URL generator for php:// protocol
├── Traits/                      # 20+ build system traits
└── Pending*.php                 # PendingBuilder classes (9 total)
```

```
bootstrap/
├── ios/native.php               # iOS PHP bootstrap
└── android/native.php           # Android PHP bootstrap

config/
├── nativephp.php                # Public config (app_id, version, etc.)
└── nativephp-internal.php       # Runtime flags (platform, tempdir)

resources/
├── stubs/                       # Code generation templates
│   ├── android/                 # Kotlin stubs for plugin compilation
│   └── ios/                     # Swift stubs for plugin compilation
├── androidstudio/               # Android Studio project template
├── xcode/                       # Xcode project template
├── js/                          # Vite plugin, PHP protocol adapter
└── dist/                        # Compiled JS bridge (native.js)

tests/
├── Unit/                        # Unit tests
├── Feature/                     # Feature/integration tests
└── Fixtures/                    # Test data (manifests, configs)
```

## Development Tips

### Working with a Host App

To test your changes in a real Laravel app, add the package as a path repository in the app's `composer.json`:

```json
{
    "repositories": [
        {"type": "path", "url": "../mobile-air"}
    ]
}
```

Then require it:

```bash
composer require nativephp/mobile:@dev
```

Changes to PHP code are picked up immediately. Changes to native project templates (`resources/xcode/`,
`resources/androidstudio/`) require a fresh install:

```bash
php artisan native:install --force
```

### Debugging the Build System

The build commands write detailed logs. If a build fails, check the log output for the specific trait and step that
failed. Each trait logs its actions with `$this->logToFile()`.

### Adding Tests

Place unit tests in `tests/Unit/` and integration tests in `tests/Feature/`. Use the `Tests\TestCase` base class,
which automatically boots the package's service provider and sets up a clean config.

```php
use Tests\TestCase;

it('creates a pending photo capture', function () {
    $pending = new PendingPhotoCapture();

    expect($pending)->toBeInstanceOf(PendingPhotoCapture::class);
});
```

## Contributing Tips

- When adding new PendingBuilder methods, always return `$this` for builder methods and only call `nativephp_call()` in the terminal method
- `System::flashlight()` is deprecated in favor of `Device::toggleFlashlight()`
- `setupComposerPostUpdateScript()` in `NativeServiceProvider` is currently disabled (temporarily for testing)
- Follow existing patterns -- consistency matters more than cleverness
- Each architecture sub-page includes contributing recipes specific to that system

## Pull Request Guidelines

1. Run `composer qa` and ensure it passes
2. Add tests for new functionality
3. Follow existing patterns -- consistency is more important than cleverness
4. Keep PRs focused: one feature or fix per PR
5. Write clear commit messages describing the "why"
