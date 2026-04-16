---
name: servus-registry-extensions
description: Shorthand extensions for looking up actors from Akka.Hosting IActorRegistry using Servus.Akka — GetActor, TryGetActor, GetActorAsync
invocable: true
---

# Registry Extensions with Servus.Akka

Use when looking up registered actors from `IActorRegistry`. Provides shorthand extensions on `IActorContext` and `ActorSystem` so you don't need to resolve the registry manually.

## When to Use

- Looking up a registered actor from inside another actor
- Looking up a registered actor from the `ActorSystem`
- Needing a safe try-pattern for actor lookup
- Waiting for an actor to be registered (async startup scenarios)

## API Reference

```csharp
// Direct lookup — throws if actor is not registered
IActorRef GetActor<T>(this IActorContext context)
IActorRef GetActor<T>(this ActorSystem system)

// Safe lookup — returns false if not registered
bool TryGetActor<T>(this IActorContext context, out IActorRef actor)
bool TryGetActor<T>(this ActorSystem system, out IActorRef actor)

// Safe lookup by Type key
bool TryGetActor(this IActorContext context, Type key, out IActorRef actor)
bool TryGetActor(this ActorSystem system, Type key, out IActorRef actor)

// Async lookup — waits for actor to be registered
Task<IActorRef> GetActorAsync<T>(this IActorContext context)
Task<IActorRef> GetActorAsync<T>(this ActorSystem system)

// Access the registry directly
IReadOnlyActorRegistry GetRegistry(this ActorSystem system)
```

All generic methods use the actor type `T` as the registry key. Actors must be registered via `WithResolvableActors` or Akka.Hosting's `WithActors` to be found.

## Usage Pattern

### Direct lookup inside an actor

```csharp
public class OrderActor : ReceiveActor
{
    public OrderActor()
    {
        Receive<ProcessOrder>(msg =>
        {
            // Lookup another registered actor by type
            var paymentActor = Context.GetActor<PaymentActor>();
            paymentActor.Tell(new ChargeCustomer(msg.OrderId, msg.Amount));
        });
    }
}
```

### Safe lookup with TryGetActor

```csharp
Receive<NotifyUser>(msg =>
{
    if (Context.TryGetActor<NotificationActor>(out var notifier))
    {
        notifier.Tell(new SendEmail(msg.UserId, msg.Content));
    }
    else
    {
        Log.Warning("NotificationActor not registered — skipping notification");
    }
});
```

### Async lookup during startup

```csharp
// Useful when actor registration order is not guaranteed
protected override async void PreStart()
{
    var paymentActor = await Context.GetActorAsync<PaymentActor>();
    // Now safe to use paymentActor
}
```

### Lookup by Type (dynamic dispatch)

```csharp
Receive<RouteMessage>(msg =>
{
    if (Context.TryGetActor(msg.TargetActorType, out var target))
    {
        target.Forward(msg.Payload);
    }
});
```

## Common Mistakes

- **Using `GetActor<T>` for optional actors**: If the actor might not be registered, use `TryGetActor<T>` to avoid exceptions.
- **Calling `GetActorAsync<T>` in the constructor**: Use `PreStart` for async operations. Actor constructors must be synchronous.
- **Confusing registry key with actor name**: The registry uses the actor's type `T` as the key, not its name string. The type must match what was used during `Register<T>()`.
- **Looking up actors created with `ResolveActor<T>`**: `ResolveActor`/`ResolveChildActor` do not register actors in the registry. Only `WithResolvableActors`/`WithResolvableActor<T>` do.
