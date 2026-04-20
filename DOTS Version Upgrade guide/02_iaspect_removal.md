# IAspect Removal Migration Guide – Switching to Direct EntityQuery Code

The `IAspect` API is scheduled for deprecation in 1.4.  
Use the steps below to convert to a **direct `SystemAPI.Query` + `RefRW / RefRO`** pattern.

---

## 1. Legacy Aspect Code

```csharp
public readonly partial struct MovementAspect : IAspect
{
    public readonly RefRW<LocalTransform> Transform;
    public readonly RefRO<Velocity> Velocity;
}

partial struct MoveSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var aspect in SystemAPI.Query<MovementAspect>())
        {
            aspect.Transform.ValueRW.Position +=
                aspect.Velocity.ValueRO.Value * SystemAPI.Time.DeltaTime;
        }
    }
}
```

---

## 2. Converted Code

```csharp
public partial struct MoveSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (transform, velocity) in
                 SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * dt;
        }
    }
}
```

### Key Changes
| Item | Replacement |
|------|-----------|
| **Aspect fields** | Use `RefRW<T>`, `RefRO<T>` tuples directly |
| **Aspect helpers** | Move to `static` utility methods |
| **Queries** | Write using `SystemAPI.Query<…>` |
| **Test code** | Refactor Aspect unit tests into function-level tests |

---

## 3. Complex Aspect Conversion Example

### 3-1 An Aspect with multiple components and helper methods

#### Legacy code (IAspect)
```csharp
public readonly partial struct CombatAspect : IAspect
{
    public readonly RefRW<Health> Health;
    public readonly RefRW<LocalTransform> Transform;
    public readonly RefRO<Damage> Damage;
    public readonly RefRO<CombatStats> Stats;

    public bool IsDead => Health.ValueRO.Value <= 0;
    
    public void TakeDamage(float damage)
    {
        Health.ValueRW.Value = math.max(0, Health.ValueRO.Value - damage);
    }

    public void MoveTowards(float3 target, float speed)
    {
        var direction = math.normalize(target - Transform.ValueRO.Position);
        Transform.ValueRW.Position += direction * speed;
    }

    public float CalculateEffectiveDamage()
    {
        return Damage.ValueRO.Value * Stats.ValueRO.AttackMultiplier;
    }
}

// Usage site
partial struct CombatSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var combatAspect in SystemAPI.Query<CombatAspect>())
        {
            if (!combatAspect.IsDead)
            {
                combatAspect.TakeDamage(10f);
                combatAspect.MoveTowards(float3.zero, 2f);
                var effectiveDamage = combatAspect.CalculateEffectiveDamage();
            }
        }
    }
}
```

#### Converted code (Static Helper + Direct Query)
```csharp
// Extract helper methods into a static class
public static class CombatHelper
{
    public static bool IsDead(in Health health) => health.Value <= 0;
    
    public static void TakeDamage(ref Health health, float damage)
    {
        health.Value = math.max(0, health.Value - damage);
    }

    public static void MoveTowards(ref LocalTransform transform, float3 target, float speed)
    {
        var direction = math.normalize(target - transform.Position);
        transform.Position += direction * speed;
    }

    public static float CalculateEffectiveDamage(in Damage damage, in CombatStats stats)
    {
        return damage.Value * stats.AttackMultiplier;
    }
}

// Converted system
partial struct CombatSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (health, transform, damage, stats) in 
                 SystemAPI.Query<RefRW<Health>, RefRW<LocalTransform>, 
                               RefRO<Damage>, RefRO<CombatStats>>())
        {
            if (!CombatHelper.IsDead(health.ValueRO))
            {
                CombatHelper.TakeDamage(ref health.ValueRW, 10f);
                CombatHelper.MoveTowards(ref transform.ValueRW, float3.zero, 2f);
                var effectiveDamage = CombatHelper.CalculateEffectiveDamage(
                    damage.ValueRO, stats.ValueRO);
            }
        }
    }
}
```

### 3-2 Entity Filtering and Conditional Component Handling

#### Legacy code (IAspect + Conditional Components)
```csharp
public readonly partial struct PlayerAspect : IAspect
{
    public readonly Entity Entity;
    public readonly RefRW<LocalTransform> Transform;
    public readonly RefRO<PlayerInput> Input;
    public readonly RefRO<MovementSpeed> Speed;
    
    // Enableable component handling
    public readonly EnabledRefRO<Sprinting> SprintEnabled;
    
    public float GetCurrentSpeed()
    {
        float baseSpeed = Speed.ValueRO.Value;
        if (SprintEnabled.ValueRO)
            return baseSpeed * 2f;
        return baseSpeed;
    }
}
```

#### Converted code
```csharp
partial struct PlayerMovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Basic movement handling
        foreach (var (entity, transform, input, speed) in 
                 SystemAPI.Query<Entity, RefRW<LocalTransform>, 
                               RefRO<PlayerInput>, RefRO<MovementSpeed>>())
        {
            float currentSpeed = speed.ValueRO.Value;
            
            // Conditional handling of the Enableable component
            if (SystemAPI.IsComponentEnabled<Sprinting>(entity))
            {
                currentSpeed *= 2f;
            }
            
            var movement = input.ValueRO.Direction * currentSpeed * SystemAPI.Time.DeltaTime;
            transform.ValueRW.Position += movement;
        }
    }
}
```

### 3-3 Conversion Guidelines

| Aspect Feature | Conversion Approach | Example |
|-------------|-----------|------|
| **Property accessors** | `static` helper method | `IsDead => static bool IsDead(in Health)` |
| **Entity reference** | Add `Entity` to the query | `Query<Entity, RefRW<T>>()` |
| **EnabledRef** | Use `SystemAPI.IsComponentEnabled<T>()` | Branch on conditional logic |
| **Complex logic** | Standalone `static` utility class | `CombatHelper.CalculateDamage()` |
| **Test code** | Aspect → per-function unit tests | `Assert.IsTrue(CombatHelper.IsDead(...))` |

---

## 4. Checklist

- [ ] Have you deleted or commented out the `IAspect` files?  
- [ ] Have you converted all related queries to `SystemAPI.Query`?  
- [ ] Have you moved Aspect helper logic to `static` methods where needed?  
- [ ] Have you verified with the Profiler that there is no performance regression?  
