---
name: servus-diagnostics
description: Distributed tracing infrastructure with Servus.Core — IWithTracing interface, ActivitySourceRegistry, and ActivitySource attributes
invocable: true
---

# Diagnostics & Tracing with Servus.Core

Use when adding distributed tracing support to messages, commands, or events outside of Akka.NET actors. Provides the core `IWithTracing` interface and `ActivitySourceRegistry` for managing `System.Diagnostics.ActivitySource` instances.

## When to Use

- Adding TraceId/SpanId propagation to messages or DTOs
- Managing ActivitySource instances per type with automatic naming
- Starting Activities from trace context on incoming messages
- Building tracing infrastructure that integrates with OTLP, Jaeger, or Zipkin

## API Reference

**Namespace:** `Servus.Core.Diagnostics`

### IWithTracing

```csharp
public interface IWithTracing
{
    string? TraceId { get; set; }
    string? SpanId { get; set; }

    // Build an ActivityContext from TraceId/SpanId (generates random if empty)
    ActivityContext GetContext();

    // Copy TraceId/SpanId from Activity.Current
    void AddTracing();

    // Copy TraceId/SpanId from another IWithTracing
    void AddTracing(IWithTracing tracing);

    // Set TraceId/SpanId directly (generates random if null/empty)
    void AddTracing(string? traceId, string? spanId);

    // Start an Activity linked to this trace context
    Activity? StartActivity(string name, ActivitySource source, ActivityKind kind = ActivityKind.Consumer);
}
```

All methods have default interface implementations — just implement the two properties.

### ActivitySourceRegistry

```csharp
public static class ActivitySourceRegistry
{
    // Register an ActivitySource for a type (name auto-discovered or snake_cased)
    static void Add<T>(string? name = null);
    static void Add(Type key, string? name = null);

    // Start an Activity using the registered source for type T
    static Activity? StartActivity<T>(string activityName, IWithTracing trace,
        ActivityKind kind = ActivityKind.Consumer);
    static Activity? StartActivity(Type key, string activityName, IWithTracing trace,
        ActivityKind kind = ActivityKind.Consumer);
}
```

### Attributes

```csharp
// Override the ActivitySource name for a type
[ActivitySourceName("custom-source-name")]
public class MyActor { }

// Redirect ActivitySource lookup to a different type
[ActivitySourceKey(typeof(SharedSource))]
public class MyHandler { }
```

Name resolution order: explicit `name` parameter > `[ActivitySourceName]` attribute > `[ActivitySourceKey]` redirect > `TypeName.ToSnakeCase()`.

## Usage Pattern

### Implement IWithTracing on a message

```csharp
public record ProcessOrder(Guid OrderId) : IWithTracing
{
    public string? TraceId { get; set; }
    public string? SpanId { get; set; }
}
```

### Propagate trace context

```csharp
// From an HTTP controller — capture current Activity
var command = new ProcessOrder(orderId);
command.AddTracing(); // copies Activity.Current TraceId/SpanId

// From another traced message
var evt = new OrderProcessed(orderId);
evt.AddTracing(command); // copies TraceId/SpanId from the command
```

### Register and use ActivitySources

```csharp
// At startup — register sources
ActivitySourceRegistry.Add<OrderService>();
ActivitySourceRegistry.Add<PaymentService>("payment-service");

// In a handler — start an Activity from trace context
public void Handle(ProcessOrder msg)
{
    using var activity = ActivitySourceRegistry.StartActivity<OrderService>(
        "process-order", msg, ActivityKind.Consumer);

    // activity is linked to the incoming trace
    ProcessOrderInternal(msg);
}
```

### Use attributes for naming

```csharp
[ActivitySourceName("order-processing")]
public class OrderService
{
    public void Handle(ProcessOrder msg)
    {
        // Source name is "order-processing" (from attribute)
        using var activity = ActivitySourceRegistry.StartActivity<OrderService>(
            "handle-order", msg);
    }
}

// Share a source across multiple types
[ActivitySourceKey(typeof(OrderService))]
public class OrderValidator
{
    public void Validate(ProcessOrder msg)
    {
        // Uses OrderService's ActivitySource
        using var activity = ActivitySourceRegistry.StartActivity<OrderValidator>(
            "validate-order", msg);
    }
}
```

## Common Mistakes

- **Forgetting to implement `TraceId`/`SpanId` as get/set properties**: The default interface methods need writable properties. Auto-properties in a record work fine.
- **Calling `AddTracing()` when no `Activity.Current` exists**: If there's no ambient Activity, `AddTracing()` generates random IDs. This is safe but may not be what you want — pass explicit IDs if you have them.
- **Not disposing the Activity**: `StartActivity` returns a nullable `Activity`. Always use `using` to ensure it's disposed when the operation completes.
- **Confusing Servus.Core `IWithTracing` with Servus.Akka tracing**: Servus.Core provides the interface and registry. Servus.Akka's `TracedMessageActor` builds on top of it for actor-specific tracing.
