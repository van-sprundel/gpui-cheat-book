# The Three Registers

GPUI offers three different levels of abstraction, called "[registers](https://en.wikipedia.org/wiki/Register_(sociolinguistics))", that you can choose from depending on your needs. You can mix and match these approaches in the same application.

## The Three Levels

### 1. State Management (Entities)

**When to use**: Managing application state that needs to communicate between different parts of your app.

Entities are GPUI's solution for state management. They're similar to `Rc<RefCell<T>>` but owned by GPUI itself, and can only be accessed through the framework's context APIs.

```rust
struct Counter {
    count: i32,
}

// Create an entity
let counter = cx.new(|_cx| Counter { count: 0 });

// Update it later
counter.update(cx, |this, cx| {
    this.count += 1;
    cx.notify(); // Tell observers about the change
});

// Read it
let current_count = counter.read(cx).count;
```

**Key features**:
- Owned by GPUI, accessed via `Entity<T>` handles
- Observable: other parts of your app can react to changes
- Safe concurrent access through the context system

See [Entities and Ownership](./entities.md) for details.

### 2. High-Level Declarative UI (Views)

**When to use**: Building most of your user interface.

Views are entities that can be rendered by implementing the `Render` trait. This is the main way you'll build UIs in GPUI. Think of it like React components or SwiftUI views.

```rust
struct CounterView {
    counter: Entity<Counter>,
}

impl Render for CounterView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let count = self.counter.read(cx).count;

        div()
            .flex()
            .flex_col()
            .gap_2()
            .child(format!("Count: {}", count))
            .child(
                div()
                    .bg(rgb(0x2563eb))
                    .text_color(white())
                    .px_4()
                    .py_2()
                    .rounded_md()
                    .child("Increment")
                    .on_click(cx.listener(|this, _event, cx| {
                        this.counter.update(cx, |counter, cx| {
                            counter.count += 1;
                            cx.notify();
                        });
                    }))
            )
    }
}
```

**Key features**:
- Tailwind-inspired styling API (`.flex()`, `.px_4()`, etc.)
- Declarative: describe what the UI should look like
- Reactive: automatically re-renders when state changes
- Built on top of the element system

See [Views and Rendering](./views.md) for details.

### 3. Low-Level Imperative UI (Elements)

**When to use**: Maximum control and performance (custom lists, text editors, complex layouts).

Elements are the building blocks that views create. You can implement the `Element` trait directly for complete control over layout and rendering.

```rust
struct CustomElement {
    // Your state
}

impl Element for CustomElement {
    fn request_layout(
        &mut self,
        _: &WindowContext,
        style: &StyleRefinement,
        cx: &mut LayoutContext,
    ) -> LayoutId {
        // Define your custom layout using Taffy
        cx.request_layout(style, None)
    }

    fn prepaint(
        &mut self,
        _: &Bounds<Pixels>,
        _: &mut WindowContext,
        cx: &mut PaintContext,
    ) {
        // Register hitboxes and prepare for painting
    }

    fn paint(
        &mut self,
        bounds: &Bounds<Pixels>,
        _: &mut WindowContext,
        cx: &mut PaintContext,
    ) {
        // Custom drawing code
        cx.paint_quad(/* ... */);
    }
}
```

**Key features**:
- Total control over layout and rendering
- Maximum performance (e.g., `UniformList` for virtualized lists)
- Direct access to GPU rendering primitives
- More complex but more powerful

See [Custom Elements](../advanced/custom-elements/README.md) for details.

## Choosing the Right Register

| Register | Use When | Examples |
|----------|----------|----------|
| **Entities** | You need shared, observable state | App settings, document state, shared counters |
| **Views** | Building standard UI (90% of the time) | Buttons, forms, layouts, most components |
| **Elements** | You need maximum control/performance | Text editors, large lists, custom rendering |

## Mixing Registers

You'll typically use all three together:

1. **Entities** store your application state
2. **Views** render UI based on that state
3. **Elements** provide the primitives that views are built from (usually you just use the built-in ones like `div()`)

```rust
// Entity for state
struct AppState {
    items: Vec<String>,
}

// View that uses the state
struct ListView {
    state: Entity<AppState>,
}

impl Render for ListView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let items = self.state.read(cx).items.clone();

        // Using built-in elements (div, uniform_list)
        uniform_list(items, |item, _cx| {
            div().child(item)
        })
    }
}
```

## Next Steps

- Learn about [Entities and Ownership](./entities.md)
- Understand [Contexts](./contexts.md) - your interface to GPUI
- Deep dive into [Views and Rendering](./views.md)
