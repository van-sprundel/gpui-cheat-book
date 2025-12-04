# Contexts (App, Visual, Context\<T\>)

Contexts are your main interface to GPUI. Every interaction with the framework - creating entities, opening windows, rendering UI - goes through a context. GPUI provides three context types, each with progressively more capabilities.

## The Three Context Types

```
App (fewest capabilities)
 ├── VisualContext (adds window/UI capabilities)
     └── Context<T> (adds entity-specific capabilities)
```

### App - Application Context

The base context for all application-level services. Available everywhere.

**Capabilities**:
- Create and manage entities
- Access global state
- Spawn async tasks
- Manage platform services

**Where you'll see it**:
```rust,ignore
Application::new().run(|cx: &mut App| {
    // Application startup
});
```

### VisualContext - Window-Aware Context

Extends `App` with window and UI capabilities. Available when you're working with windows or views.

**Additional capabilities beyond App**:
- Open windows
- Focus management
- Access to window-specific services

**Where you'll see it**:
```rust,ignore
cx.open_window(WindowOptions::default(), |_, cx| {
    // cx is a VisualContext here
    cx.new(|_| MyView::default())
})
```

### Context\<T\> - Entity-Specific Context

The most powerful context, tied to a specific entity. Extends `VisualContext` with entity-level services.

**Additional capabilities beyond VisualContext**:
- `cx.notify()` - notify observers of this entity
- `cx.emit()` - emit typed events from this entity
- `cx.observe()` - observe other entities
- `cx.subscribe()` - subscribe to events from other entities
- `cx.entity()` / `cx.weak_entity()` - get handle to self

**Where you'll see it**:
```rust,ignore
let counter = cx.new(|cx: &mut Context<Counter>| {
    // cx knows it's for a Counter entity
    Counter { count: 0 }
});

counter.update(cx, |counter, cx: &mut Context<Counter>| {
    // cx is tied to the counter entity
    counter.count += 1;
    cx.notify();
});
```

## Context Type Hierarchy

All context types implement `AppContext`, but not all have access to advanced features:

| Feature | App | VisualContext | Context\<T\> |
|---------|-----|---------------|--------------|
| Create entities (`cx.new()`) | ✅ | ✅ | ✅ |
| Global state | ✅ | ✅ | ✅ |
| Spawn tasks | ✅ | ✅ | ✅ |
| Open windows | ❌ | ✅ | ✅ |
| Focus management | ❌ | ✅ | ✅ |
| `cx.notify()` | ❌ | ❌ | ✅ |
| `cx.emit()` | ❌ | ❌ | ✅ |
| `cx.observe()` | ❌ | ❌ | ✅ |
| `cx.subscribe()` | ❌ | ❌ | ✅ |

## Common Context APIs

### Creating Entities

Available on all contexts:

```rust,ignore
let counter = cx.new(|cx| Counter { count: 0 });
```

### Accessing Entities

```rust,ignore
// Read immutably
let value = counter.read(cx).count;

// Update mutably
counter.update(cx, |counter, cx| {
    counter.count += 1;
    cx.notify();
});
```

### Global State

```rust,ignore
// Define a global
#[derive(Clone, Default)]
struct Theme {
    background: Rgb,
}

impl Global for Theme {}

// Set global (once)
cx.set_global(Theme::default());

// Read global
let theme = Theme::global(cx);

// Update global
cx.update_global::<Theme, _>(|theme, cx| {
    theme.background = rgb(0x1e1e1e);
});
```

### Spawning Async Tasks

```rust,ignore
// Spawn on UI thread
cx.spawn(|cx| async move {
    // Async work that can access cx
    load_data().await
})

// Spawn on background thread
cx.background_spawn(async move {
    // Heavy computation, no cx access
    compute_something()
})
```

### Entity-Level APIs (Context\<T\> only)

```rust,ignore
// Notify observers
cx.notify();

// Emit typed events
cx.emit(MyEvent { data: 42 });

// Observe another entity
cx.observe(&other_entity, |this, other, cx| {
    // React to changes
}).detach();

// Subscribe to events
cx.subscribe(&other_entity, |this, other, event, cx| {
    // Handle event
}).detach();

// Get handle to self
let self_handle = cx.entity();
let weak_self = cx.weak_entity();
```

## When to Use Which Context

### Use App when:
- Application startup
- Managing top-level state
- Creating initial entities

```rust,ignore
Application::new().run(|cx: &mut App| {
    let app_state = cx.new(|_| AppState::default());
    // Start your app
});
```

### Use VisualContext when:
- Opening windows
- Working with window-level UI
- No specific entity in focus

```rust,ignore
cx.open_window(WindowOptions::default(), |window, cx| {
    // cx is VisualContext
    cx.new(|_| RootView::default())
});
```

### Use Context\<T\> when:
- Inside entity methods
- Reacting to state changes
- Emitting or observing events

```rust,ignore
impl MyView {
    fn handle_click(&mut self, cx: &mut Context<Self>) {
        self.count += 1;
        cx.notify(); // Only available on Context<T>
    }
}
```

## Window and WindowContext

There's also `Window` and `WindowContext` types used during rendering:

```rust,ignore
impl Render for MyView {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // window: access to window-specific state
        // cx: full Context<MyView> capabilities
        div().child("Hello")
    }
}
```

## Common Patterns

### Pattern 1: Passing Context Down

You can't store contexts, but you can pass them through function calls:

```rust,ignore
impl MyView {
    fn build_ui(&mut self, cx: &mut Context<Self>) -> impl IntoElement {
        self.build_header(cx) // Pass cx to helpers
    }

    fn build_header(&mut self, cx: &mut Context<Self>) -> impl IntoElement {
        div().child("Header")
    }
}
```

### Pattern 2: Getting Self Handle

```rust,ignore
let my_view = cx.new(|cx: &mut Context<MyView>| {
    let self_handle = cx.entity(); // Get handle to entity being created

    // Store it for later or pass to others
    MyView {
        self_ref: self_handle,
    }
});
```

### Pattern 3: Upgrading Context Types

You often receive a less capable context but need a more capable one:

```rust,ignore
// This won't work - can't upgrade context types directly
// let visual_cx: &mut VisualContext = cx; // ❌

// Instead, you already have the right context type based on where you are:
// - In entity callbacks: Context<T>
// - In window callbacks: VisualContext
// - In app startup: App
```

## Pitfalls

### Trying to Store Contexts

```rust,ignore
// ❌ Bad: Can't store contexts
struct MyView {
    cx: Context<MyView>, // Won't compile
}

// ✅ Good: Contexts are always passed as parameters
impl MyView {
    fn do_something(&mut self, cx: &mut Context<Self>) {
        // Use cx here, don't store it
    }
}
```

### Wrong Context Type

```rust,ignore
// ❌ Bad: Can't notify from App context
fn update_state(cx: &mut App) {
    cx.notify(); // No such method!
}

// ✅ Good: Use Context<T> for entity operations
fn update_state(cx: &mut Context<MyEntity>) {
    cx.notify(); // Works!
}
```

### Forgetting WeakEntity for Self-References

```rust,ignore
// ❌ Bad: Strong self-reference creates cycle
let entity = cx.new(|cx| {
    let self_handle = cx.entity();
    MyStruct {
        self_ref: self_handle, // Entity<MyStruct> - cycle!
    }
});

// ✅ Good: Use weak for self-references
let entity = cx.new(|cx| {
    let self_handle = cx.weak_entity();
    MyStruct {
        self_ref: self_handle, // WeakEntity<MyStruct> - no cycle
    }
});
```

## Next Steps

- See [Views and Rendering](./views.md) to learn how contexts are used in UI
- Check [State Management](../patterns/state/README.md) for practical context usage patterns
- Explore [Async Patterns](../advanced/async/patterns.md) for spawning tasks
