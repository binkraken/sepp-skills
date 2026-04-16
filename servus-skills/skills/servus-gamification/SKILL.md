---
name: servus-gamification
description: Achievement system with Servus.Core — AchievementCollection, AchievementProperty with CompareRule, auto-reset, and unlock events
invocable: true
---

# Gamification with Servus.Core

Use when implementing an achievement or milestone system based on tracked properties with comparison rules.

## When to Use

- Tracking progress toward goals with numeric thresholds
- Unlocking achievements when multiple conditions are met simultaneously
- Building reward systems with auto-resettable counters
- Reacting to achievement unlocks via events

## API Reference

**Namespace:** `Servus.Core.Gamification`

### CompareRule

```csharp
public enum CompareRule
{
    Equals,
    GreaterThen,
    LowerThen,
    GreaterOrEquals,
    LowerOrEquals
}
```

### AchievementProperty

```csharp
public class AchievementProperty
{
    string Name { get; }
    double Value { get; set; }        // Current value
    double TargetValue { get; }       // Threshold
    CompareRule CompareRule { get; }
    bool AutoReset { get; set; }      // Reset Value after activation
    bool IsActive { get; }            // True when condition is met

    event Action? Activated;
}
```

### Achievement

```csharp
public class Achievement
{
    string Name { get; }
    bool IsUnlocked { get; }          // True when ALL properties are active

    event Action? Unlocked;
}
```

### AchievementCollection

```csharp
public class AchievementCollection
{
    IEnumerable<Achievement> Achievements { get; }

    // Define tracked properties
    void AddProperty(string name, double initialValue, double targetValue,
        CompareRule compareRule, bool autoReset = false);

    // Create an achievement linked to one or more properties
    void AddAchievement(string name, params string[] propertyNames);

    // Update a property value — triggers evaluation
    void SetValue(string propertyName, double value);

    // Fired when an achievement unlocks (sends achievement name)
    event Action<string>? AchievementUnlocked;
}
```

Achievements are removed from the collection after unlocking (one-time unlock).

## Usage Pattern

### Basic achievement setup

```csharp
var achievements = new AchievementCollection();

// Define properties with thresholds
achievements.AddProperty("orders_completed", 0, 10, CompareRule.GreaterOrEquals);
achievements.AddProperty("revenue", 0, 1000, CompareRule.GreaterOrEquals);

// Achievement unlocks when BOTH conditions are met
achievements.AddAchievement("power_seller", "orders_completed", "revenue");

// React to unlocks
achievements.AchievementUnlocked += name =>
    Console.WriteLine($"Achievement unlocked: {name}!");

// Update values as events happen
achievements.SetValue("orders_completed", 10);
achievements.SetValue("revenue", 1500);
// → "Achievement unlocked: power_seller!"
```

### Auto-reset for repeatable milestones

```csharp
achievements.AddProperty("daily_logins", 0, 5, CompareRule.GreaterOrEquals, autoReset: true);
achievements.AddAchievement("login_streak", "daily_logins");

// Value resets to initial after activation — can be triggered again
achievements.SetValue("daily_logins", 5); // Activates and resets
achievements.SetValue("daily_logins", 5); // Can activate again
```

### Multiple independent achievements

```csharp
achievements.AddProperty("kills", 0, 100, CompareRule.GreaterOrEquals);
achievements.AddProperty("deaths", 0, 0, CompareRule.Equals);
achievements.AddProperty("playtime_hours", 0, 50, CompareRule.GreaterOrEquals);

// Each achievement tracks its own subset of properties
achievements.AddAchievement("centurion", "kills");
achievements.AddAchievement("immortal", "kills", "deaths");
achievements.AddAchievement("dedicated", "playtime_hours");
```

### Listening to individual property activation

```csharp
var prop = new AchievementProperty("score", 0, 100, CompareRule.GreaterOrEquals);
prop.Activated += () => Console.WriteLine("Score threshold reached!");
prop.Value = 100;
```

## Common Mistakes

- **Expecting achievements to be re-lockable**: Achievements are removed from the collection after unlocking. For repeatable achievements, use `autoReset: true` on properties.
- **Setting properties before adding achievements**: Properties must be created before achievements that reference them. `AddAchievement` links to existing property names.
- **Using `Equals` with floating-point values**: Double comparison with `CompareRule.Equals` can be unreliable due to floating-point precision. Prefer `GreaterOrEquals` / `LowerOrEquals` for thresholds.
- **Forgetting that ALL properties must be active**: An achievement unlocks only when every linked property is simultaneously active. If one resets before another activates, the achievement stays locked.
