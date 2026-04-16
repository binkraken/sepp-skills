---
name: servus-startup-gates
description: Startup gates with Servus.Core AppBuilder — IStartupGate, WithStartupGate, exponential backoff retry until dependencies are ready
invocable: true
---

# Startup Gates with Servus.Core

Use when your application must wait for external dependencies (database, message broker, external API) to be available before starting.

## When to Use

- Waiting for a database to be reachable before the web host starts accepting requests
- Ensuring a message broker connection is established before processing messages
- Implementing health-check gates with automatic retry and exponential backoff
- Registering gates via type (DI-resolved) or inline function

## API Reference

**Namespace:** `Servus.Core.Application.Startup`

### IStartupGate

```csharp
public interface IStartupGate
{
    Task<bool> CheckAsync(CancellationToken token);
}
```

### AppBuilder gate registration

```csharp
public class AppBuilder
{
    // Register a gate type — resolved from DI
    AppBuilder WithStartupGate<TGate>() where TGate : IStartupGate;

    // Register an inline gate function
    AppBuilder WithStartupGate(Func<Task<bool>> gate);

    // Register a gate instance
    AppBuilder WithStartupGate(IStartupGate gate);
}
```

### ISetupStartupGates

```csharp
// Implement on a setup container to register gates during setup
public interface ISetupStartupGates
{
    void OnRegisterStartupGates(/* gate registration context */);
}
```

### Execution behavior

Gates run after `CoreSetupAsync()` completes but before the host starts. All registered gates are checked in sequence. If any gate returns `false`, the runner waits with **exponential backoff** (starting at 1 second, capped at 60 seconds) and retries all gates until they all pass or cancellation is requested.

## Usage Pattern

### Implement a startup gate

```csharp
public class DatabaseGate : IStartupGate
{
    private readonly IDbConnection _connection;

    public DatabaseGate(IDbConnection connection)
    {
        _connection = connection;
    }

    public async Task<bool> CheckAsync(CancellationToken token)
    {
        try
        {
            await _connection.OpenAsync(token);
            return true;
        }
        catch
        {
            return false; // Will be retried with backoff
        }
    }
}
```

### Register gates in AppBuilder

```csharp
var runner = AppBuilder.Create()
    .WithSetup<MyServicesSetup>()
    .WithSetup<MyActorSystemSetup>()

    // Register by type — resolved from DI
    .WithStartupGate<DatabaseGate>()

    // Register inline
    .WithStartupGate(async () =>
    {
        try
        {
            using var client = new HttpClient();
            var response = await client.GetAsync("http://broker:15672/api/health");
            return response.IsSuccessStatusCode;
        }
        catch { return false; }
    })

    .Build();

// Gates are checked before the host starts
await runner.RunAsync();
```

### Multiple gates

```csharp
AppBuilder.Create()
    .WithStartupGate<DatabaseGate>()
    .WithStartupGate<MessageBrokerGate>()
    .WithStartupGate<ExternalApiGate>()
    .Build();

// All three must return true before the app starts.
// If any returns false → exponential backoff → retry all.
```

### Lifecycle events alongside gates

```csharp
AppBuilder.Create()
    .WithStartupGate<DatabaseGate>()
    .OnApplicationStarted(sp =>
    {
        // Runs AFTER all gates pass and host starts
        var logger = sp.GetRequiredService<ILogger<Program>>();
        logger.LogInformation("All gates passed, application started");
    })
    .OnApplicationStopping(() => Console.WriteLine("Shutting down..."))
    .OnApplicationStopped(() => Console.WriteLine("Stopped"))
    .Build();
```

## Common Mistakes

- **Throwing exceptions in `CheckAsync`**: Return `false` instead of throwing. The retry mechanism expects a boolean result. Exceptions will propagate and may crash the startup.
- **Not using CancellationToken**: Always honor the `CancellationToken` in gate checks to allow graceful shutdown during startup.
- **Long-running checks without timeout**: Each gate check should have its own timeout (e.g., `HttpClient.Timeout`). The retry backoff handles the wait between attempts.
- **Expecting gates to run in parallel**: Gates run sequentially. If one fails, all are retried. Design gates to be fast (check connectivity, not full initialization).
- **Confusing gates with health checks**: Startup gates run once at startup. For runtime health monitoring, use ASP.NET Core health checks (`AddHealthChecks`).
