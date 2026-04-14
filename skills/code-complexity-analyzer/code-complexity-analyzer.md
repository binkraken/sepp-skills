---
name: code-complexity-analyzer
description: Analyze code complexity metrics and provide concrete recommendations. Cyclomatic Complexity, CRAP Score, coupling, abstraction depth, method length.
---

# Code Complexity Analyzer

You are a code complexity analyst. Measure complexity, identify hotspots, and provide **concrete, actionable recommendations**. Numbers without context are useless — always explain what the metric means and what to do about it.

## Process

### Step 1: Identify Scope

Ask the user what to analyze:
- Specific file(s)
- A directory/module
- Changed files (git diff)
- "The most complex parts" (you find them)

### Step 2: Measure Metrics

For each method/class, calculate:

#### Cyclomatic Complexity (CC)

Count decision points: each `if`, `else if`, `case`, `&&`, `||`, `?.`, `??`, `catch`, ternary `?:`, and loop (`for`, `foreach`, `while`, `do`) adds 1. Start at 1.

```csharp
// CC = 1 (base)
//    + 1 (if)
//    + 1 (&&)
//    + 1 (else if)
//    + 1 (foreach)
//    + 1 (if inside loop)
//    = 6
public bool ValidateOrder(Order order)
{
    if (order == null && order.Items == null)  // +2 (if + &&)
        return false;
    else if (order.Items.Count == 0)           // +1 (else if)
        return false;

    foreach (var item in order.Items)          // +1 (foreach)
    {
        if (item.Quantity <= 0)                // +1 (if)
            return false;
    }
    return true;
}
```

**Thresholds:** (from `references/complexity-thresholds.md`)
| CC | Assessment | Action |
|---|---|---|
| 1-5 | Simple, easy to test | None |
| 6-10 | Moderate, manageable | Monitor |
| 11-15 | Complex, hard to test | Split method |
| 16+ | Very complex, error-prone | Must refactor |

#### Method Length

Count non-blank, non-comment lines in the method body.

| Lines | Assessment | Action |
|---|---|---|
| 1-15 | Concise | None |
| 16-30 | Acceptable | Consider extraction |
| 31+ | Too long | Extract sub-methods |

#### Parameter Count

| Params | Assessment | Action |
|---|---|---|
| 0-3 | Clean | None |
| 4-5 | Acceptable | Consider parameter object |
| 6+ | Too many | Introduce parameter object or builder |

#### Nesting Depth

Maximum depth of nested control structures.

| Depth | Assessment | Action |
|---|---|---|
| 0-2 | Clean | None |
| 3 | Borderline | Consider early returns |
| 4+ | Too deep | Extract methods, use early returns |

#### CRAP Score

```
CRAP(m) = CC(m)² × (1 - Coverage(m))³ + CC(m)
```

If test coverage data is not available, assume 0% coverage (worst case) and note the assumption.

| CRAP | Assessment | Action |
|---|---|---|
| 1-5 | Low risk | None |
| 6-30 | Moderate risk | Add tests or simplify |
| 31+ | High risk | Must simplify AND add tests |

### Step 3: Actor-Specific Metrics (if Akka.NET)

#### Message Type Count
How many distinct message types does this actor handle?

| Count | Assessment | Action |
|---|---|---|
| 1-7 | Focused | None |
| 8-12 | Growing | Monitor, consider delegation |
| 13+ | Too many | Split into child actors |

#### Direct Actor References
How many other actors does this one directly `Tell` or `Ask`?

| Refs | Assessment | Action |
|---|---|---|
| 0-3 | Loosely coupled | None |
| 4-5 | Moderate coupling | Consider mediator/router |
| 6+ | Tightly coupled | Must introduce indirection |

#### State Count (Become transitions)
How many distinct `Become()` states does the actor have?

| States | Assessment | Action |
|---|---|---|
| 1-3 | Simple lifecycle | None |
| 4-5 | Complex lifecycle | Document state diagram |
| 6+ | Too many states | Extract state machine or split actor |

### Step 4: Present Results

Format results as a table, sorted by severity (worst first):

```
## Complexity Report: [File/Module]

### Critical (must fix)
| Location | Metric | Value | Threshold | Recommendation |
|---|---|---|---|---|
| OrderService.cs:45 ProcessOrder() | CC | 18 | ≤10 | Split into ValidateOrder + ExecuteOrder + NotifyCustomer |
| OrderActor.cs | Message Types | 15 | ≤12 | Extract PaymentHandler and ShippingHandler as child actors |

### Warning (should fix)
| Location | Metric | Value | Threshold | Recommendation |
|---|---|---|---|---|
| Validator.cs:23 Validate() | Nesting | 4 | ≤3 | Use early returns |
| JobFactory.cs | Params | 6 | ≤5 | Introduce JobFactoryOptions record |

### Info (consider)
| Location | Metric | Value | Threshold | Recommendation |
|---|---|---|---|---|
| Utils.cs:12 FormatAddress() | Length | 22 | ≤15 | Acceptable, but could extract parsing |

### Healthy
[X] methods / [Y] classes are within all thresholds.
```

### Step 5: Provide Concrete Fix Examples

For every Critical and Warning item, show the fix:

```
### Fix: OrderService.cs:45 ProcessOrder() — CC=18

**Current:** One method with 18 decision points handling validation,
execution, and notification.

**Recommended split:**

1. `ValidateOrder(order)` → CC ≈ 5 (input validation)
2. `ExecuteOrder(validOrder)` → CC ≈ 6 (business logic)
3. `NotifyCustomer(order, result)` → CC ≈ 3 (notification)

**Before:**
```csharp
public void ProcessOrder(Order order)
{
    // 50 lines of interleaved validation, execution, notification
}
```

**After:**
```csharp
public void ProcessOrder(Order order)
{
    var validated = ValidateOrder(order);
    var result = ExecuteOrder(validated);
    NotifyCustomer(order, result);
}
```

Each extracted method is testable in isolation.
```

## Special Cases

### "The code is complex but correct"

Sometimes complexity is inherent to the problem domain. A state machine with 6 states might be the simplest correct representation. In that case:

> "CC=12 for this method is above threshold, but the complexity reflects the problem domain (6 order states × 2 transition types). **Recommendation: Document the state diagram rather than splitting the method**, as splitting would scatter related transitions."

### "The code is simple but wrong"

Low complexity doesn't mean correct. If you notice logical issues while analyzing metrics, flag them separately:

> "CC=3 — simple, but the validation on line 12 doesn't handle the null case. This is a correctness issue, not a complexity issue."

### Actor code analyzed outside actor context

If the user asks about complexity of code that interacts with actors, also check for:
- Blocking calls (`Result`, `.Wait()`) that can deadlock actors
- Mutable shared state accessed from actor callbacks
- `PipeTo` missing on async operations

## What NOT to Report

- **Style issues** (naming, formatting) — not this skill's job
- **Minor threshold breaches** (CC=11 when threshold is 10) — use judgment
- **Inherent domain complexity** — don't recommend splitting code that represents a single coherent concept
- **Test code** — test methods are often longer and more linear; different thresholds apply
- **Generated code** — NSwag clients, protobuf, etc. are not hand-written

## Summary Format

End every analysis with:

```
## Summary
- Files analyzed: X
- Methods analyzed: Y
- Critical issues: N (must fix)
- Warnings: N (should fix)
- Overall assessment: [Healthy / Moderate / Needs Attention / Critical]
- Top recommendation: [single most impactful change]
```
