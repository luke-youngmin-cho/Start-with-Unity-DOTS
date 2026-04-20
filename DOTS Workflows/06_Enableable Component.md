# Enableable Component Usage
### Toggle state with zero cost using `IEnableableComponent`

> Enableable components provide a 1-bit mask feature that lets you toggle component state On/Off **without moving component data out of the Chunk**.
> They avoid the **Structural Change** caused by `AddComponent` / `RemoveComponent`, while cleanly preserving the same semantics as Tag Components.

---

## 1. Concept Summary

| Item | Description |
|------|-------------|
| **Interface** | `public struct MyFlag : IEnableableComponent { public int Value; }` |
| **Storage** | Data stays inside the Chunk; state is managed via an **Enable Mask** (bit array) |
| **Toggle cost** | 1-bit memory modification -> 0 StructuralChanges |
| **Compatible API** | `SystemAPI.Query<EnabledRefRW<MyFlag>>()`, `ComponentLookup<MyFlag>().SetEnabled(entity, bool)` |
| **Use cases** | Temporary deactivation / Debuff state / Visibility Toggle, etc. |

---

## 2. Declaration & Basic Usage

### 2-1 Component Declaration
```csharp
using Unity.Entities;

public struct Invisible : IEnableableComponent
{
    public float FadeTime;   // Data is preserved even when disabled
}
```

### 2-2 Adding on Entity Creation
```csharp
// Add in the enabled state with an initial value
entityManager.AddComponentData(entity, new Invisible { FadeTime = 0 });
```

### 2-3 Toggling Enabled / Disabled
```csharp
var lookup = SystemAPI.GetComponentLookup<Invisible>();
lookup.SetEnabled(entity, false);  // Disable
lookup.SetEnabled(entity, true);   // Re-enable
```

* **Important**: The `Invisible` data is preserved across toggles. No StructuralChange is triggered.

---

## 3. Query Patterns

| Pattern | Example Code | Description |
|---------|--------------|-------------|
| **Enabled components only** | `SystemAPI.Query<RefRW<Invisible>>()` | Default Query returns only **enabled** entities |
| **Include disabled** | `SystemAPI.Query<EnabledRefRW<Invisible>>()` | Use `EnabledRefRW` / `EnabledRefRO` |
| **Disabled only** | `.WithNone<Invisible>() .WithAllDisabled<Invisible>()` | `WithDisabled<>()` attribute |
| **Toggle-in-place** | `EnabledRefRW<T>.ValueRW = false;` | Modify the bit directly inside the job |

### 3-1 Example: Blink Toggle Over Time
```csharp
partial struct BlinkJob : IJobEntity
{
    public double Time;
    void Execute(EnabledRefRW<Invisible> flag)
    {
        flag.ValueRW = (Time % 1.0) < 0.5; // On/Off at 0.5-second intervals
    }
}
```

---

## 4. Use in Jobs

* `EnabledRefRW<T>` / `EnabledRefRO<T>` are Burst-friendly and read/modify the **Enable Mask** directly without additional memory copying.
* To avoid toggling the same entity concurrently from parallel jobs, split your queries or handle toggles on a single-threaded path.
n### 4-2 Concrete Example of Direct EnableMask Manipulation

#### Accessing EnableMask in IJobChunk
```csharp
[BurstCompile]
public struct EnableMaskProcessorJob : IJobChunk
{
    public ComponentTypeHandle<LocalTransform> TransformHandle;
    [ReadOnly] public ComponentTypeHandle<Invisible> InvisibleHandle;

    public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
    {
        var transforms = chunk.GetNativeArray(ref TransformHandle);
        
        // Direct access to the EnableMask
        var enableMask = chunk.GetEnabledMask(ref InvisibleHandle);
        
        // Iterate the bit mask - process only enabled entities
        for (int i = 0; i < chunk.Count; i++)
        {
            if (enableMask[i])  // O(1) bit access
            {
                var transform = transforms[i];
                transform.Scale = 0.5f;  // Semi-transparent effect
                transforms[i] = transform;
            }
        }
    }
}
```

#### Performance Comparison: EnableMask vs StructuralChange
```csharp
// Bad: StructuralChange approach (Chunk relocation)
foreach (var entity in entities)
{
    if (shouldHide)
        entityManager.AddComponent<Invisible>(entity);
    else
        entityManager.RemoveComponent<Invisible>(entity);
}

// Good: EnableMask approach (1-bit change)
foreach (var invisibleRef in SystemAPI.Query<EnabledRefRW<Invisible>>())
{
    invisibleRef.ValueRW = shouldHide;
}
```


---

## 5. Tag Component vs Enableable

| Comparison | Tag Component (`IComponentData` 0-byte) | Enableable (`IEnableableComponent`) |
|------------|------------------------------------------|-------------------------------------|
| **On/Off cost** | Add/Remove -> StructuralChange | Bit modification (O(1)) |
| **Data fields** | Cannot have any | Can have fields |
| **Memory** | 0 bytes + Chunk header | Field size + 1 bit |
| **GC overhead** | None | None |
| **Use case** | Rarely granted/removed state | Per-frame toggling or when values must persist |

---

## 6. Caveats & Best Practices

1. **Frequent toggling**: Even hundreds of toggles per second are safe since there are no StructuralChanges, but avoid **contention from multiple toggles on the same entity**.
2. **Initial Disabled**: Immediately after `AddComponent`, call `SetEnabled(false)` or set `EnabledRefRW.ValueRW = false` to configure the initial disabled state.
3. **Query performance**: `EnabledRef*` queries add a small mask check but the overhead compared to regular `Ref*` is negligible.
4. **Profiler check**: If there are no **Archetype move** events in the Entity Timeline, the toggle succeeded.
5. **Hybrid Renderer**: The renderer system respects the enabled state, so toggling Invisible can reduce draw calls immediately.

---

## 7. Hands-on Checklist

- [ ] Declare an Enableable component (`IEnableableComponent`).
- [ ] Implement toggling in a job using the `EnabledRefRW<T>` pattern.
- [ ] Verify in the Profiler that toggling occurs without StructuralChange.
- [ ] Can you explain the difference between Tag Components and Enableable to a teammate?

> Using Enableable components, you can implement **gameplay state toggling** or **temporary deactivation** logic at no cost, greatly reducing the total number of StructuralChanges and improving performance.
