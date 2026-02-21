---
title: File Modification Patterns
order: 100
---

## Overview

NativePHP uses 7 distinct patterns to modify native iOS and Android project files during installation, builds, and
plugin compilation. This page is the detailed reference for each pattern with code examples and conventions.

For a high-level overview of what gets modified and why, see the
[Core Installation](../architecture/native-project-files) architecture page.

<aside>

All file modifications must be **idempotent** -- running the same operation twice should produce the same result. Always
check before inserting, and use comment markers for clean removal.

</aside>

## Pattern 1: Direct String Replacement

The most common pattern. Use `str_replace()` or `preg_replace()` to swap configuration values in native project files.

```php
// Replace the app bundle ID in the Xcode project
$content = str_replace(
    'com.nativephp.app',
    config('nativephp.app_id'),
    $content
);

// Replace version strings in build.gradle.kts
$content = preg_replace(
    '/versionName\s*=\s*"[^"]*"/',
    'versionName = "' . config('nativephp.version') . '"',
    $content
);
```

**Used by:** `InstallsIos`, `RunsAndroid`, `PreparesBuild`, `InstallCommand`

**Target files:** `.pbxproj`, `build.gradle.kts`, `AndroidManifest.xml`, `Info.plist`, `.env`, `vite.config.js`

## Pattern 2: Stub-Based Generation

For generating new files (especially during plugin compilation), NativePHP uses the `Stub` class with
`@{{ PLACEHOLDER }}` markers:

```php
use Native\Mobile\Support\Stub;

Stub::make('android/PluginBridgeFunctionRegistration.kt.stub')
    ->replace('IMPORTS', $importStatements)
    ->replace('REGISTRATIONS', $registrationCode)
    ->saveTo($outputPath);
```

The `Stub` class loads templates from `resources/stubs/`, replaces placeholders, and writes the result. It's a fluent
builder -- chain `replace()` calls and finish with `saveTo()` or `render()`.

**Used by:** `AndroidPluginCompiler`, `IOSPluginCompiler`

**Stub locations:** `resources/stubs/android/`, `resources/stubs/ios/`

## Pattern 3: XML/Plist Regex Manipulation

Native project configuration lives in XML and Plist files. NativePHP manipulates these with regex, not DOM libraries.
This is intentional -- it preserves formatting and avoids introducing XML parser dependencies.

```php
// Extract existing permissions from AndroidManifest.xml
preg_match_all('/<uses-permission\s+android:name="([^"]+)"/', $manifest, $matches);

// Inject deep link intent filters using comment markers
$deepLinkBlock = <<<XML
<!-- NATIVEPHP-DEEPLINKS-START -->
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:scheme="{$scheme}" android:host="{$host}" />
</intent-filter>
<!-- NATIVEPHP-DEEPLINKS-END -->
XML;

$manifest = preg_replace(
    '/<!-- NATIVEPHP-DEEPLINKS-START -->.*?<!-- NATIVEPHP-DEEPLINKS-END -->/s',
    $deepLinkBlock,
    $manifest
);
```

**Used by:** `AndroidPluginCompiler`, `IOSPluginCompiler`, `RunsAndroid`

**Target files:** `AndroidManifest.xml`, `Info.plist`, `NativePHP.entitlements`

**Convention:** Use paired comment markers (`<!-- NATIVEPHP-*-START -->` / `<!-- NATIVEPHP-*-END -->`) so content can be
cleanly replaced on subsequent builds.

## Pattern 4: File/Directory Copy

Direct file copying for project setup, app icons, splash screens, and plugin source code.

```php
// Copy plugin Swift source files
File::copyDirectory(
    $plugin->path() . '/resources/ios/Sources',
    $buildPath . '/NativePHP/Sources/Plugins/' . $plugin->namespace()
);
```

For cross-platform operations, always use the `PlatformFileOperations` trait, which abstracts between `rsync` (Unix)
and `robocopy` (Windows).

**Used by:** `InstallsIos`, `InstallsAndroid`, `InstallsAppIcon`, plugin compilers

## Pattern 5: Property/Config Value Replacement

A specialized form of Pattern 1 for key-value pairs in Gradle and Kotlin configuration files. These replacements target
specific property assignments:

```php
// Update Android version code
$content = preg_replace(
    '/versionCode\s*=\s*(?:\d+|REPLACEMECODE)/',
    'versionCode = ' . $versionCode,
    $content
);

// Update min SDK version
$content = preg_replace(
    '/minSdk\s*=\s*\d+/',
    'minSdk = ' . $minSdk,
    $content
);
```

**Used by:** `RunsAndroid`, `PreparesBuild`

**Target files:** `build.gradle.kts`, `gradle.properties`, `local.properties`, `MainActivity.kt`

## Pattern 6: Build Configuration Injection

Dependencies, repositories, and configuration blocks are injected into build files. This pattern always requires an
idempotency check:

```php
// Add a Gradle dependency -- only if not already present
if (! str_contains($buildGradle, $dependency)) {
    $buildGradle = str_replace(
        'dependencies {',
        "dependencies {\n    implementation(\"$dependency\")",
        $buildGradle
    );
}
```

**Critical rule:** Always call `str_contains()` before injecting content. Without this check, repeated builds would
duplicate dependencies and corrupt the build file.

**Used by:** Plugin compilers, `RunsAndroid`, `RunsIos`

**Target files:** `build.gradle.kts`, `settings.gradle.kts`, `project.pbxproj`

## Pattern 7: Platform-Optimized File Operations

The `PlatformFileOperations` trait provides cross-platform file operations that use the most efficient tool available:

```php
// Uses rsync on macOS/Linux, robocopy on Windows
$this->platformOptimizedCopy($source, $destination, $excludedDirs);
```

The trait also provides:

- `removeDirectory()` -- Platform-optimized directory removal
- `normalizeLineEndings()` -- CRLF to LF normalization
- `replaceFileContents()` / `replaceFileContentsRegex()` -- File content replacement with line ending normalization
- `isRunningInWSL()` -- WSL detection for edge cases

This pattern is used whenever large directory trees need to be copied or synchronized, particularly during the build
preparation step when your Laravel app is bundled into the native project.

**Used by:** `PreparesBuild`, `InstallsIos`, `InstallsAndroid`

## Cross-Cutting Conventions

When working with any of these patterns, keep these rules in mind:

- **Idempotency** -- Always `str_contains()` before inserting. Running a build twice must produce identical results.
- **Comment markers** -- Use `<!-- NATIVEPHP-SECTION-START -->` / `<!-- NATIVEPHP-SECTION-END -->` pairs for blocks
  that need clean replacement or removal.
- **Line endings** -- Normalize CRLF to LF when writing files. Mixed line endings cause build failures on some
  platforms. Use the `normalizeLineEndings()` method from `PlatformFileOperations`.
- **No DOM libraries** -- XML and Plist files are manipulated with regex. This is a deliberate design choice.
- **Temp-file validation** -- When generating critical files, write to a temp path first and validate before overwriting
  the original.
