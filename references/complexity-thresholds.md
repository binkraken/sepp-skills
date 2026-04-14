# Complexity Thresholds

Reference values for the code-complexity-analyzer skill. These are guidelines, not hard rules — context matters.

## Method-Level Metrics

| Metric | Good | Acceptable | Needs Attention | Action |
|---|---|---|---|---|
| Cyclomatic Complexity | 1-5 | 6-10 | 11-15 | Split method |
| Method Length (lines) | 1-15 | 16-30 | 31+ | Extract submethods |
| Parameter Count | 0-3 | 4-5 | 6+ | Introduce parameter object |
| Nesting Depth | 0-2 | 3 | 4+ | Early return / extract |
| CRAP Score | 1-5 | 6-30 | 31+ | Add tests or simplify |

## Class-Level Metrics

| Metric | Good | Acceptable | Needs Attention | Action |
|---|---|---|---|---|
| Methods per Class | 1-7 | 8-15 | 16+ | Split responsibilities |
| Lines per Class | 1-100 | 101-300 | 301+ | Extract collaborators |
| Dependencies (constructor params) | 0-3 | 4-5 | 6+ | Facade or mediator |
| Abstraction Depth (inheritance) | 0-1 | 2 | 3+ | Prefer composition |

## Actor-Specific Metrics

| Metric | Good | Acceptable | Needs Attention | Action |
|---|---|---|---|---|
| Direct Actor References | 0-3 | 4-5 | 6+ | Introduce router/mediator |
| States (Become transitions) | 1-3 | 4-5 | 6+ | State machine extraction |
| Message Types Handled | 1-7 | 8-12 | 13+ | Split into child actors |
| Child Actors Created | 0-3 | 4-5 | 6+ | Consider hierarchy refactor |

## CRAP Score Formula

```
CRAP(m) = CC(m)² × (1 - Coverage(m))³ + CC(m)
```

Where:
- `CC(m)` = Cyclomatic Complexity of method m
- `Coverage(m)` = Test coverage ratio (0.0 to 1.0)

**Interpretation:**
- CRAP ≤ 5: Low risk, well-tested
- CRAP 6-30: Moderate risk, consider improving coverage or simplifying
- CRAP > 30: High risk — must either simplify or add tests

## Complexity Budget

A useful mental model: each component has a **complexity budget**. Spending it on the right things matters more than the absolute number.

**Worth the complexity:**
- Pattern solves a real concurrency/distribution problem
- Abstraction used by 3+ consumers
- Error handling at system boundaries
- Type safety that prevents runtime errors

**Not worth the complexity:**
- Abstraction for one use case
- Interface with one implementation (no test justification)
- Pattern where 5 lines of direct code would work
- Wrapper that adds indirection but no value
- Generic where concrete suffices

## The Simplicity Test

Before introducing any abstraction, answer these questions:

1. **Would a new team member understand this in under 2 minutes?**
2. **Does this abstraction have at least 3 consumers (or will it within this sprint)?**
3. **Does removing it make the code LONGER or HARDER to understand?**

If any answer is "no" → don't add the abstraction.
