# Entities and Ownership

In GPUI, **you don't own your data** - the `App` does. Every model or view in your application is actually owned by a single top-level object called the `App`. This might feel strange coming from regular Rust, but it's the foundation of GPUI's reactive state management.

## The Key Insight

Think of `Entity<T>` like an `Rc<RefCell<T>>`, except:
- The `App` owns the actual data
- You hold a handle (`Entity<T>`) that's just an identifier
- You can only access the data when you have a reference to the `App` (via a context)

This design enables GPUI's observer pattern, where changes to one entity can automatically notify others.

## Creating Entities

You create entities using `cx.new()`:

```rust
use gpui::{App, Application, Entity};

struct Counter {
    count: i32,
}

Application::new().run(|cx: &mut App| {
    // Give ownership to the App, get back a handle
    let counter: Entity<Counter> = cx.new(|_cx| Counter { count: 0 });
});
```

The `Entity<Counter>` handle is:
- **Cheap to clone** (reference counted)
- **Type-safe** (carries `Counter` type information)
- **Inert by itself** (can't access data without a context)

## Accessing Entity Data

### Reading

Use `.read(cx)` for immutable access:

```rust
let count = counter.read(cx).count;
println!("Count: {}", count);
```

### Updating

Use `.update(cx, |entity, cx| { ... })` for mutable access:

```rust
counter.update(cx, |counter, cx| {
    counter.count += 1;
    cx.notify(); // Tell observers about the change
});
```

The callback receives:
1. `&mut Counter` - mutable reference to your data
2. `&mut Context<Counter>` - entity-specific context for services

## Notifying Observers

When you change an entity's state, call `cx.notify()` to inform observers:

```rust
counter.update(cx, |counter, cx| {
    counter.count += 1;
    cx.notify(); // Triggers observers
});
```

Without `cx.notify()`, other parts of your app won't know the state changed!

## Observing State Changes

Use `cx.observe()` to react when an entity's state changes:

```rust
struct Counter {
    count: i32,
}

Application::new().run(|cx: &mut App| {
    let first_counter = cx.new(|_cx| Counter { count: 0 });

    let second_counter = cx.new(|cx| {
        // Set up observation during creation
        cx.observe(&first_counter, |second, first, cx| {
            // This runs whenever first_counter calls cx.notify()
            second.count = first.read(cx).count * 2;
        })
        .detach(); // Keep observing until entities are dropped

        Counter { count: 0 }
    });

    // Update first counter
    first_counter.update(cx, |counter, cx| {
        counter.count += 1;
        cx.notify(); // Triggers second_counter's callback
    });

    assert_eq!(second_counter.read(cx).count, 2);
});
```

**Key points**:
- The observer gets `&mut Self` (the observer) and `Entity<Observed>` (handle to observed)
- Call `.detach()` to keep the subscription alive until the entities are dropped
- Or store the `Subscription` and drop it to cancel observation

## Event Emission (Typed Events)

For more structured communication, emit typed events:

```rust
use gpui::EventEmitter;

struct Counter {
    count: i32,
}

struct CounterChangeEvent {
    increment: i32,
}

// Enable Counter to emit this event type
impl EventEmitter<CounterChangeEvent> for Counter {}

Application::new().run(|cx: &mut App| {
    let first_counter = cx.new(|_cx| Counter { count: 0 });

    let second_counter = cx.new(|cx| {
        // Subscribe to specific event type
        cx.subscribe(&first_counter, |second, _first, event, _cx| {
            // event is CounterChangeEvent
            second.count += event.increment * 2;
        })
        .detach();

        Counter { count: 0 }
    });

    first_counter.update(cx, |counter, cx| {
        counter.count += 2;
        cx.emit(CounterChangeEvent { increment: 2 }); // Emit typed event
    });

    assert_eq!(second_counter.read(cx).count, 4);
});
```

## observe vs subscribe

| Method | When to Use | What You Get |
|--------|-------------|--------------|
| `cx.observe()` | React to any state change | Just the entity handle |
| `cx.subscribe()` | React to specific events | The typed event data |

**Rule of thumb**: Use `observe` for "something changed", use `subscribe` for "here's what changed".

## Common Patterns

### Pattern 1: Shared State

Multiple views sharing a single state entity:

```rust
struct AppState {
    user: String,
}

struct HeaderView {
    state: Entity<AppState>,
}

struct FooterView {
    state: Entity<AppState>,
}

// Both views hold the same Entity<AppState> handle
let state = cx.new(|_| AppState { user: "Alice".into() });
let header = cx.new(|_| HeaderView { state: state.clone() });
let footer = cx.new(|_| FooterView { state: state.clone() });
```

### Pattern 2: Parent-Child Communication

Child notifies parent of changes:

```rust
struct ChildView {
    value: String,
}

impl EventEmitter<ChildChangedEvent> for ChildView {}

struct ParentView {
    child: Entity<ChildView>,
}

impl ParentView {
    fn new(cx: &mut Context<Self>) -> Self {
        let child = cx.new(|_| ChildView { value: String::new() });

        cx.subscribe(&child, |parent, child, event, cx| {
            // React to child events
            cx.notify(); // Propagate to parent's observers
        }).detach();

        Self { child }
    }
}
```

### Pattern 3: Weak Handles (Avoiding Cycles)

Use `WeakEntity` to avoid reference cycles:

```rust
struct Parent {
    child: Entity<Child>,
}

struct Child {
    parent: WeakEntity<Parent>, // Weak reference breaks the cycle
}

let parent = cx.new(|cx| {
    let parent_weak = cx.weak_entity(); // Get weak reference to self

    let child = cx.new(|_| Child {
        parent: parent_weak,
    });

    Parent { child }
});
```

## Pitfalls

### Forgetting to Notify

```rust
// ❌ Bad: Changes but doesn't notify
counter.update(cx, |counter, _cx| {
    counter.count += 1;
    // Observers won't be notified!
});

// ✅ Good: Always notify after changes
counter.update(cx, |counter, cx| {
    counter.count += 1;
    cx.notify();
});
```

### Detaching Subscriptions

```rust
// ❌ Bad: Subscription dropped immediately
cx.observe(&other, |this, other, cx| {
    // This will never run!
});

// ✅ Good: Detach to keep it alive
cx.observe(&other, |this, other, cx| {
    // This will run whenever `other` notifies
}).detach();

// ✅ Also good: Store it to control lifetime
let subscription = cx.observe(&other, |this, other, cx| {
    // Runs until subscription is dropped
});
self.subscription = Some(subscription);
```

### Reference Cycles

```rust
// ❌ Bad: Creates a reference cycle
struct Node {
    next: Option<Entity<Node>>, // Strong reference
    prev: Option<Entity<Node>>, // Strong reference = cycle!
}

// ✅ Good: Use weak references to break cycles
struct Node {
    next: Option<Entity<Node>>,
    prev: Option<WeakEntity<Node>>, // Weak reference
}
```

## Next Steps

- Learn about [Contexts](./contexts.md) - the different context types and their APIs
- See [State Management Patterns](../patterns/state/README.md) for real-world examples
- Read the [official tutorial](https://github.com/zed-industries/zed/blob/main/crates/gpui/src/_ownership_and_data_flow.rs)
