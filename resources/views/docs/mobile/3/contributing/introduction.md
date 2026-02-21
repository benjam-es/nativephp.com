---
title: Introduction
order: 100
---

## Welcome, Contributor!

NativePHP for Mobile is open source and we're thrilled to have you here. Whether you're fixing a bug, adding a new
native API, building an Edge component, or improving the build system, your contributions make NativePHP better for
everyone.

The `nativephp/mobile` package is a standard Laravel/Composer package with some unique characteristics: it bridges PHP
to native iOS and Android code, manages complex build toolchains, and provides a component system for truly native UI.
This section will help you understand the codebase so you can contribute with confidence.

## Plugins First

Before diving in, an important principle: **large new functionality belongs in a plugin, not in the core package.**

The plugin system is the primary mechanism for introducing and expanding native functionality. If you're thinking about
adding a significant new device capability (Bluetooth, NFC, ML, maps, etc.), it almost certainly belongs in its own
plugin package. This keeps the core lean and lets features evolve independently.

Core contributions are best suited for:
- Improvements to the plugin system itself
- Build system and tooling enhancements
- Bug fixes and performance improvements
- Edge component system improvements
- Developer experience and documentation

Check out the [Plugin docs](/docs/mobile/3/plugins/introduction) if you're unsure whether your idea is a core
contribution or a plugin.

## What Can You Contribute?

There are many ways to get involved:

- **Plugins** -- Build new plugins that add native capabilities. This is the single biggest impact you can have, and it's where large new functionality belongs. See the [Plugin docs](/docs/mobile/3/plugins/creating-plugins)
- **Plugin system improvements** -- Enhance the compiler, discovery, manifest handling, or hook system
- **Edge components** -- Build new native UI components that developers can use from Blade
- **Build system** -- Improve the traits that manage Xcode and Gradle builds
- **Bug fixes** -- Squash issues across any part of the package
- **Tests** -- Increase coverage and add regression tests
- **Documentation** -- Improve these very docs!

## Where to Start

If you're new to the codebase, we recommend reading through these pages in order:

1. **[Architecture](architecture/)** -- Understand the major systems: the native bridge, Edge, plugins, and the build pipeline. Each system has its own page with contributing recipes.
2. **[Development Workflow](development-workflow)** -- Set up your environment, run tests, and submit pull requests

## Getting Help

- **GitHub Issues** -- [nativephp/mobile-air](https://github.com/nativephp/mobile-air/issues) for bugs and feature
  requests
- **Discord** -- Join the [NativePHP community]({{ $discordLink }}) for real-time discussion
- **Existing code** -- The best reference is always the codebase itself; the patterns are consistent and well-established
