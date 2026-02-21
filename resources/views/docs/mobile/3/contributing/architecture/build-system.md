---
title: Build System
order: 400
---

## Overview

The build system lives in `src/Traits/` -- over 20 composable traits, each responsible for a specific aspect of the build
pipeline. Artisan commands compose these traits to assemble the full workflow rather than implementing build logic
directly. This keeps commands thin and lets you reuse build steps across different commands.

## Key Traits

| Trait | Responsibility |
|-------|---------------|
| `PreparesBuild` | Creates the Laravel bundle ZIP, updates configs, cleans env files |
| `RunsAndroid` | Executes Gradle builds, configures deep links, sets build metadata |
| `RunsIos` | Executes xcodebuild, manages signing, configures capabilities |
| `InstallsAndroid` | Scaffolds the Android Studio project, downloads PHP binary |
| `InstallsIos` | Scaffolds the Xcode project, downloads PHP binary |
| `PlatformFileOperations` | Cross-platform rsync/robocopy abstraction |
| `CleansEnvFile` | Strips secrets from the `.env` before bundling |
| `InstallsAppIcon` | Processes and installs app icons for both platforms |
| `InstallsSplashScreen` | Generates platform-specific splash screens |
| `ManagesViteDevServer` | Starts/stops Vite for hot reload during development |
| `ManagesIosSigning` | Handles code signing identity and provisioning profiles |

Traits frequently compose each other. For example, `PreparesBuild` uses `CleansEnvFile`, `InstallsAppIcon`,
`InstallsAndroidSplashScreen`, and `PlatformFileOperations` internally.

## Build Flow

A typical `native:run` invocation follows these steps:

1. **Validate the build environment** -- Checks that `app_id` and `version` are set in config
2. **Install/update the native project** -- Runs `InstallsAndroid` or `InstallsIos` to scaffold the Xcode/Android Studio
   project if needed, downloading the PHP binary for the target platform
3. **Compile registered plugins** -- `AndroidPluginCompiler` or `IOSPluginCompiler` processes each registered plugin,
   copying native source, generating bridge registration, and merging manifests
4. **Prepare the build** -- `PreparesBuild` bundles your Laravel app into a ZIP archive, runs `composer install --no-dev`,
   optimizes the autoloader, cleans the `.env`, and copies platform bootstrap files
5. **Execute the platform build** -- `RunsAndroid` runs Gradle or `RunsIos` runs xcodebuild with the appropriate
   configuration
6. **Launch on device or emulator** -- The built app is deployed and launched

### Bundle Preparation Details

The `prepareLaravelBundle()` method in `PreparesBuild` is the most complex step. It:

- Copies your entire Laravel app to a temp directory (excluding `.git`, `node_modules`, `nativephp/`)
- Runs `composer install --no-dev` in the temp directory
- Runs `composer dump-autoload --optimize --classmap-authoritative`
- Writes a `.version` file
- Copies and cleans the `.env` file (stripping keys matching `cleanup_env_keys` patterns)
- Creates a ZIP archive and places it in the native project's assets directory

## Contributing: Adding a New Command

Commands live in `src/Commands/`. Use traits for any build-related logic rather than implementing it inline:

```php
namespace Native\Mobile\Commands;

use Illuminate\Console\Command;
use Native\Mobile\Traits\PlatformFileOperations;

class MyCommand extends Command
{
    use PlatformFileOperations;

    protected $signature = 'native:my-command {platform?}';
    protected $description = 'Does something useful';

    public function handle(): int
    {
        // Your logic here
        return self::SUCCESS;
    }
}
```

Then register it in `NativeServiceProvider::configurePackage()`:

```php
->hasCommands([
    // ... existing commands
    MyCommand::class,
])
```

### Tips

- All `native:*` commands accept an optional `{platform?}` argument (`ios`/`i`, `android`/`a`)
- Use `$this->logToFile()` for detailed logging that helps debug build failures
- Use `$this->components->task()` for user-visible progress output
- Compose existing traits rather than duplicating their logic
- If your command modifies native project files, follow the patterns in
  [Core Installation](native-project-files) page and the [File Modification Patterns](/docs/mobile/3/contributing/appendix/file-modification-patterns) appendix
