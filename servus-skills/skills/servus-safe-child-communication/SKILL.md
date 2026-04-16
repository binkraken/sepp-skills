---
name: servus-safe-child-communication
description: Safe child actor lookup and messaging with Servus.Akka — GetChild returns Option, ChildTell/ChildForward return bool
invocable: true
---

# Safe Child Communication with Servus.Akka

Use when communicating with child actors where the child may or may not exist. Eliminates `Nobody` checks and silent message loss.

## When to Use

- Sending messages to child actors that may not have been created yet
- Forwarding messages to optional child actors
- Replacing manual `Context.Child(name)` + `IsNobody()` checks
- Adding trace context to child messages

## API Reference

```csharp
// Safe child lookup — returns Option<IActorRef> instead of potentially Nobody
Option<IActorRef> GetChild(this IActorContext context, string name)

// Tell message to child — returns false if child doesn't exist
bool ChildTell(this IActorContext context, string name, object message)

// Forward message to child (preserves original sender) — returns false if child doesn't exist
bool ChildForward(this IActorContext context, string name, object message)

// Traced variants — add TraceId/SpanId before sending
bool ChildTellTraced(this IActorContext context, string name, IWithTracing message)
bool ChildForwardTraced(this IActorContext context, string name, IWithTracing message)
```

## Usage Pattern

### Basic: Tell or Forward to optional children

```csharp
public class SupervisorActor : ReceiveActor
{
    public SupervisorActor()
    {
        Receive<WorkItem>(msg =>
        {
            // Returns false if "worker-1" doesn't exist — no exception, no dead letter
            if (!Context.ChildTell("worker-1", msg))
            {
                // Child doesn't exist — handle accordingly
                var worker = Context.ActorOf(Props.Create<WorkerActor>(), "worker-1");
                worker.Tell(msg);
            }
        });

        Receive<ForwardToChild>(msg =>
        {
            // Forward preserves original sender
            Context.ChildForward(msg.ChildName, msg.Payload);
        });
    }
}
```

### With Option<T> pattern matching

```csharp
Receive<QueryChild>(msg =>
{
    Context.GetChild("processor").Match(
        some: actor => actor.Tell(new GetStatus()),
        none: () => Sender.Tell(new ChildNotFound(msg.Name))
    );
});
```

### With distributed tracing

```csharp
Receive<TracedCommand>(msg =>
{
    // Automatically calls msg.AddTracing() before sending
    Context.ChildTellTraced("worker", msg);
});
```

## Common Mistakes

- **Using `Context.Child()` directly**: Returns `Nobody` if child doesn't exist — messages sent to Nobody are silently lost. Use `GetChild` or `ChildTell` instead.
- **Forgetting the bool return**: `ChildTell`/`ChildForward` return `false` when the child doesn't exist. Check the return value when you need to handle the missing-child case.
- **Using `ChildTellTraced` with non-IWithTracing messages**: The traced variants require messages implementing `IWithTracing`. For plain objects, use `ChildTell`.
- **Assuming ChildTell creates the child**: These methods only send to existing children. Use `Context.ActorOf` or `Context.ResolveChildActor<T>` to create children first.
