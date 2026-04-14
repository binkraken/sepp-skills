---
name: domain-modeling-patterns
description: Design domain models with value objects, record hierarchies, complex type parsers, and attribute-driven discovery. Make illegal states unrepresentable.
invocable: true
---

# Domain Modeling Patterns

You are a domain modeling architect. Design types that **make illegal states unrepresentable**. The compiler should catch domain errors, not runtime validation.

## Core Principles

1. **Primitive obsession is the enemy** — Wrap meaningful data in value objects
2. **Records over classes** — Immutable by default, structural equality, `with` expressions
3. **Sealed hierarchies** — Closed set of types = exhaustive pattern matching
4. **Parse, don't validate** — Convert untyped input into typed domain objects at the boundary

---

## Pattern 1: Value Objects

### Rich Value Object (with behavior)

```csharp
public record Money
{
    public decimal Amount { get; init; }
    public string Currency { get; init; } = "USD";

    public static readonly Money Zero = new() { Amount = 0 };

    public static Money Parse(string input)
    {
        // "49.99 USD" → Money { Amount = 49.99, Currency = "USD" }
        var parts = input.Split(' ');
        return new Money
        {
            Amount = decimal.Parse(parts[0]),
            Currency = parts.Length > 1 ? parts[1] : "USD"
        };
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot add {Currency} and {other.Currency}");
        return this with { Amount = Amount + other.Amount };
    }

    public bool IsPositive => Amount > 0;
    public bool IsZero => Amount == 0;

    public override string ToString() => $"{Amount:F2} {Currency}";
}
```

### When to Create a Value Object

| Signal | Action |
|---|---|
| Same primitive used with different meanings | `CustomerId(Guid)` vs `OrderId(Guid)` |
| Primitive has validation rules | `Email(string)` with format check |
| Primitive has formatting logic | `Money` with `ToString()` / `Parse()` |
| Group of primitives always travel together | `Dimensions(Width, Height, Depth)` |
| "Stringly typed" parameters | Replace `string address` with `Address` |

### When NOT to

- The primitive IS the concept (loop counter, boolean flag)
- Only used in one place with no validation
- Internal implementation detail, not domain concept

---

## Pattern 2: Record Hierarchies (Sealed Discriminated Unions)

Model a closed set of variants using sealed abstract records.

```csharp
// Payment method can be one of these — nothing else
public abstract record PaymentMethod
{
    public static readonly PaymentMethod None = new NoPayment();
}

public sealed record NoPayment : PaymentMethod;
public sealed record CreditCard(string Last4, string Brand, int ExpiryMonth, int ExpiryYear) : PaymentMethod;
public sealed record BankTransfer(string Iban, string Bic) : PaymentMethod;
public sealed record DigitalWallet(string Provider, string AccountId) : PaymentMethod;

// Exhaustive pattern matching — compiler warns if you miss a case
public string Describe(PaymentMethod method) => method switch
{
    NoPayment => "No payment",
    CreditCard cc => $"{cc.Brand} ending in {cc.Last4}",
    BankTransfer bt => $"Bank transfer to {bt.Iban}",
    DigitalWallet dw => $"{dw.Provider} wallet",
    _ => throw new UnreachableException()
};
```

### Task/Step Hierarchy

```csharp
public abstract record PipelineStep
{
    public required int Index { get; init; }
    public required bool IsRequired { get; init; }
}

public sealed record FetchData(string Url, TimeSpan Timeout) : PipelineStep;
public sealed record TransformData(string TransformerId, string Format) : PipelineStep;
public sealed record ValidateResult(IReadOnlyList<string> Rules) : PipelineStep;
public sealed record PublishResult(string Topic) : PipelineStep;
public sealed record WaitForApproval(TimeSpan Deadline) : PipelineStep;
public sealed record Notify(string Channel, string Template) : PipelineStep;
```

### When to Use Hierarchies

- Finite set of known variants (instruction types, payload types, states)
- Different variants carry different data
- Switch/match must handle all cases

### When NOT to

- Open set (new types added by plugins) → use interface
- All variants have same data → use enum flag
- Only 2 variants → bool or nullable might be simpler

---

## Pattern 3: Parse, Don't Validate

Convert untyped input to typed domain objects at the system boundary. After parsing, the type guarantees validity.

```csharp
// BAD: Validate everywhere
public void ProcessOrder(string email, int quantity)
{
    if (!IsValidEmail(email)) throw ...  // Must check every time
    if (quantity <= 0) throw ...
    // Now use email and quantity — but what if caller forgot to validate?
}

// GOOD: Parse at boundary, use typed values everywhere
public readonly record struct Email
{
    public string Value { get; }

    private Email(string value) => Value = value;

    public static Email Parse(string input)
    {
        if (string.IsNullOrWhiteSpace(input) || !input.Contains('@'))
            throw new FormatException($"Invalid email: {input}");
        return new Email(input.Trim().ToLowerInvariant());
    }

    public override string ToString() => Value;
}

public readonly record struct Quantity
{
    public int Value { get; }

    private Quantity(int value) => Value = value;

    public static Quantity Parse(int input)
    {
        if (input <= 0)
            throw new ArgumentOutOfRangeException(nameof(input), "Quantity must be positive");
        return new Quantity(input);
    }
}

// After parsing, no validation needed — the type IS the proof
public void ProcessOrder(Email email, Quantity quantity)
{
    // email is guaranteed valid, quantity is guaranteed positive
    // The compiler prevents passing raw strings/ints
}
```

---


## Pattern 4: Dimensional Modeling

For physical/spatial domains, model dimensions as proper types.

```csharp
public record Dimensions(double Width, double Height, double Depth)
{
    public double Volume => Width * Height * Depth;

    public bool FitsInside(Dimensions container)
        => Width <= container.Width
        && Height <= container.Height
        && Depth <= container.Depth;
}

public record BoxDimensions(Dimensions Inner, Dimensions Outer)
{
    public Dimensions WallThickness => new(
        (Outer.Width - Inner.Width) / 2,
        (Outer.Height - Inner.Height) / 2,
        (Outer.Depth - Inner.Depth) / 2);
}

public record Position(double X, double Y, double Z = 0)
{
    public double DistanceTo(Position other)
        => Math.Sqrt(
            Math.Pow(X - other.X, 2) +
            Math.Pow(Y - other.Y, 2) +
            Math.Pow(Z - other.Z, 2));
}
```

---

## Design Checklist

Before finalizing any domain model:

- [ ] Are IDs typed (not raw Guid/string)?
- [ ] Are records used (not mutable classes)?
- [ ] Are hierarchies sealed (exhaustive matching)?
- [ ] Is parsing at the boundary (not validation everywhere)?
- [ ] Are primitives that travel together grouped into value objects?
- [ ] Can the compiler prevent illegal states?
- [ ] Are collections `IReadOnlyList<T>` (not `List<T>`)?

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `string` for everything | No type safety, easy to mix up | Value objects |
| Mutable entity classes | Race conditions, identity confusion | Immutable records |
| Open class hierarchies | Can't exhaustively match | Sealed + abstract |
| Validation sprinkled everywhere | Easy to forget, duplicated logic | Parse at boundary |
| `null` for "no value" | NullReferenceException | `Option<T>` or sentinel value |
| Anemic domain model | Logic in services, not types | Put behavior on the type |
