---
name: servus-collections
description: Collection types from Servus.Core — CircularQueue, TypeRegistry, LazyValueCache, and collection/async-enumerable extensions
invocable: true
---

# Collections with Servus.Core

Use when you need a bounded circular buffer, a type-keyed registry, a lazy-loading cache, or async enumerable helpers.

## When to Use

- Keeping the last N items in a fixed-size buffer (logs, metrics, recent events)
- Storing and retrieving values by `Type` key (service locator, factory patterns)
- Caching computed values with expiration using `IMemoryCache`
- Aggregating `IAsyncEnumerable<bool>` results with short-circuit evaluation

## API Reference

**Namespace:** `Servus.Core.Collections`

### CircularQueue&lt;T&gt;

```csharp
public class CircularQueue<T>(int capacity)
{
    void Enqueue(T item);          // Overwrites oldest when full
    bool TryDequeue(out T item);
    int Count { get; }
    int Capacity { get; }
    IEnumerable<T> Items { get; }
}
```

### TypeRegistry&lt;TValue&gt;

```csharp
public class TypeRegistry<TValue>
{
    void Add<TKey>(TValue value);
    void Add(Type key, TValue value);         // AddOrUpdate semantics
    TValue Get<TKey>();                        // Throws KeyNotFoundException
    TValue Get(Type key);
    TValue GetOrAdd<TKey>(Func<TValue> factory);
    TValue GetOrAdd(Type key, Func<TValue> factory);
}
```

Thread-safe — backed by `ConcurrentDictionary`.

### LazyValueCache&lt;TKey, TValue&gt;

```csharp
public sealed class LazyValueCache<TKey, TValue> where TKey : notnull
{
    LazyValueCache(IMemoryCache? cache = null, TimeSpan? defaultExpiration = null);

    TimeSpan DefaultExpiration { get; set; }   // Default: 30 minutes
    TValue GetOrCreate(TKey type, Func<TValue> provider, TimeSpan? expiration = null);
    bool TryGetValue(TKey type, out TValue? value);
}
```

### Collection Extensions

**Namespace:** `Servus.Core`

```csharp
public static class CollectionExtensions
{
    void AddRange<T>(this ICollection<T> collection, IEnumerable<T> items);
    bool IsEmpty<T>(this IReadOnlyCollection<T> enumerable);
}
```

### AsyncEnumerable Extensions

```csharp
public static class AsyncEnumerableExtensions
{
    Task<bool> AnyAsync(this IAsyncEnumerable<bool> enumerable);
    Task<bool> AnyAsync<T>(this IAsyncEnumerable<T> enumerable, Func<T, bool> predicate);
    Task<bool> AllAsync(this IAsyncEnumerable<bool> enumerable);
    Task<bool> AllAsync<T>(this IAsyncEnumerable<T> enumerable, Func<T, bool> predicate);
}
```

Both `AnyAsync` and `AllAsync` use short-circuit evaluation — they stop iterating as soon as the result is determined.

## Usage Pattern

### CircularQueue as a rolling buffer

```csharp
var recentLogs = new CircularQueue<LogEntry>(100);

// Always keeps the last 100 entries
recentLogs.Enqueue(new LogEntry("Request received"));
recentLogs.Enqueue(new LogEntry("Processing started"));

// Iterate over buffered items
foreach (var log in recentLogs.Items)
    Console.WriteLine(log);
```

### TypeRegistry for factory lookup

```csharp
var serializers = new TypeRegistry<ISerializer>();

serializers.Add<OrderEvent>(new JsonSerializer());
serializers.Add<MetricEvent>(new ProtobufSerializer());

// Retrieve by type key
var serializer = serializers.Get<OrderEvent>();
serializer.Serialize(myEvent);

// Lazy creation
var s = serializers.GetOrAdd<PaymentEvent>(() => new JsonSerializer());
```

### LazyValueCache for computed values

```csharp
var cache = new LazyValueCache<string, UserProfile>(
    defaultExpiration: TimeSpan.FromMinutes(5));

var profile = cache.GetOrCreate("user-42", () =>
{
    // Only called on cache miss
    return LoadUserProfile("user-42");
});
```

### AsyncEnumerable aggregation

```csharp
var registry = new ActionRegistry<IHealthCheck, HealthResult>();

bool allHealthy = await registry
    .RunAllAsync(serviceProvider)
    .AllAsync(result => result.IsHealthy);
```

## Common Mistakes

- **Expecting CircularQueue to block on full**: It silently drops the oldest item. If you need backpressure, use `Channel<T>` instead.
- **Using TypeRegistry.Get without checking existence**: `Get` throws `KeyNotFoundException`. Use `GetOrAdd` if you want lazy creation.
- **Sharing LazyValueCache without considering thread safety**: `LazyValueCache` delegates to `IMemoryCache` which is thread-safe, but the factory `Func` might not be — ensure the provider function is safe to call concurrently.
