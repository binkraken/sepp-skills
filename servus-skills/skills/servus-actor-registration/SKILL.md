---
name: servus-actor-registration
description: Register Akka.NET actors with DI support and resolve ActorRef<T> from IServiceProvider using Servus.Akka
invocable: true
---

# Actor Registration & Dependency Injection with Servus.Akka

Use when registering actors that need constructor-injected services, or when resolving typed actor references from the DI container.

## When to Use

- Setting up actors in `AddAkka()` that need services injected
- Wanting to resolve `ActorRef<T>` from `IServiceProvider` (e.g., in controllers, services)
- Registering multiple actors in a batch with the fluent helper

## API Reference

### Registration

```csharp
// Batch registration with fluent helper
AkkaConfigurationBuilder.WithResolvableActors(Action<ActorRegistrationHelper> helper)

// Single actor registration
AkkaConfigurationBuilder.WithResolvableActor<TActor>(string? name = null, params object[] args)

// Fluent helper (chainable)
ActorRegistrationHelper.Register<TActor>(string? name = null, params object[] args)
```

All methods use `DependencyResolver.Props<T>()` internally, so actor constructors can accept any service registered in the DI container.

### Resolution from IServiceProvider

```csharp
// ActorRef<T> implements IActorRef — delegates all calls to the real actor
ActorRef<TActor> : IActorRef where TActor : ActorBase
```

Requires `ActorRefProviderFactory` to be registered as the service provider factory.

### Runtime Actor Creation with DI

```csharp
// Create a top-level actor with DI (not registered in IActorRegistry)
ActorSystem.ResolveActor<TActor>(string? name = null, params object[] args)
IActorContext.ResolveActor<TActor>(string? name = null, params object[] args)

// Create a child actor with DI
IActorContext.ResolveChildActor<TActor>(string? name = null, params object[] args)
```

## Usage Pattern

### Step 1: Register actors at startup

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAkka("my-system", b =>
{
    b.WithResolvableActors(helper =>
    {
        helper
            .Register<OrderActor>()
            .Register<PaymentActor>("payment-actor")
            .Register<NotificationActor>();
    });
});
```

### Step 2: Enable ActorRef<T> resolution (optional)

```csharp
// Enables resolving ActorRef<T> from IServiceProvider
builder.Host.UseServiceProviderFactory(new ActorRefProviderFactory());
```

### Step 3: Inject or resolve actors

```csharp
// In a controller or service
public class OrderController(ActorRef<OrderActor> orderActor)
{
    public IActionResult Process(Guid id)
    {
        orderActor.Tell(new ProcessOrder(id));
        return Accepted();
    }
}

// Or resolve manually
var actor = serviceProvider.GetService<ActorRef<OrderActor>>();
actor.Tell(message);
```

### Creating child actors with DI inside an actor

```csharp
public class ParentActor : ReceiveActor
{
    public ParentActor()
    {
        Receive<CreateWorker>(msg =>
        {
            // Creates a child actor with DI — not registered in IActorRegistry
            var worker = Context.ResolveChildActor<WorkerActor>(msg.WorkerId);
        });
    }
}
```

## Common Mistakes

- **Forgetting `ActorRefProviderFactory`**: Without it, `GetService<ActorRef<T>>()` returns null. It is only needed for resolving from `IServiceProvider` — `WithResolvableActors` works without it.
- **Using `Props.Create<T>()` instead**: Plain `Props.Create` does not resolve DI services. Use `WithResolvableActors` or `ResolveActor<T>` instead.
- **Confusing `ResolveActor` with `WithResolvableActors`**: `WithResolvableActors` registers actors in `IActorRegistry` at startup. `ResolveActor`/`ResolveChildActor` creates new actor instances at runtime (not registered).
- **Resolving `IActorRef` instead of `ActorRef<T>`**: The DI fallback only works for the generic `ActorRef<T>` type, not plain `IActorRef`.
