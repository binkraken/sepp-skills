---
name: servus-startup-patterns
description: Structured Akka.NET startup using Servus.Core's AppBuilder pattern — IServiceSetupContainer, ApplicationSetupContainer, and ActorSystemSetupContainer
invocable: true
---

# Startup Patterns with Servus.Core + Servus.Akka

Use when integrating Akka.NET into applications that use Servus.Core's `AppBuilder` pattern for structured service registration.

## When to Use

- Setting up Akka.NET in a project that uses Servus.Core's `ISetupContainer` pattern
- Wanting a clean separation of actor system configuration from `Program.cs`

## Prerequisites

Both `Servus.Core` (≥ 0.33.2) and `Servus.Akka` (≥ 0.3.10) must be referenced.
`Servus.Core` is a transitive dependency of `Servus.Akka` — add it explicitly to the csproj anyway
so the `AppBuilder` type is directly referenceable.

## Verified API (0.33.2 / 0.3.10)

### AppBuilder — `Servus.Core.Application.Startup`

```csharp
// Entry point — wraps WebApplication.CreateBuilder() internally
AppBuilder.Create()

// Also available: bring your own builder
AppBuilder.Create<T>(T builder, Func<T, IHost> createHost)

// Register a setup container — NOT .With<>()
AppBuilder.WithSetup<TContainer>() where TContainer : class, ISetupContainer, new()
AppBuilder.WithSetup(ISetupContainer container)

// Build returns AppRunner, NOT WebApplication
AppBuilder.Build() -> AppRunner

// Run the app — NOT app.Run()
AppRunner.RunAsync(CancellationToken token = default)
```

### Setup container interfaces — `Servus.Core.Application.Startup`

| Interface / Base class | Purpose | Method to implement |
|---|---|---|
| `IServiceSetupContainer` | Register DI services | `void SetupServices(IServiceCollection, IConfiguration)` |
| `IHostBuilderSetupContainer` | Configure `IHostBuilder` (e.g. service provider factory) | `void ConfigureHostBuilder(IHostBuilder)` |
| `ILoggingSetupContainer` | Configure logging | `void SetupLogging(ILoggingBuilder)` |
| `ApplicationSetupContainer<THost>` | Configure middleware / map routes | `void SetupApplication(THost)` — use `WebApplication` for web apps |

### ActorSystemSetupContainer — `Servus.Akka.Startup`

```csharp
// Abstract base — inherit and implement the two abstract methods.
// Internally calls services.AddAkka(name, BuildSystem).
public abstract class ActorSystemSetupContainer : IServiceSetupContainer
{
    protected abstract string GetActorSystemName();
    protected abstract void BuildSystem(AkkaConfigurationBuilder builder, IServiceProvider serviceProvider);
}
```

## Usage Pattern

### Step 1: Services setup

```csharp
using Servus.Core.Application.Startup;

public sealed class MyServicesSetup : IServiceSetupContainer
{
    public void SetupServices(IServiceCollection services, IConfiguration configuration)
    {
        services.AddSingleton<MyService>();
        // ... other DI registrations
    }
}
```

### Step 2: Actor system setup

```csharp
using Akka.Hosting;
using Servus.Akka.Startup;

public sealed class MyActorSystemSetup : ActorSystemSetupContainer
{
    protected override string GetActorSystemName() => "my-system";

    protected override void BuildSystem(AkkaConfigurationBuilder builder, IServiceProvider serviceProvider)
    {
        // Use WithActors() — WithResolvableActors() does NOT compile in this project
        builder.WithActors((system, registry, resolver) =>
        {
            var myActor = system.ActorOf(resolver.Props<MyActor>(), "my-actor");
            registry.Register<MyActor>(myActor);
        });
    }
}
```

### Step 3: Application / middleware setup

```csharp
using Microsoft.AspNetCore.Builder;
using Servus.Core.Application.Startup;

public sealed class MyApplicationSetup : ApplicationSetupContainer<WebApplication>
{
    protected override void SetupApplication(WebApplication app)
    {
        app.MapGet("/health", () => Results.Ok("healthy"));
        app.UseWebSockets();
        // ... other middleware
    }
}
```

### Step 4: Wire up in Program.cs

```csharp
using Servus.Core.Application.Startup;

var runner = AppBuilder.Create()
    .WithSetup<MyActorSystemSetup>()
    .WithSetup<MyServicesSetup>()
    .WithSetup<MyApplicationSetup>()
    .Build();

await runner.RunAsync();
```

## Common Mistakes

- **`.With<>()` does not exist** — the correct method is `.WithSetup<>()`.
- **`AppRunner` is not `WebApplication`** — don't call `.Run()`, call `.RunAsync()`.
- **`WithResolvableActors()` does not compile** in this project — use `WithActors()` instead.
- **`ActorRef<T>` (typed actor ref) does not compile** in this project — use `IActorRef` + `IRequiredActor<T>` or `ActorRegistry`.
- **Registering multiple `ActorSystemSetupContainer` subclasses** creates separate actor systems — use one per actor system.
- **`Servus.Core` missing from csproj** — even though it's a transitive dependency of `Servus.Akka`, add an explicit `<PackageReference>` so `AppBuilder` is resolvable without relying on transitive exposure.
