---
name: resilience-patterns
description: Implement fault tolerance patterns for Akka.NET and .NET services. Supervision, Mitigation/Fallback, Stashing, Circuit Breaker.
invocable: true
---

# Resilience Patterns

You are a resilience architect. Design fault-tolerant systems that **degrade gracefully** instead of failing completely. Every failure mode has a planned response.

## Core Principle

> "What should happen when X fails?" must have an answer BEFORE you write the code.

## Pattern Selection

```
What fails?
├── Child actor crashes → Supervision Strategy
├── Business process fails → Mitigation/Fallback Actor
├── External service is down → Circuit Breaker
├── Component not yet ready → Stashing for Availability
├── Transient network error → Retry with Backoff
└── Nothing expected to fail → No pattern needed (don't over-engineer)
```

---

## Pattern 1: Supervision Strategy

The parent actor decides what happens when a child actor fails. This is Akka.NET's primary resilience mechanism.

### OneForOneStrategy (Default)

Each child is handled independently. Use for actors that don't depend on each other.

```csharp
protected override SupervisorStrategy SupervisorStrategy()
{
    return new OneForOneStrategy(
        maxNrOfRetries: 3,
        withinTimeRange: TimeSpan.FromMinutes(1),
        decider: Decider.From(ex => ex switch
        {
            // Transient errors → Restart the actor (state is reset)
            TimeoutException => Directive.Restart,
            IOException => Directive.Restart,

            // Invalid input → Resume (keep state, skip the message)
            ArgumentException => Directive.Resume,
            ValidationException => Directive.Resume,

            // Fatal errors → Stop the actor permanently
            OutOfMemoryException => Directive.Stop,
            
            // Unknown → Escalate to parent's supervisor
            _ => Directive.Escalate
        }));
}
```

### AllForOneStrategy

Restart ALL children when one fails. Use when children have shared state or depend on each other.

```csharp
protected override SupervisorStrategy SupervisorStrategy()
{
    return new AllForOneStrategy(
        maxNrOfRetries: 3,
        withinTimeRange: TimeSpan.FromMinutes(1),
        decider: Decider.From(ex => ex switch
        {
            // If one child in the group fails, restart all
            _ => Directive.Restart
        }));
}
```

### Decision Guide

| Directive | When to Use | Effect |
|---|---|---|
| **Resume** | Transient error, state is fine | Skip bad message, keep processing |
| **Restart** | State might be corrupted | Reset actor state, continue |
| **Stop** | Permanent failure | Remove actor, don't restart |
| **Escalate** | Parent can't decide | Let grandparent handle it |

### Anti-Patterns
- Never use `Restart` for `OutOfMemoryException` — it will just happen again
- Never use `Resume` for unknown exceptions — corrupted state goes undetected
- Never set unlimited retries — failed actors consume resources forever
- Never ignore `Escalate` — if you can't decide, let the parent decide

---

## Pattern 2: Mitigation/Fallback Actor

A dedicated actor that handles failure recovery as a first-class concern. Separates "normal path" from "failure path."

```csharp
public class RecoveryCoordinator : ReceiveActor
{
    public RecoveryCoordinator(IServiceProvider serviceProvider)
    {
        var scope = serviceProvider.CreateScope();
        var repository = scope.ServiceProvider.GetRequiredService<IOrderRepository>();

        // Mitigation action: refund a failed payment
        Receive<RefundPayment>(msg =>
        {
            repository.GetOrder(msg.OrderId)
                .Some(order =>
                {
                    var refundCommand = new IssueRefund(order.PaymentId, order.Total);
                    Log.Information("Mitigation: issuing refund for order [{Id}]", msg.OrderId);
                    Context.GetRegistry().Get<PaymentGateway>().Tell(refundCommand);
                })
                .None(() =>
                    Log.Error("Mitigation failed: order [{Id}] not found", msg.OrderId));
        });

        // Mitigation action: park an item back in storage
        Receive<ReturnToStorage>(msg =>
        {
            Log.Information("Mitigation: returning item [{Id}] to storage", msg.ItemId);
            Context.ResolveActor<StorageActor>(
                msg.ItemId.ToString(), msg.ItemId);
        });

        // Generic fallback: log and alert
        ReceiveAny(msg =>
        {
            Log.Warning("RecoveryCoordinator received unhandled: [{Type}]", msg.GetType().Name);
        });
    }
}
```

### When to Use
- Business process has a meaningful fallback (not just "log and continue")
- Recovery logic is complex enough to deserve its own actor
- Multiple failure types need coordinated recovery
- You need audit trail of mitigation actions

### Integration Pattern
```csharp
// In the normal-path actor:
try
{
    ProcessOrder(order);
}
catch (ProcessingException ex)
{
    // Delegate to RecoveryCoordinator instead of handling inline
    var recovery = Context.GetRegistry().Get<RecoveryCoordinator>();
    recovery.Tell(new RefundPayment(orderId));
}
```

---

## Pattern 3: Stashing for Availability

Queue messages during temporary unavailability (initialization, reconnection, maintenance).

```csharp
public class ResilientService : ReceiveActor, IWithUnboundedStash
{
    public IStash Stash { get; set; } = null!;

    public ResilientService()
    {
        // Unavailable state: stash everything, attempt recovery
        Become(Unavailable);
    }

    private void Unavailable()
    {
        Receive<RecoveryComplete>(_ =>
        {
            Log.Information("Service recovered, processing {Count} stashed messages");
            Become(Available);
        });

        Receive<RecoveryFailed>(msg =>
        {
            Log.Warning("Recovery failed, retrying in 5s: {Reason}", msg.Reason);
            Context.System.Scheduler.ScheduleTellOnce(
                TimeSpan.FromSeconds(5), Self, new RetryRecovery(), Self);
        });

        Receive<RetryRecovery>(_ => AttemptRecovery());

        ReceiveAny(msg =>
        {
            Log.Debug("Stashing [{Type}] while unavailable", msg.GetType().Name);
            Stash.Stash();
        });
    }

    private void Available()
    {
        Stash.UnstashAll();

        Receive<DoWork>(msg => ProcessWork(msg));

        Receive<ConnectionLost>(_ =>
        {
            Log.Warning("Connection lost, entering unavailable state");
            Become(Unavailable);
            AttemptRecovery();
        });
    }

    private void AttemptRecovery()
    {
        RecoverAsync()
            .PipeTo(Self,
                success: _ => new RecoveryComplete(),
                failure: ex => new RecoveryFailed(ex.Message));
    }

    private record RecoveryComplete;
    private record RecoveryFailed(string Reason);
    private record RetryRecovery;
}
```

### Key Points
- Always have a path from Unavailable → Available (don't stash forever)
- Implement retry with backoff for recovery attempts
- Log stash/unstash transitions for debugging
- Consider a stash timeout — dead letters are better than infinite waiting

---

## Pattern 4: Circuit Breaker

Protect against cascading failures when calling external services.

### For HTTP Calls (using Polly)

```csharp
// Registration:
services.AddHttpClient<IExternalApi>()
    .AddPolicyHandler(GetCircuitBreakerPolicy());

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30),
            onBreak: (result, duration) =>
                Log.Warning("Circuit OPEN for {Duration}s: {Reason}",
                    duration.TotalSeconds, result.Exception?.Message),
            onReset: () => Log.Information("Circuit CLOSED — service recovered"),
            onHalfOpen: () => Log.Information("Circuit HALF-OPEN — testing..."));
}
```

### For Actor-Based Calls

```csharp
// Akka.NET has a built-in CircuitBreaker:
private readonly CircuitBreaker _breaker;

public ExternalServiceActor()
{
    _breaker = new CircuitBreaker(
        maxFailures: 5,
        callTimeout: TimeSpan.FromSeconds(10),
        resetTimeout: TimeSpan.FromSeconds(30))
        .OnOpen(() => Log.Warning("Circuit breaker OPEN"))
        .OnClose(() => Log.Information("Circuit breaker CLOSED"))
        .OnHalfOpen(() => Log.Information("Circuit breaker HALF-OPEN"));

    Receive<CallExternalService>(msg =>
    {
        _breaker.WithCircuitBreaker(() => CallServiceAsync(msg))
            .PipeTo(Self,
                success: result => new ServiceResult(result),
                failure: ex => new ServiceCallFailed(ex));
    });

    Receive<ServiceCallFailed>(msg =>
    {
        if (msg.Exception is OpenCircuitException)
        {
            // Circuit is open — use fallback
            Sender.Tell(new FallbackResult());
            return;
        }
        // Other failures
        Log.Error(msg.Exception, "Service call failed");
    });
}
```

### Circuit Breaker States
```
CLOSED (normal)
  ↓ N failures within window
OPEN (rejecting all calls)
  ↓ after reset timeout
HALF-OPEN (allowing one probe call)
  ↓ probe succeeds → CLOSED
  ↓ probe fails → OPEN
```

---

## Pattern 5: Retry with Exponential Backoff

For transient failures that resolve on their own.

### With Polly (HTTP)
```csharp
services.AddHttpClient<IExternalApi>()
    .AddPolicyHandler(Policy<HttpResponseMessage>
        .Handle<HttpRequestException>()
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: attempt =>
                TimeSpan.FromSeconds(Math.Pow(2, attempt)),  // 2s, 4s, 8s
            onRetry: (outcome, delay, attempt, _) =>
                Log.Warning("Retry {Attempt} after {Delay}s: {Error}",
                    attempt, delay.TotalSeconds, outcome.Exception?.Message)));
```

### With Actor Scheduler
```csharp
Receive<RetryableWork>(msg =>
{
    try
    {
        DoWork(msg);
    }
    catch (TransientException)
    {
        if (msg.Attempt < 3)
        {
            var delay = TimeSpan.FromSeconds(Math.Pow(2, msg.Attempt));
            Context.System.Scheduler.ScheduleTellOnce(
                delay, Self, msg with { Attempt = msg.Attempt + 1 }, Self);
        }
        else
        {
            Log.Error("Work failed after {Attempts} retries", msg.Attempt);
            // Escalate to mitigation
        }
    }
});

public record RetryableWork(string Data, int Attempt = 0);
```

---

## Pattern 6: Startup Dependency Retry

Before the app starts serving traffic, verify all dependencies are available. Retry with configurable limits.

```csharp
public static async Task WaitForDependencies(
    IServiceProvider services,
    IReadOnlyList<IStartupCheck> checks,
    int maxRetries = 10,
    int delayMs = 3000)
{
    var pending = checks.ToList();

    for (int attempt = 1; attempt <= maxRetries; attempt++)
    {
        var failing = new List<IStartupCheck>();

        foreach (var check in pending)
        {
            using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
            try
            {
                var ok = await check.CheckAsync(services, cts.Token);
                if (ok)
                    Log.Information("Startup check [{Name}] passed", check.Name);
                else
                    failing.Add(check);
            }
            catch (Exception ex)
            {
                Log.Warning("Startup check [{Name}] threw: {Error}", check.Name, ex.Message);
                failing.Add(check);
            }
        }

        if (failing.Count == 0)
        {
            Log.Information("All dependencies ready after {N} attempt(s)", attempt);
            return;
        }

        Log.Warning("Attempt {N}/{Max}: waiting for [{Names}]",
            attempt, maxRetries,
            string.Join(", ", failing.Select(c => c.Name)));

        pending = failing;
        if (attempt < maxRetries)
            await Task.Delay(delayMs);
    }

    throw new InvalidOperationException(
        $"Dependencies not ready after {maxRetries} retries: " +
        string.Join(", ", pending.Select(c => c.Name)));
}
```

**When to use:**
- Container orchestration (k8s) where deps start in parallel
- Services that depend on databases, message brokers, or other services
- Before `app.RunAsync()` — not after

**Key decisions:**
- `maxRetries`: 10 for k8s (with startupProbe), 3 for local dev
- `delayMs`: 3000ms is a good default (enough for DB warmup)
- Timeout per check: 10s prevents hanging on unresponsive deps
- Failed checks are retried, passed checks are skipped

---

## Combining Patterns

Real systems combine multiple resilience patterns:

```
External Request
    ↓
[Circuit Breaker] — rejects if service is down
    ↓
[Retry with Backoff] — retries transient failures
    ↓
[Stashing Actor] — queues during temporary unavailability
    ↓
[Supervision] — restarts on unexpected crashes
    ↓
[Mitigation] — fallback when all else fails
```

Don't use all of them everywhere. Use the **minimum set** that handles your actual failure modes.

## Decision Matrix

| Failure Type | Pattern | Complexity |
|---|---|---|
| Child actor crash | Supervision | Low |
| Business process failure with fallback | Mitigation Actor | Medium |
| Component not ready yet | Stashing | Low |
| External service flaky | Circuit Breaker + Retry | Medium |
| Network partition | Stashing + Reconnect | Medium |
| Unknown/unexpected | Supervision (Escalate) | Low |
| None expected | No pattern | None |
