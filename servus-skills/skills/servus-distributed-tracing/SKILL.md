---
name: servus-distributed-tracing
description: Distributed tracing for Akka.NET actors with Servus.Akka — TracedMessageActor base class, TellTraced, AskTraced, ForwardTraced
invocable: true
---

# Distributed Tracing with Servus.Akka

Use when building actors that participate in distributed tracing via `System.Diagnostics.Activity`. Provides automatic trace context propagation through the actor system.

## When to Use

- Building actors that should appear in distributed traces (Jaeger, Zipkin, OTLP)
- Propagating TraceId/SpanId across actor messages
- Replacing manual `ActivitySource` boilerplate in actors
- Sending traced messages between actors

## Architecture

`TracedMessageActor` seals `OnReceive` and wraps every message in an `Activity` if the message implements `IWithTracing`. Messages that don't implement `IWithTracing` are automatically wrapped in a `TracedMessageEnvelope` by the `TellTraced`/`AskTraced`/`ForwardTraced` extensions. The envelope is unwrapped before the message reaches your handler.

```
Sender                       TracedMessageActor
  |                                |
  |-- TellTraced(msg) ----------->|
  |   (wraps in envelope,         |
  |    sets TraceId/SpanId)       |
  |                          OnReceive (sealed)
  |                            |-- starts Activity from trace context
  |                            |-- unwraps IMessageEnvelope
  |                            |-- dispatches to Receive<T> handler
  |                            |-- Activity disposed
```

## API Reference

### TracedMessageActor (base class)

```csharp
public class TracedMessageActor : UntypedActor
{
    // Register typed message handlers (same API as ReceiveActor)
    protected void Receive<T>(Action<T> handler, Predicate<T>? shouldHandle = null)
    protected void ReceiveAsync<T>(Func<T, Task> handler, Predicate<T>? shouldHandle = null)
    protected void ReceiveAny(Action<object> handler)
    protected void ReceiveAnyAsync(Func<object, Task> handler)

    // Behavior switching (handlers are stacked/replaced properly)
    protected void Become(Action configure)
    protected void BecomeStacked(Action configure)
    protected void UnbecomeStacked()

    // Reply helpers
    protected void ReplyTraced(object message)  // sends traced reply to Sender
    protected void Reply(object message)        // sends plain reply to Sender
}
```

### Traced Message Extensions

```csharp
// Tell with trace context — wraps in TracedMessageEnvelope if not IWithTracing
void TellTraced(this ICanTell recipient, object message, IActorRef? sender = null)

// Ask with trace context — returns typed response
Task<T> AskTraced<T>(this IActorRef recipient, object message)

// Forward with trace context — preserves original sender
void ForwardTraced(this IActorRef recipient, object message)
```

### Message Types

```csharp
// Implement on your messages for native tracing (no envelope needed)
public interface IWithTracing
{
    string? TraceId { get; set; }
    string? SpanId { get; set; }
}

// Auto-wrapper for plain messages
public sealed record TracedMessageEnvelope(object Message) : IWithTracing, IMessageEnvelope

// Envelope interface — TracedMessageActor unwraps these before dispatching
public interface IMessageEnvelope { object Message { get; } }
```

## Usage Pattern

### Define a traced actor

```csharp
public class OrderActor : TracedMessageActor
{
    private readonly IOrderRepository _repo;

    public OrderActor(IOrderRepository repo)
    {
        _repo = repo;

        Receive<ProcessOrder>(Handle);
        Receive<GetOrderStatus>(HandleStatus);
        ReceiveAsync<CancelOrder>(HandleCancel);
    }

    private void Handle(ProcessOrder msg)
    {
        // This runs inside an Activity named "ProcessOrder"
        // TraceId/SpanId are automatically extracted from the envelope
        _repo.Process(msg.OrderId);
        ReplyTraced(new OrderProcessed(msg.OrderId));
    }

    private void HandleStatus(GetOrderStatus msg)
    {
        var status = _repo.GetStatus(msg.OrderId);
        Reply(status);
    }

    private async Task HandleCancel(CancelOrder msg)
    {
        await _repo.CancelAsync(msg.OrderId);
        ReplyTraced(new OrderCancelled(msg.OrderId));
    }
}
```

### Send traced messages

```csharp
// From outside the actor system (e.g., controller)
orderActor.TellTraced(new ProcessOrder(orderId));
var result = await orderActor.AskTraced<OrderProcessed>(new ProcessOrder(orderId));

// From inside another actor
Context.GetActor<OrderActor>().TellTraced(new ProcessOrder(orderId));

// Forward preserving original sender
otherActor.ForwardTraced(new ProcessOrder(orderId));
```

### Use native IWithTracing on messages (optional)

```csharp
// If your message implements IWithTracing, no envelope is created
public record ProcessOrder(Guid OrderId) : IWithTracing
{
    public string? TraceId { get; set; }
    public string? SpanId { get; set; }
}

// TellTraced detects IWithTracing and sets TraceId/SpanId directly
orderActor.TellTraced(new ProcessOrder(orderId));
```

### Behavior switching

```csharp
public class StatefulActor : TracedMessageActor
{
    public StatefulActor()
    {
        Receive<Start>(_ => BecomeStacked(Active));
    }

    private void Active()
    {
        Receive<DoWork>(msg => { /* handle work */ });
        Receive<Stop>(_ => UnbecomeStacked());
    }
}
```

## Common Mistakes

- **Overriding `OnReceive`**: It is `sealed` in `TracedMessageActor`. Use `Receive<T>` handlers instead.
- **Inheriting from `ReceiveActor`**: Use `TracedMessageActor` as your base class, not `ReceiveActor` or `UntypedActor`, when you want tracing.
- **Manually wrapping in `TracedMessageEnvelope`**: `TellTraced`/`AskTraced`/`ForwardTraced` handle wrapping automatically. Only create `TracedMessageEnvelope` manually if you need to customize it.
- **Expecting trace context on plain `Tell`**: Only `TellTraced`/`AskTraced`/`ForwardTraced` propagate trace context. Regular `Tell` does not.
- **Forgetting `ReplyTraced`**: If you want the reply to carry trace context back to the caller, use `ReplyTraced` instead of `Sender.Tell`.
