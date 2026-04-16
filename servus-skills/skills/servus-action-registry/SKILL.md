---
name: servus-action-registry
description: Register and execute typed actions with ActionRegistry from Servus.Core — sequential, parallel, and async execution with IAsyncTask
invocable: true
---

# ActionRegistry & IAsyncTask with Servus.Core

Use when registering and executing a set of typed actions — either by type (resolved via DI) or by instance. Supports sequential, parallel, and async execution patterns.

## When to Use

- Running a batch of startup tasks, health checks, or migration steps
- Executing multiple implementations of a task interface in parallel
- Building plugin-style architectures where tasks are registered by type and resolved at runtime
- Collecting results from multiple async operations via `IAsyncEnumerable`

## API Reference

**Namespace:** `Servus.Core.Threading.Tasks`

### Interfaces

```csharp
// Marker interface for task identification
public interface ITaskMarker;

// Async task without result
public interface IAsyncTask : ITaskMarker
{
    ValueTask RunAsync(CancellationToken token);
}

// Async task with result
public interface IAsyncTask<T> : ITaskMarker
{
    ValueTask<T> RunAsync(CancellationToken token);
}

// Registration
public interface IActionRegistry<T>
{
    void Register<TImplementation>() where TImplementation : T;
    void Register(T instance);
}

// Execution
public interface IActionRegistryRunner<out T>
{
    void RunAll(IServiceProvider sp, Action<T, CancellationToken> executor, CancellationToken token = default);
    ValueTask RunAllAsync(IServiceProvider sp, Func<T, CancellationToken, ValueTask> executor, CancellationToken token = default);
    ValueTask RunAsyncParallel(IServiceProvider sp, Func<T, CancellationToken, ValueTask> executor, CancellationToken token);
}
```

### ActionRegistry

```csharp
// Base registry — register by type (DI-resolved) or instance
public class ActionRegistry<T> : IActionRegistry<T>, IActionRegistryRunner<T>

// Extended registry — yields results from IAsyncTask<TOut>
public class ActionRegistry<TIn, TOut> : ActionRegistry<TIn> where TIn : IAsyncTask<TOut>
{
    IAsyncEnumerable<TOut> RunAllAsync(IServiceProvider serviceProvider, CancellationToken token = default);
}
```

Type-registered actions are resolved via `IServiceProvider` using `Activator.CreateInstance` with constructor injection.

## Usage Pattern

### Define a task interface and implementations

```csharp
public interface IStartupTask : IAsyncTask { }

public class MigrateDatabase : IStartupTask
{
    private readonly IDbContext _db;
    public MigrateDatabase(IDbContext db) => _db = db;

    public async ValueTask RunAsync(CancellationToken token)
    {
        await _db.MigrateAsync(token);
    }
}

public class SeedCacheData : IStartupTask
{
    private readonly ICacheService _cache;
    public SeedCacheData(ICacheService cache) => _cache = cache;

    public async ValueTask RunAsync(CancellationToken token)
    {
        await _cache.WarmUpAsync(token);
    }
}
```

### Register and execute sequentially

```csharp
var registry = new ActionRegistry<IStartupTask>();

// Register by type — resolved from DI at execution time
registry.Register<MigrateDatabase>();
registry.Register<SeedCacheData>();

// Run all sequentially
await registry.RunAllAsync(
    serviceProvider,
    (task, token) => task.RunAsync(token),
    cancellationToken);
```

### Execute in parallel

```csharp
// Run all in parallel using Parallel.ForEachAsync
await registry.RunAsyncParallel(
    serviceProvider,
    (task, token) => task.RunAsync(token),
    cancellationToken);
```

### Collect results with ActionRegistry<TIn, TOut>

```csharp
public interface IHealthCheck : IAsyncTask<HealthResult> { }

var registry = new ActionRegistry<IHealthCheck, HealthResult>();
registry.Register<DatabaseHealthCheck>();
registry.Register<CacheHealthCheck>();

// Yields results as IAsyncEnumerable
await foreach (var result in registry.RunAllAsync(serviceProvider, cancellationToken))
{
    Console.WriteLine($"{result.Name}: {result.Status}");
}
```

### Register instances directly

```csharp
var registry = new ActionRegistry<IStartupTask>();

// Register a pre-built instance instead of a type
registry.Register(new SeedCacheData(myCache));
```

## Common Mistakes

- **Forgetting DI registration**: Type-registered actions are resolved via `Activator.CreateInstance` with constructor DI — their dependencies must be in the `IServiceProvider`.
- **Using `RunAsyncParallel` for non-thread-safe tasks**: Parallel execution means tasks run concurrently. Ensure the task implementations are thread-safe.
- **Mixing `RunAll` (sync) with async work**: `RunAll` takes a synchronous `Action`. For async tasks, use `RunAllAsync` or `RunAsyncParallel`.
- **Expecting ordering from `RunAsyncParallel`**: Parallel execution does not guarantee order. Use `RunAllAsync` (sequential) if order matters.
