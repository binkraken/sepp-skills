---
name: actor-pattern-library
description: Generate Akka.NET actor implementations following production-proven patterns. Persistent Actor, Cluster Singleton, Sharded Entity, Stashing Actor, Gateway/Proxy.
invocable: true
---

# Actor Pattern Library

You are an Akka.NET actor architect. Generate actor implementations using battle-tested patterns. Every template includes the pattern, anti-patterns to avoid, and a testing approach.

## Before You Start

Ask these questions to choose the right template:

1. **Does the actor need persistent state?** → Persistent Actor
2. **Must there be exactly one instance cluster-wide?** → Cluster Singleton
3. **Is there one instance per entity key (thousands)?** → Sharded Entity
4. **Does it need to wait for initialization before processing?** → Stashing Actor
5. **Does it bridge an external system (MQTT, gRPC, HTTP)?** → Gateway/Proxy

If none apply → the user probably doesn't need an actor. Suggest a plain class instead.

---

## Template 1: Persistent Actor

Use when an actor must survive restarts and recover its state from events.

```csharp
public class MyEntityActor : ReceivePersistentActor
{
    // --- Identity ---
    private readonly string _entityId;
    public override string PersistenceId => _entityId;

    // --- State ---
    private MyEntityState _state = MyEntityState.Empty;

    // --- Dependencies (via DI scope) ---
    private readonly IServiceScope _scope;
    private readonly IMyService _myService;

    public MyEntityActor(string entityId, IServiceProvider serviceProvider)
    {
        _entityId = entityId;
        _scope = serviceProvider.CreateScope();
        _myService = _scope.ServiceProvider.GetRequiredService<IMyService>();

        // Recovery: replay persisted events to rebuild state
        Recover<MyEvent>(evt => _state = _state.Apply(evt));
        Recover<SnapshotOffer>(offer =>
        {
            if (offer.Snapshot is MyEntityState snapshot)
                _state = snapshot;
        });

        // Commands: validate, then persist events
        Command<MyCommand>(HandleCommand);
    }

    private void HandleCommand(MyCommand cmd)
    {
        // Validate
        if (!IsValid(cmd))
        {
            Sender.Tell(new CommandRejected(cmd.Id, "Validation failed"));
            return;
        }

        // Create event
        var evt = new MyEvent(cmd.Id, cmd.Data);

        // Persist, then apply to state and notify
        Persist(evt, e =>
        {
            _state = _state.Apply(e);
            // Snapshot periodically
            if (LastSequenceNr % 100 == 0)
                SaveSnapshot(_state);
        });
    }

    protected override void PostStop()
    {
        _scope?.Dispose();
        base.PostStop();
    }
}
```

**State as pure function:**
```csharp
public record MyEntityState(string Name, int Count)
{
    public static readonly MyEntityState Empty = new("", 0);

    public MyEntityState Apply(MyEvent evt) => evt switch
    {
        NameChanged e => this with { Name = e.NewName },
        CountIncremented e => this with { Count = Count + e.Amount },
        _ => this
    };
}
```

**Anti-Patterns:**
- Never use `async/await` inside `Command<T>` handlers — use `PipeTo()` for async work
- Never mutate messages — use immutable records
- Never skip recovery handlers — the actor won't restore state after restart
- Never persist commands directly — persist domain events derived from commands

---

## Template 2: Cluster Singleton

Use when exactly one instance must run across the entire cluster.

```csharp
// Registration (in cluster setup):
builder.WithSingleton<MyCoordinator>(
    "my-coordinator",
    Props.Create(() => new MyCoordinator(/* deps */)),
    new ClusterSingletonOptions { Role = "backend" });

// The actor itself — regular ReceiveActor, nothing special:
public class MyCoordinator : ReceiveActor
{
    public MyCoordinator(IServiceProvider serviceProvider)
    {
        var scope = serviceProvider.CreateScope();
        // ... setup ...

        Receive<CoordinateWork>(msg =>
        {
            // Only one instance handles this cluster-wide
        });
    }
}

// Accessing from other nodes via proxy:
var coordinator = Context.GetRegistry().Get<MyCoordinator>();
coordinator.Tell(new CoordinateWork(...));
```

**When to use Singleton vs Sharding:**
- Singleton: Global coordination, rate limiting, leader election
- Sharding: Per-entity state, high throughput, horizontal scaling

**Anti-Patterns:**
- Never put high-throughput work in a singleton — it's a bottleneck by design
- Never assume the singleton is always available — it migrates during failover
- Never store singleton-specific state without persistence — it's lost on migration

---

## Template 3: Sharded Entity

Use when you need one actor per entity key, distributed across cluster nodes.

```csharp
// Registration:
builder.AddAkkaService<MyEntityActor, string>(
    entityIdExtractor: msg => msg switch
    {
        IEntityMessage e => (e.EntityId, msg),
        _ => throw new ArgumentException($"Unknown message type: {msg.GetType()}")
    },
    shardIdExtractor: msg => msg switch
    {
        IEntityMessage e => (Math.Abs(e.EntityId.GetHashCode() % 100).ToString(), msg),
        _ => throw new ArgumentException($"Unknown message type: {msg.GetType()}")
    });

// Entity message interface:
public interface IEntityMessage
{
    string EntityId { get; }
}

// Envelope for routing:
public record EntityEnvelope(string EntityId, object Message) : IEntityMessage;
```

**Anti-Patterns:**
- Never use sequential entity IDs for shard extraction — causes hot shards
- Never create too many shards (keep it 10× the max node count)
- Never forget passivation — idle actors consume memory forever

---

## Template 4: Stashing Actor

Use when the actor must initialize before it can process messages.

```csharp
public class MyGateway : ReceiveActor, IWithUnboundedStash
{
    public IStash Stash { get; set; } = null!;

    public MyGateway(IServiceProvider serviceProvider)
    {
        // State 1: Initializing — stash everything except init results
        Receive<InitResult>(result =>
        {
            if (result.Success)
            {
                Become(Ready);
                return;
            }
            // Init failed — stop or retry
            Self.Tell(PoisonPill.Instance);
        });

        ReceiveAny(_ => Stash.Stash());  // Queue all other messages
    }

    private void Ready()
    {
        // State 2: Ready — process stashed messages first
        Stash.UnstashAll();

        Receive<DoWork>(msg =>
        {
            // Process normally
        });
    }

    protected override void PreStart()
    {
        // Kick off async initialization
        InitializeAsync()
            .PipeTo(Self,
                success: result => new InitResult(true),
                failure: ex => new InitResult(false));
    }

    private record InitResult(bool Success);
}
```

**Key Points:**
- Use `IWithUnboundedStash` (not `IWithStash`) unless you need a bounded mailbox
- Always `UnstashAll()` when transitioning to ready state
- Always handle init failure (don't leave actor in limbo)
- Consider a timeout for init — don't stash forever

**Anti-Patterns:**
- Never stash without a transition to unstash — messages are lost
- Never use `IWithStash` with high-throughput actors — bounded stash drops messages
- Never forget to set `Stash` property — it's injected, not constructed

---

## Template 5: Gateway/Proxy

Use when bridging an external system (MQTT broker, gRPC service, WebSocket).

```csharp
public class ExternalGateway : ReceiveActor, IWithUnboundedStash
{
    public IStash Stash { get; set; } = null!;

    private readonly IExternalClient _client;
    private readonly IActorRef _eventPublisher;

    public ExternalGateway(IServiceProvider serviceProvider)
    {
        _client = serviceProvider.GetRequiredService<IExternalClient>();
        _eventPublisher = Context.GetRegistry().Get<EventPublisher>();

        // Disconnected state
        Receive<ConnectionResult>(msg =>
        {
            if (msg.Success) { Become(Connected); return; }
            Log.Warning("Connection failed, scheduling retry...");
            Context.System.Scheduler.ScheduleTellOnce(
                TimeSpan.FromSeconds(5), Self, new Reconnect(), Self);
        });

        Receive<Reconnect>(_ => Connect());
        ReceiveAny(_ => Stash.Stash());
    }

    private void Connected()
    {
        Log.Information("Gateway connected and ready");
        Stash.UnstashAll();

        // Outbound: actor messages → external system
        Receive<SendToExternal>(msg =>
        {
            _client.PublishAsync(msg.Topic, msg.Payload)
                .PipeTo(Self,
                    success: _ => new SendAck(msg.CorrelationId),
                    failure: ex => new SendFailed(msg.CorrelationId, ex));
        });

        // Inbound: external system → actor messages
        Receive<ExternalEvent>(msg =>
        {
            _eventPublisher.Tell(msg);
        });

        // Connection lost
        Receive<ConnectionLost>(_ =>
        {
            Log.Warning("Connection lost, reconnecting...");
            Become(Disconnected);
            Connect();
        });

        Receive<SendAck>(_ => { }); // Acknowledged
        Receive<SendFailed>(msg =>
            Log.Error(msg.Exception, "Failed to send {Id}", msg.CorrelationId));
    }

    private void Disconnected()
    {
        Receive<ConnectionResult>(msg =>
        {
            if (msg.Success) { Become(Connected); return; }
            Context.System.Scheduler.ScheduleTellOnce(
                TimeSpan.FromSeconds(5), Self, new Reconnect(), Self);
        });
        Receive<Reconnect>(_ => Connect());
        ReceiveAny(_ => Stash.Stash());
    }

    private void Connect()
    {
        _client.ConnectAsync()
            .PipeTo(Self,
                success: _ => new ConnectionResult(true),
                failure: _ => new ConnectionResult(false));
    }

    protected override void PreStart() => Connect();

    private record ConnectionResult(bool Success);
    private record Reconnect;
    private record SendAck(string CorrelationId);
    private record SendFailed(string CorrelationId, Exception Exception);
}
```

**Key Points:**
- Always handle disconnection and reconnection
- Use `PipeTo()` for all async operations
- Stash messages during disconnected state
- Use correlation IDs for tracking request/response
- Log connection state changes at Information level

**Anti-Patterns:**
- Never `await` inside Receive — use `PipeTo()` exclusively
- Never ignore connection failures — implement reconnection with backoff
- Never let the gateway become a god actor — keep it focused on communication

---

## Template 6: Request-Response with Timeout

Use when you need fire-and-retry: send a request, wait for response, retry on timeout.

```csharp
public class RequestHandlerActor : ReceiveActor
{
    private sealed record RequestTimedOut;

    public RequestHandlerActor(
        Action<IActorRef> sendRequest,
        int timeoutSeconds = 30)
    {
        // Send the request immediately, passing Self as reply-to
        sendRequest(Self);

        // Schedule periodic timeout checks
        var cancelable = Context.System.Scheduler
            .ScheduleTellRepeatedlyCancelable(
                TimeSpan.FromSeconds(timeoutSeconds),
                TimeSpan.FromSeconds(timeoutSeconds),
                Self, new RequestTimedOut(), Self);

        ReceiveAny(msg =>
        {
            if (msg is not Status.Failure and not RequestTimedOut)
            {
                // Success: forward response to parent and stop
                Context.Parent.Tell(msg);
                cancelable.Cancel();
                Context.Stop(Self);
            }
            else
            {
                // Timeout or failure: retry the request
                sendRequest(Self);
            }
        });
    }
}

// Usage in parent actor:
Context.ActorOf(Props.Create(() =>
    new RequestHandlerActor(
        replyTo => targetActor.Tell(new GetState(entityId), replyTo),
        timeoutSeconds: 10)));
```

**When to use:**
- Request-response across unreliable channels
- Fire-and-forget with automatic retry
- Short-lived child actor per request

**Anti-Patterns:**
- Don't use for long-running operations (use persistent actors instead)
- Don't forget to handle the final response in the parent
- Set reasonable timeout — too short causes message storms

---

## Template 7: Leader-Only Router with Selective Stashing

Use when messages should only be processed by the cluster leader for a specific role. Stashes messages during leader transitions.

```csharp
public class LeaderOnlyRouter : ReceiveActor, IWithUnboundedStash
{
    public IStash Stash { get; set; } = null!;

    public record LeaderOnlyMessage(string Role, string ActorName, object Message);

    private readonly Dictionary<string, Address?> _leaders = new();

    public LeaderOnlyRouter()
    {
        var cluster = Cluster.Get(Context.System);
        cluster.Subscribe(Self,
            ClusterEvent.SubscriptionInitialStateMode.InitialStateAsEvents,
            typeof(ClusterEvent.RoleLeaderChanged));

        Receive<ClusterEvent.RoleLeaderChanged>(changed =>
        {
            _leaders[changed.Role] = changed.Leader;

            // Selective unstash: only messages for the role that just got a leader
            Stash.UnstashAll(envelope =>
                envelope.Message is LeaderOnlyMessage msg && msg.Role == changed.Role);
        });

        Receive<LeaderOnlyMessage>(msg =>
        {
            if (_leaders.TryGetValue(msg.Role, out var leader) && leader != null)
            {
                var path = $"{leader}/user/{msg.ActorName}";
                Context.ActorSelection(path).Tell(msg.Message, Sender);
            }
            else
            {
                // No leader yet — stash until leader is elected
                Stash.Stash();
            }
        });
    }
}

// Usage:
var router = system.ActorOf(Props.Create<LeaderOnlyRouter>(), "leader-router");
router.Tell(new LeaderOnlyRouter.LeaderOnlyMessage("backend", "coordinator", payload));
```

**When to use:**
- Coordination tasks that must run on the leader node
- Cluster-wide decisions (scheduling, resource allocation)
- Leader-only writes with follower reads

**Key insight:** `UnstashAll` with a predicate enables **selective unstashing** — only messages for the newly elected role get processed, others stay stashed.

---

## Testing Approach

For every actor template, test with `Akka.TestKit.Xunit2`:

```csharp
public class MyEntityActorTests : TestKit
{
    [Fact]
    public void Should_handle_command_and_persist_event()
    {
        // Arrange
        var actor = Sys.ActorOf(Props.Create(() =>
            new MyEntityActor("test-1", BuildServiceProvider())));

        // Act
        actor.Tell(new MyCommand("test-1", "data"));

        // Assert
        ExpectMsg<CommandAccepted>(msg => msg.Id == "test-1");
    }

    [Fact]
    public void Should_stash_messages_during_init()
    {
        var actor = Sys.ActorOf(Props.Create(() =>
            new MyGateway(BuildServiceProvider())));

        // Send message before init completes
        actor.Tell(new DoWork("test"));

        // Complete init
        actor.Tell(new InitResult(true));

        // Stashed message should now be processed
        ExpectMsg<WorkDone>();
    }
}
```
