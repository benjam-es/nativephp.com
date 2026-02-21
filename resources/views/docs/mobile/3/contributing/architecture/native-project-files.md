---
title: Core Installation
order: 50
---

## Overview

When NativePHP installs and builds your app, it sets up and modifies the core iOS and Android project files --
Xcode projects, Gradle build scripts, AndroidManifest.xml, Info.plist, entitlements, and more. These modifications
configure the native projects with your app's identity, permissions, dependencies, and plugin integrations.

## What Gets Modified

During `native:install`, NativePHP scaffolds the native projects from templates in `resources/xcode/` (iOS) and
`resources/androidstudio/` (Android). It then rewrites key configuration values throughout the project files:

- **App identity** -- Bundle ID, app name, version, and build number are injected into `.pbxproj`,
  `build.gradle.kts`, `Info.plist`, and `AndroidManifest.xml`
- **Permissions** -- Camera, location, microphone, and other permission declarations are added to `Info.plist`
  and `AndroidManifest.xml`
- **Deep links** -- URL schemes and intent filters are configured for custom deep link handling
- **Signing** -- Code signing identity, development team, and provisioning profiles are set in the Xcode project
- **Dependencies** -- Gradle dependencies, CocoaPods, and Swift Package Manager entries are added to build files
- **Plugin integration** -- Native source files (`.kt`, `.swift`) are copied into the project, bridge functions
  are registered, and plugin manifests are merged into the native configuration

During builds (`native:run`, `native:package`), many of these values are re-applied or updated to reflect the
latest configuration.

## Key Principles

<aside>

All file modifications must be **idempotent** -- running the same operation twice should produce the same result.

</aside>

- **Check before inserting** -- Always use `str_contains()` before injecting content into build files. Without this,
  repeated builds duplicate entries and corrupt the project.
- **Comment markers** -- Use paired markers (`<!-- NATIVEPHP-SECTION-START -->` / `<!-- NATIVEPHP-SECTION-END -->`)
  for blocks that need clean replacement on subsequent builds.
- **Regex over DOM** -- XML and Plist files are manipulated with regex, not DOM libraries. This preserves formatting
  and avoids parser dependencies.
- **Line endings** -- Always normalize CRLF to LF. Mixed line endings cause build failures on some platforms.

## File Modification Patterns

The codebase uses 7 distinct patterns for modifying native project files. Each pattern has specific use cases,
conventions, and target files. These patterns are used by the build traits (`src/Traits/`), the install command,
and the plugin compilers.

For the full reference with code examples for each pattern, see the
[File Modification Patterns appendix](/docs/mobile/3/contributing/appendix/file-modification-patterns).

| Pattern | Summary |
|---------|---------|
| Direct String Replacement | `str_replace()` / `preg_replace()` for config values in project files |
| Stub-Based Generation | `Stub` class with `@{{ PLACEHOLDER }}` markers for generating new files |
| XML/Plist Regex | Regex manipulation of XML/Plist with comment markers for clean replacement |
| File/Directory Copy | Direct copying for project setup, icons, splash screens, plugin source |
| Property Value Replacement | Targeted key-value replacements in Gradle/Kotlin config files |
| Build Config Injection | Injecting dependencies and config blocks with idempotency guards |
| Platform-Optimized Operations | Cross-platform file ops via `PlatformFileOperations` trait |

## Which Traits Modify Which Files

| Trait | Files Modified |
|-------|---------------|
| `InstallsIos` | `.pbxproj`, `Info.plist`, `NativePHP.entitlements`, Xcode project structure |
| `InstallsAndroid` | `build.gradle.kts`, `AndroidManifest.xml`, Android Studio project structure |
| `RunsIos` | `.pbxproj`, `Info.plist`, signing configuration |
| `RunsAndroid` | `build.gradle.kts`, `AndroidManifest.xml`, `MainActivity.kt`, `gradle.properties` |
| `PreparesBuild` | `.env`, bundle ZIP, version files, config files |
| `AndroidPluginCompiler` | `build.gradle.kts`, `settings.gradle.kts`, `AndroidManifest.xml`, Kotlin source |
| `IOSPluginCompiler` | `.pbxproj`, `Info.plist`, Swift source, CocoaPods/SPM config |
