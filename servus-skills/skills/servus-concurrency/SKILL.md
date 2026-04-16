---
name: servus-concurrency
description: Concurrency utilities from Servus.Core — NamedSemaphoreSlimStore for shared locks, SemaphoreSlim scoped extensions, BlockingTimer
invocable: true
---

# Concurrency Utilities with Servus.Core

Use when you need named shared locks, scoped semaphore patterns, or a blocking timer that ensures serial execution.

## When to Use

- Coordinating access to shared resources by name (e.g., per-entity or per-tenant locking)
- Using `SemaphoreSlim` in a `using` block for automatic release
- Running a periodic action that must not overlap (timer that waits for completion)

## API Reference

### NamedSemaphoreSlimStore

**Namespace:** `Servus.Core.Concurrency`

```csharp
// Static store — manages a shared dictionary of named semaphores
public static class NamedSemaphoreSlimStore
{
    // Get or create a named semaphore (thread-safe)
    static NamedSemaphoreSlim OpenOrCreate(
        string name, int defaultInitialCount, int defaultMaximumCount);
}

// Named semaphore — extends SemaphoreSlim with name and reference counting
public class NamedSemaphoreSlim : SemaphoreSlim
{
    string Name { get; }
    // Auto-removed from store when disposed and no more references
}
```

### SemaphoreSlim Extensions

**Namespace:** `Servus.Core.Threading`

```csharp
public static class SemaphoreSlimExtensions
{
    // Async scoped lock — releases on Dispose
    static Task<IDisposable> WaitScopedAsync(
        this SemaphoreSlim semaphoreSlim, CancellationToken cancellationToken = default);

    // Sync scoped lock — releases on Dispose
    static IDisposable WaitScoped(this SemaphoreSlim semaphoreSlim);
}
```

### BlockingTimer

**Namespace:** `Servus.Core.Threading`

```csharp
public sealed class BlockingTimer : IDisposable
{
    // Starts immediately — action runs on a LongRunning task
    BlockingTimer(Action timerAction, double intervalInMilliseconds, CancellationToken cancellationToken);

    // Stop and wait for current action to complete
    void Stop();
}
```

If the action takes longer than the interval, the next tick waits for it to finish — no overlapping executions.

## Usage Pattern

### Named locks for per-entity access

```csharp
// Multiple callers sharing a lock by name
public async Task ProcessOrderAsync(string orderId, CancellationToken ct)
{
    var semaphore = NamedSemaphoreSlimStore.OpenOrCreate(
        name: $"order-{orderId}",
        defaultInitialCount: 1,
        defaultMaximumCount: 1);

    await semaphore.WaitAsync(ct);
    try
    {
        await DoExclusiveWork(orderId);
    }
    finally
    {
        semaphore.Release();
    }
}
```

### Scoped semaphore with using

```csharp
private readonly SemaphoreSlim _lock = new(1, 1);

public async Task UpdateAsync(CancellationToken ct)
{
    using var scope = await _lock.WaitScopedAsync(ct);
    // Lock is held for the duration of this scope
    await PerformUpdate();
    // Automatically released when scope is disposed
}

// Synchronous variant
public void Update()
{
    using var scope = _lock.WaitScoped();
    PerformUpdate();
}
```

### BlockingTimer for periodic serial work

```csharp
var cts = new CancellationTokenSource();

// Polls every 5 seconds — never overlaps
var timer = new BlockingTimer(
    timerAction: () => PollForChanges(),
    intervalInMilliseconds: 5000,
    cancellationToken: cts.Token);

// Later: stop gracefully
timer.Stop();    // waits for current action to complete
timer.Dispose();
```

## Common Mistakes

- **Forgetting to release NamedSemaphoreSlim**: Unlike `WaitScopedAsync`, raw `WaitAsync`/`Release` on `NamedSemaphoreSlim` requires manual release. Prefer `WaitScopedAsync` for safety.
- **Using BlockingTimer for async work**: `BlockingTimer` takes a synchronous `Action`. If you need async periodic work, consider a `PeriodicTimer` with `async`/`await` instead.
- **Creating NamedSemaphoreSlim directly**: Always use `NamedSemaphoreSlimStore.OpenOrCreate` — it manages the shared dictionary and reference counting.
- **Ignoring CancellationToken in WaitScopedAsync**: Always pass a `CancellationToken` to avoid hanging indefinitely if the semaphore is never released.
