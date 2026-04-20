# Entities 1.4 — Deprecated API & Replacements Guide

## 1. Deprecated API Overview

| API | Status | Recommended Replacement |
|-----|------|-----------|
| `Entities.ForEach` | **Obsolete** (from 1.4) | `IJobEntity` or `SystemAPI.Query` |
| `Job.WithCode` | **Obsolete** | Write a dedicated `IJob` struct |
| `IAspect` and its source generators | **Obsolete** (scheduled for future removal) | Individual `Component` / `EntityQuery` APIs |
| `ComponentLookup.GetRefRWOptional`<br>`ComponentLookup.GetRefROOptional` | **Obsolete** | `TryGetRefRW` / `TryGetRefRO` |

> **Note**  
> These APIs still work in the 1.x versions, but will be **removed in the next major version**. Replacing them now will make future upgrades much smoother.

---

## 2. Representative Migration Examples

### 2-1 `Entities.ForEach` → `IJobEntity`

```csharp
// Before: SystemBase + Entities.ForEach
protected override void OnUpdate()
{
    float dt = SystemAPI.Time.DeltaTime;
    Entities
        .ForEach((ref LocalTransform t, in RotationSpeed s) =>
        {
            t.Rotation = math.mul(
                math.normalize(t.Rotation),
                quaternion.AxisAngle(math.up(), s.RadiansPerSecond * dt));
        })
        .ScheduleParallel();
}

// After: IJobEntity + Burst
[BurstCompile]
public partial struct RotateJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref LocalTransform t, in RotationSpeed s)
    {
        t.Rotation = math.mul(
            math.normalize(t.Rotation),
            quaternion.AxisAngle(math.up(), s.RadiansPerSecond * DeltaTime));
    }
}
protected override void OnUpdate()
{
    new RotateJob { DeltaTime = SystemAPI.Time.DeltaTime }
        .ScheduleParallel();
}
```

### 2-2 `ComponentLookup.GetRef*Optional` → `TryGetRef*`

```csharp
// Before
if (lookup.GetRefROOptional(entity, out var compRO))
{
    // ...
}

// After
if (lookup.TryGetRefRO(entity, out var compRO))
{
    // ...
}
```

`TryGetRef*` performs **existence checking and reference retrieval** in a single call, making the code more concise and safer.

### 2-3 Removing `IAspect`

* Fields previously grouped in an Aspect should be handled directly as **individual components** or  
  via the `SystemAPI.Query<RefRW<T>, …>` pattern.  
* If shared logic is needed, extract it into dedicated **static helper methods** to keep systems clean.

---

## 3. Migration Notes

1. **Performance** — `IJobEntity` / `SystemAPI.Query` are internally converted into `IJobChunk` code, giving better runtime and compile speed than `Entities.ForEach`.  
2. **Burst compilation** — `IJobEntity` has Burst disabled by default, so be sure to add the `[BurstCompile]` attribute.  
3. **Static analysis** — If Obsolete warnings remain after migration, use your IDE's search to find and fix any remaining calls in bulk.  
4. **Testing** — If you have modified structural-change logic, use the Profiler to check for **Sync Points** and prevent performance regressions.

---

## 4. Step-by-Step Migration Checklist

1. Search project-wide for calls to deprecated APIs  
2. Replace them using the patterns above and compile  
3. Apply `[BurstCompile]` and validate performance with the **Profiler**  
4. Inspect scheduling and dependencies via the `System Inspector → Dependencies` tab  
5. Review the package **Changelog / Upgrade Guide** for any additional changes
