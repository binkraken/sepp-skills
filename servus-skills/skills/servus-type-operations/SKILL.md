---
name: servus-type-operations
description: Type conversion and conditional invocation with Servus.Core — TryConvert, Convert, InvokeIf, WhenType pattern-matching helpers
invocable: true
---

# Type Operations with Servus.Core

Use when you need safe type conversion, conditional interface invocation, or lightweight pattern matching on object types.

## When to Use

- Safely converting or casting objects with fallback behavior
- Invoking an action only if an object implements a specific interface
- Replacing `is`/`as` checks followed by casts with a fluent API
- Dispatching logic based on runtime type without switch expressions

## API Reference

### TypeCheckExtensions

**Namespace:** `Servus.Core.Reflection`

```csharp
public static class TypeCheckExtensions
{
    // Safe conversion — returns false if not assignable
    bool TryConvert<TTarget>(this object? item, out TTarget? value);

    // Direct cast — throws if not assignable
    TTarget Convert<TTarget>(this object? item);

    // Convert with custom converter function
    TTarget? Convert<TTarget>(this object? item, Func<object, TTarget> converter);

    // Convert with typed input converter
    TTarget? Convert<TInput, TTarget>(this object? item, Func<TInput, TTarget> converter);
}
```

### InterfaceInvokeExtensions

**Namespace:** `Servus.Core.Reflection`

```csharp
public static class InterfaceInvokeExtensions
{
    // Execute action if object implements TTarget
    void InvokeIf<TTarget>(this object entity, Action<TTarget> action);

    // Async variant
    Task InvokeIfAsync<TTarget>(this object entity, Func<TTarget, Task> action);

    // Execute function with return value (returns default if not TTarget)
    TResult? InvokeIf<TTarget, TResult>(this object entity, Func<TTarget, TResult> action);

    // Async variant with return value
    Task<TResult?> InvokeIfAsync<TTarget, TResult>(this object entity, Func<TTarget, Task<TResult>> action);
}
```

### WhenType Extensions

**Namespace:** `Servus.Core.Functional`

```csharp
public static class TypeExtensions
{
    // Execute handler if target is of type T
    void WhenType<T>(this object target, Action<T> handler);
}
```

## Usage Pattern

### Safe type conversion

```csharp
object message = GetMessage();

// TryConvert — like TryParse for types
if (message.TryConvert<OrderCommand>(out var command))
{
    ProcessOrder(command);
}

// Direct convert — when you're confident about the type
var order = message.Convert<OrderCommand>();

// Custom converter
var dto = message.Convert<OrderDto>(obj => MapToDto((OrderCommand)obj));
```

### Conditional interface invocation

```csharp
// Execute only if the object implements the interface
object handler = GetHandler();

handler.InvokeIf<IDisposable>(d => d.Dispose());

handler.InvokeIf<IInitializable>(init => init.Initialize());

// With return value
int? count = handler.InvokeIf<ICountable, int>(c => c.GetCount());

// Async
await handler.InvokeIfAsync<IAsyncInitializable>(
    async init => await init.InitializeAsync());
```

### WhenType for lightweight dispatch

```csharp
object evt = GetEvent();

evt.WhenType<OrderCreated>(e => HandleOrderCreated(e));
evt.WhenType<OrderCancelled>(e => HandleOrderCancelled(e));
evt.WhenType<PaymentReceived>(e => HandlePayment(e));
```

### Processing setup containers

```csharp
// Real-world pattern from Servus.Core's AppBuilder
foreach (var container in setupContainers)
{
    container.InvokeIf<IServiceSetupContainer>(
        c => c.SetupServices(services, configuration));

    container.InvokeIf<ILoggingSetupContainer>(
        c => c.SetupLogging(loggingBuilder));

    await container.InvokeIfAsync<IAsyncSetupContainer>(
        async c => await c.SetupAsync());
}
```

## Common Mistakes

- **Using `Convert<T>` on null**: Returns `default(T)`. Use `TryConvert` if null means "not found" vs. "is null".
- **Expecting `WhenType` to short-circuit**: `WhenType` executes if matched but does not prevent subsequent `WhenType` calls. It's not a switch — all matching types execute.
- **Confusing `InvokeIf` with `WhenType`**: `InvokeIf` is for interface checks (`IDisposable`, `IInitializable`). `WhenType` is for concrete type dispatch. Both work, but `InvokeIf` communicates intent better for interface contracts.
- **Ignoring the return value of `InvokeIf<T, TResult>`**: Returns `default(TResult)` if the type doesn't match — check for null on reference types.
