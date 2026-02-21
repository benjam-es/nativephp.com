---
title: Edge System
order: 200
---

## Overview

Edge (Element Definition and Generation Engine) turns Blade syntax into native UI components. It does **not** render
HTML -- it produces a JSON component tree that the native layer renders as platform-native elements.

See the user-facing [Edge docs](/docs/mobile/3/edge-components/introduction) for usage examples.

## Component Lifecycle

Every Edge component extends `EdgeComponent`, which itself extends Laravel's `Component` base class. Each component
defines:

- **`$type`** (string) -- An identifier like `'bottom-nav'` or `'top-bar'` that the native layer uses to select the
  right native widget
- **`$hasChildren`** (bool) -- Whether the component can contain child components (default: `false`)
- **`toNativeProps(): array`** (abstract) -- Returns the component's configuration as an associative array
- **`requiredProps(): array`** (optional) -- Defines which props must have non-empty values; validated at render time

The `render()` method is where the magic happens. It validates required props, then either adds a leaf component to the
tree or starts a context for collecting children. In both cases, it returns an empty placeholder view -- Edge components
produce native UI, not HTML.

## Tree Building

The `Edge` class manages a static component tree using a context stack:

- **Leaf components** -- `Edge::add($type, $props)` appends a component to the current context (or to the root if no
  context is active)
- **Container components** -- `Edge::startContext()` pushes a new nesting level and returns a context index;
  `Edge::endContext($index, $type, $props)` pops it and finalizes the parent with its collected children

The `render()` method in `EdgeComponent` handles this automatically. For a component with `$hasChildren = true`, the
Blade template collects the slot content (which triggers child component rendering), then calls `Edge::endContext()` to
close the parent.

Components at the top level of a page end up as root entries in the `$components` array. Nested components end up in
their parent's `children` array.

## Sending to Native

After the response is generated, `RenderEdgeComponents` middleware calls `Edge::set()`. This method:

1. Serializes the component tree to JSON
2. Calls `nativephp_call('Edge.Set', json_encode(['components' => $components]))`
3. Resets the tree for the next request

If no components were registered during the request, `Edge::set()` returns early without making a native call.

## Auto-Discovery

Components in `src/Edge/Components/` are auto-discovered by `NativeServiceProvider::registerNativeComponents()`. The
method recursively scans the directory, converts class names to kebab-case, and registers them as Blade components with
a `native-` prefix:

- `BottomNav` becomes `<native:bottom-nav>` (or `<x-native-bottom-nav>`)
- `TopBarAction` becomes `<native:top-bar-action>`
- `SideNavGroup` becomes `<native:side-nav-group>`

The `NativeTagPrecompiler` Blade precompiler enables the `<native:*>` shorthand syntax by transforming it into
`<x-native-*>` before Blade compiles the template. No manual registration is needed -- just create a class in the right
directory and it's available.

## Contributing: Adding an Edge Component

### Step 1: Create the Component Class

Add a class in `src/Edge/Components/` (use subdirectories for organization). Extend `EdgeComponent`:

```php
// src/Edge/Components/Navigation/FloatingMenu.php
namespace Native\Mobile\Edge\Components\Navigation;

use Native\Mobile\Edge\Components\EdgeComponent;

class FloatingMenu extends EdgeComponent
{
    protected string $type = 'floating-menu';
    protected bool $hasChildren = true;

    public function __construct(
        public string $position = 'bottom-right',
        public ?string $icon = null,
    ) {}

    protected function toNativeProps(): array
    {
        return [
            'position' => $this->position,
            'icon' => $this->icon,
        ];
    }

    protected function requiredProps(): array
    {
        return ['icon'];
    }
}
```

### Step 2: Understand Registration

You don't need to register the component manually. `registerNativeComponents()` auto-discovers all classes in
`src/Edge/Components/` and registers them as Blade components. The naming convention converts the class name to
kebab-case with a `native-` prefix:

- `FloatingMenu` becomes `<native:floating-menu>`
- `BottomNavItem` becomes `<native:bottom-nav-item>`

### Step 3: Set Key Properties

- **`$type`** -- The string identifier sent to the native layer. Must match what the native code expects.
- **`$hasChildren`** -- Set to `true` if the component wraps child components. This activates the context stack for
  collecting nested components.
- **`toNativeProps()`** -- Return an associative array of all properties the native layer needs. Only include values
  that are set; avoid sending `null` values unnecessarily.
- **`requiredProps()`** -- Return an array of property names that must have non-empty values. Validation runs at render
  time and throws a `MissingRequiredPropsException` with a clear error message listing the missing props.

### Step 4: Test It

Edge components produce JSON, not HTML, so test them by verifying the component tree output:

```php
use Native\Mobile\Edge\Edge;

it('adds a floating menu to the component tree', function () {
    Edge::reset();

    $component = new FloatingMenu(position: 'top-left', icon: 'plus');
    $component->render();

    $tree = Edge::all();
    expect($tree)->toHaveCount(1);
    expect($tree[0]['type'])->toBe('floating-menu');
    expect($tree[0]['data']['position'])->toBe('top-left');
});
```
