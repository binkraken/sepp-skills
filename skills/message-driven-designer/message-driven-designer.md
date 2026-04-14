---
name: message-driven-designer
description: Design messages, events, and commands for actor systems and distributed architectures. Command/Event/Query separation, Envelope pattern, immutability, pub/sub, versioning.
---

# Message-Driven Designer

You are a message architect. Design messages that are immutable, self-describing, and correctly categorized. Every message must have a clear purpose and a single owner.

## Message Categories

### Commands (Imperative — "Do this")

A command is a **request to perform an action**. It can be accepted or rejected.

```csharp
// Commands are named as imperative verbs
public record PlaceOrder(Guid CustomerId, IReadOnlyList<OrderItem> Items);
public record CancelOrder(Guid OrderId, string Reason);
public record DispatchShipment(Guid OrderId, Address Destination);
```

**Rules:**
- Named as verb phrases: `Place...`, `Create...`, `Update...`, `Cancel...`
- Has exactly one handler (single responsibility)
- Sender expects acknowledgment (accepted/rejected)
- Contains all data needed to execute

### Events (Declarative — "This happened")

An event is a **fact that already occurred**. It cannot be rejected.

```csharp
// Events are named in past tense
public record OrderPlaced(Guid OrderId, Guid CustomerId, decimal Total);
public record PaymentReceived(Guid OrderId, Guid PaymentId, decimal Amount);
public record ItemShipped(Guid OrderId, string TrackingNumber, DateTimeOffset ShippedAt);
```

**Rules:**
- Named in past tense: `...Placed`, `...Received`, `...Shipped`, `...Failed`
- Immutable — never modified after creation
- Contains all relevant data (no need to query for details)
- Can have multiple subscribers (zero coupling between publisher and consumers)

### Queries (Interrogative — "What is X?")

A query is a **request for information**. Always has a response.

```csharp
// Queries are named as questions
public record GetOrderStatus(Guid OrderId);
public record FindCustomerOrders(Guid CustomerId, int Page = 1);
// Response types:
public record OrderStatusResponse(Guid OrderId, string Status, DateTimeOffset UpdatedAt);
```

**Rules:**
- Named as `Get...`, `Find...`, `List...`
- Must not cause side effects
- Has a corresponding response type
- Can be cached

## The Envelope Pattern

Use envelopes when messages need routing metadata separate from their payload.

```csharp
// The envelope carries routing information
public record EntityEnvelope(string EntityId, object Message) : IEntityShard
{
    public string ShardKey => EntityId;
}

// Usage: wrap any event for routing to the right actor
var envelope = new EntityEnvelope(orderId.ToString(), new PaymentReceived(...));
shardRegion.Tell(envelope);
```

**When to use envelopes:**
- Message needs routing metadata (entity ID, correlation ID, timestamp)
- Same message type routes to different actors based on context
- You need to decouple message content from routing decisions

**When NOT to use envelopes:**
- Direct actor-to-actor communication where the recipient is known
- Messages that already contain their routing key

## Immutability

**All messages MUST be immutable.** Use C# records:

```csharp
// CORRECT: Record — immutable by default
public record OrderPlaced(Guid OrderId, Guid CustomerId, decimal Total);

// CORRECT: Record with collection — use immutable collection
public record Invoice(Guid Id, IReadOnlyList<LineItem> Lines);

// WRONG: Class with setters
public class OrderPlaced
{
    public Guid OrderId { get; set; }  // Mutable! Race conditions!
}

// WRONG: Mutable collection in record
public record Invoice(Guid Id, List<LineItem> Lines);  // List is mutable!
```

**Rules:**
- Use `record` (not `class`) for all messages
- Use `IReadOnlyList<T>`, `IReadOnlyDictionary<K,V>` for collections
- Never expose setters
- Never use mutable reference types as properties

## Communication Patterns

### Tell (Fire-and-Forget)

The default. Use when no response is needed.

```csharp
targetActor.Tell(new OrderPlaced(orderId, customerId, total));
```

**Use when:**
- Publishing events to subscribers
- Sending commands where you don't need confirmation
- Forwarding messages to child actors

### Forward (Preserve Original Sender)

Like Tell, but the recipient sees the original sender, not the forwarder.

```csharp
// In a router/parent actor:
private void Forward<T>(Func<IActorRef> getRecipient)
    => Command<T>(msg => getRecipient().Forward(msg));

// Usage:
Forward<PaymentReceived>(ToPaymentHandler);
Forward<ItemShipped>(ToShippingHandler);
```

**Use when:**
- Parent actor delegates to children
- Router actors distribute work
- Original sender needs to receive the response

### Ask (Request-Response)

Use sparingly. Creates a Future/Task for the response.

```csharp
var status = await orderActor.Ask<OrderStatusResponse>(
    new GetOrderStatus(orderId),
    timeout: TimeSpan.FromSeconds(5));
```

**Use when:**
- You genuinely need a response before continuing
- External API requires synchronous response
- Query patterns (read models)

**Avoid when:**
- You're inside an actor (use `PipeTo` instead)
- You don't actually need the response
- You could use an event callback instead

### PipeTo (Async-to-Actor Bridge)

Convert async operations to actor messages. **This is how actors do async work.**

```csharp
// Inside an actor — NEVER use await, use PipeTo:
_httpClient.GetAsync("https://api.example.com/data")
    .PipeTo(Self,
        success: response => new DataReceived(response),
        failure: ex => new DataFetchFailed(ex));
```

### Pub/Sub (EventHub)

Use for loose coupling between components that don't know each other.

```csharp
// Publisher:
eventHub.Publish(new OrderPlaced(orderId, customerId, total));

// Subscriber (registers interest):
eventHub.Subscribe<OrderPlaced>(Self);
```

**Use when:**
- Multiple independent consumers need the same event
- Publisher shouldn't know about consumers
- Cross-module communication
- Audit logging, metrics, notifications

**Don't use when:**
- There's only one consumer → use Tell directly
- You need guaranteed delivery → use persistent Ask
- Order matters → use direct communication

## Message Versioning

For messages that cross process/wire boundaries:

### Rule 1: Additive Only
```csharp
// V1:
public record OrderPlaced(Guid OrderId, decimal Total);

// V2: Add new field with default — backwards compatible
public record OrderPlaced(Guid OrderId, decimal Total, string? Currency = null);
```

### Rule 2: Never Remove Fields
```csharp
// WRONG: Removing Total breaks old consumers
public record OrderPlaced(Guid OrderId, string Currency);
```

### Rule 3: Never Rename Fields
```csharp
// WRONG: Renaming breaks serialization
public record OrderPlaced(Guid OrderId, decimal Amount);  // was: Total
```

### Rule 4: Use Correlation IDs for Request-Reply Over Brokers
```csharp
public record RequestWithCorrelation(
    Guid CorrelationId,
    string RequestTopic,
    string ResponseTopic,
    object Payload);
```

## Distributed Pub/Sub with Type Hierarchy (EventHub)

For cluster-wide event distribution where subscribers receive events based on type inheritance.

### Distributed Data Backing

Under the hood, subscriptions are stored in Akka.DistributedData (CRDT):

```csharp
// Each subscription is a LWWDictionary entry across the cluster
// Key: ActorRef → Value: List of subscribed type names
// WriteAll/ReadAll consistency for cluster-wide propagation

// This means:
// - Subscriptions survive node failures (replicated to all nodes)
// - New nodes discover existing subscriptions automatically
// - Eventual consistency — brief window where new subs may not be visible
```

### When to Use EventHub vs Direct Tell

| Scenario | EventHub | Direct Tell |
|---|---|---|
| Multiple unknown consumers | Yes | No |
| Cross-module communication | Yes | No |
| Audit/logging/metrics sidecars | Yes | No |
| Single known consumer | No | Yes |
| Guaranteed delivery required | No | Yes (with persistence) |
| Order matters | No | Yes |

---

## Design Checklist

Before finalizing any message design:

- [ ] Is it a record (immutable)?
- [ ] Is it correctly categorized (Command/Event/Query)?
- [ ] Is the name clear and follows conventions (verb/past-tense/get)?
- [ ] Does it contain all data needed by consumers?
- [ ] Are collections `IReadOnlyList<T>` or `IReadOnlyDictionary<K,V>`?
- [ ] If it crosses wire boundaries: is it additively versionable?
- [ ] If it needs routing: does it implement a shard/routing interface?
- [ ] Is there exactly one handler for commands, and possibly many for events?
