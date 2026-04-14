---
name: state-machine-designer
description: Design explicit state machines with event-sourced state, transition guards, and side-effect isolation. States, transitions, recovery, visualization.
---

# State Machine Designer

You are a state machine architect. Design state machines that are **explicit, recoverable, and testable**. Every state, transition, and guard is documented and enforced in code.

## Core Principles

1. **States are explicit** — Named types or enum values, never implicit boolean combinations
2. **Transitions are pure** — `(State, Event) → (NewState, SideEffects)` — no hidden mutations
3. **Side effects are isolated** — Happen AFTER state transition, never during
4. **Recovery = Replay** — Rebuild state by replaying events from the beginning

## When to Use a State Machine

Use when:
- An entity has 3+ distinct states with different behaviors per state
- State transitions follow rules (not all transitions are valid)
- You need to recover state after restarts
- The lifecycle is complex enough that a boolean flag doesn't cut it

Don't use when:
- Only 2 states (use a boolean or simple enum + if/else)
- No rules about transitions (any state → any state)
- State is trivially derived from data (computed property)

## Design Process

### Step 1: Identify States

List all distinct states. A state is "distinct" if the entity behaves differently.

```csharp
// Option A: Enum (simple, when states have no data)
public enum OrderState
{
    Created,
    PaymentPending,
    Confirmed,
    Shipping,
    Delivered,
    Cancelled
}

### Step 2: Define Events

Events are the triggers that cause state transitions.

```csharp
public abstract record OrderEvent;
public record PaymentRequested(Guid PaymentId, decimal Amount) : OrderEvent;
public record PaymentConfirmed(Guid PaymentId) : OrderEvent;
public record PaymentFailed(string Reason) : OrderEvent;
public record ShipmentDispatched(string TrackingNumber) : OrderEvent;
public record ShipmentDelivered(DateTimeOffset DeliveredAt) : OrderEvent;
public record OrderCancelled(string Reason) : OrderEvent;
```

### Step 3: Integrate with Akka.NET (if using actors)

The `Become()` pattern in Akka.NET is a built-in state machine:

```csharp
public class OrderActor : ReceivePersistentActor
{
    private OrderState _state = new Created(DateTimeOffset.UtcNow);

    public override string PersistenceId { get; }

    public OrderActor(string entityId, IServiceProvider sp)
    {
        PersistenceId = entityId;

        // Recovery
        Recover<OrderEvent>(evt => _state = OrderTransitions.Apply(_state, evt));

        // Initial state behavior
        Command<OrderEnvelope>(UnpackAndDispatch);
        CommandAny(_ => Stash.Stash());
    }

    protected override void OnReplaySuccess() => TransitionTo(_state);

    private void TransitionTo(OrderState state)
    {
        _state = state;
        switch (state)
        {
            case Created:
                Become(CreatedBehavior);
                break;
            case PaymentPending:
                Become(PaymentPendingBehavior);
                break;
            case Confirmed:
                Become(ConfirmedBehavior);
                Stash.UnstashAll();
                break;
            case Shipping:
                Become(ShippingBehavior);
                break;
        }
    }

    private void ConfirmedBehavior()
    {
        Command<ShipmentDispatched>(msg =>
        {
            Persist(msg, e =>
            {
                _state = OrderTransitions.Apply(_state, e);
                TransitionTo(_state);
            });
        });
        // ... other Confirmed-state handlers
    }
}
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Boolean flags for state (`isReady`, `isProcessing`) | Combinatorial explosion, invalid states | Explicit state enum/records |
| State transitions with side effects mixed in | Untestable, hard to reason about | Separate Apply from Execute |
| Missing transitions (silent no-op) | Bugs go undetected | Log or throw on unexpected transitions |
| God state (one state does everything) | State machine is pointless | Split into real states with distinct behavior |
| Nested state machines without clear boundaries | Complexity explosion | Flatten or use hierarchical states explicitly |
