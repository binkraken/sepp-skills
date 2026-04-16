---
name: servus-handler-registry
description: Type-based message dispatch with HandlerRegistry from Servus.Core — Register, Handle, HandleAll, Stash/Pop for handler state management
invocable: true
---

# HandlerRegistry with Servus.Core

Use when building type-based message dispatch logic outside of Akka.NET actors, or when you need a lightweight handler pipeline with conditional matching and stash/pop state management.

## When to Use

- Dispatching messages or events to typed handlers without a full actor system
- Building a command/event handler pipeline with predicate-based routing
- Needing handler state snapshots (stash) and rollback (pop)
- Replacing long `switch`/`if-else` chains on message types

## API Reference

**Namespace:** `Servus.Core.Collections`

```csharp
public class HandlerRegistry
{
    // Register a handler for type T (always matches)
    void Register<T>(Action<T> handler)

    // Register a handler with a predicate guard
    void Register<T>(Predicate<T> canHandle, Action<T> handler)

    // Register by runtime Type with predicate
    void Register(Type type, Predicate<object> canHandle, Action<object> handler)

    // Execute the FIRST matching handler — returns true if handled
    bool Handle(object item)

    // Execute ALL matching handlers — returns count of handlers invoked
    int HandleAll(object item)

    // Check if any handler matches without executing
    bool CanHandle(object item)

    // Get all matching handlers for an item
    IEnumerable<Action<object>> GetMatchingHandlers(object item)

    // Save current handler state (push onto stack)
    void Stash()

    // Restore previous handler state (pop from stack) — returns false if stack is empty
    bool Pop()

    int Count { get; }
    void Clear()
}
```

Handler matching uses `IsAssignableTo` — registering a handler for a base type or interface matches all derived types.

## Usage Pattern

### Basic typed dispatch

```csharp
var registry = new HandlerRegistry();

registry.Register<OrderCreated>(evt =>
    Console.WriteLine($"Order {evt.OrderId} created"));

registry.Register<OrderCancelled>(evt =>
    Console.WriteLine($"Order {evt.OrderId} cancelled"));

// Dispatches to the correct handler based on runtime type
registry.Handle(new OrderCreated("order-1"));
```

### With predicate guards

```csharp
registry.Register<OrderCreated>(
    canHandle: evt => evt.Priority == Priority.High,
    handler: evt => NotifyUrgent(evt));

registry.Register<OrderCreated>(
    canHandle: evt => evt.Priority == Priority.Low,
    handler: evt => EnqueueLater(evt));
```

### HandleAll for fan-out

```csharp
registry.Register<OrderCreated>(evt => SaveToDatabase(evt));
registry.Register<OrderCreated>(evt => SendNotification(evt));
registry.Register<OrderCreated>(evt => UpdateDashboard(evt));

// Invokes all three handlers
int count = registry.HandleAll(new OrderCreated("order-1"));
```

### Stash/Pop for state management

```csharp
// Initial handlers
registry.Register<ProcessCommand>(cmd => HandleNormally(cmd));

// Save current state and switch to maintenance mode
registry.Stash();
registry.Clear();
registry.Register<ProcessCommand>(cmd => RejectDuringMaintenance(cmd));

// Restore original handlers when maintenance ends
registry.Pop();
```

## Common Mistakes

- **Expecting handler order guarantees with `Handle`**: `Handle` returns after the first match. If multiple handlers could match, use `HandleAll` for fan-out or register more specific predicates.
- **Forgetting `IsAssignableTo` semantics**: A handler registered for `IEvent` matches all classes implementing `IEvent`. Register specific types first if you need precedence.
- **Calling `Pop` without `Stash`**: `Pop` returns `false` if the stash stack is empty — check the return value.
- **Using HandlerRegistry for actor messages**: Inside Akka.NET actors, use the built-in `Receive<T>` handlers. HandlerRegistry is for non-actor dispatch scenarios.
