# Entities.ForEach → IJobEntity Migration Guide

The `Entities.ForEach` API has been deprecated (Obsolete) as of 1.4.  
Replace it with **`IJobEntity` + `ScheduleParallel()`** to keep the same performance  
while ensuring compatibility with future versions.

---

## 1. Legacy Code Example (SystemBase + Entities.ForEach)

```csharp
protected override void OnUpdate()
{
    float dt = SystemAPI.Time.DeltaTime;

    Entities
        .WithName("MoveSystem")
        .WithNone<Paused>()
        .ForEach((ref LocalTransform t, in Velocity v) =>
        {
            t.Position += v.Value * dt;
        })
        .ScheduleParallel();
}
```

---

## 2. Updated Code (IJobEntity)

```csharp
using Unity.Burst;
using Unity.Entities;

[BurstCompile]
public partial struct MoveJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref LocalTransform t, in Velocity v)
    {
        t.Position += v.Value * DeltaTime;
    }
}

protected override void OnUpdate()
{
    new MoveJob
    {
        DeltaTime = SystemAPI.Time.DeltaTime
    }.ScheduleParallel();
}
```

### Key Changes
| Point | Description |
|--------|------|
| **Lambda → Execute** | Move the lambda body into the `Execute` method |
| **[BurstCompile]** | Burst-compile the IJobEntity itself |
| **Pass captured variables** | Pass `DeltaTime` as a `public float` field |
| **Filter chain** | Convert `.WithNone<Paused>()` into a `SystemAPI.Query` chain |

---

## 3. Converting Filter Chains: .WithNone<Paused>()

Filter chains used with the legacy `Entities.ForEach` can be converted to `SystemAPI.Query`.

### 3-1 Legacy filter chain (SystemBase)
```csharp
Entities
    .WithName("MoveSystem")
    .WithNone<Paused>()
    .WithAll<Active>()
    .ForEach((ref LocalTransform t, in Velocity v) =>
    {
        t.Position += v.Value * dt;
    })
    .ScheduleParallel();
```

### 3-2 Conversion using SystemAPI.Query
```csharp
protected override void OnUpdate()
{
    var query = SystemAPI.QueryBuilder()
        .WithAll<LocalTransform, Velocity, Active>()
        .WithNone<Paused>()
        .Build();

    new MoveJob
    {
        DeltaTime = SystemAPI.Time.DeltaTime
    }.ScheduleParallel(query);
}
```

### 3-3 Built-in IJobEntity filtering (recommended)
```csharp
[BurstCompile]
public partial struct MoveJob : IJobEntity
{
    public float DeltaTime;

    // IJobEntity handles filtering automatically via the Execute method parameters
    void Execute(ref LocalTransform t, in Velocity v, 
                 in Active active)  // Equivalent to WithAll<Active>
    {
        // WithNone<Paused> is filtered automatically by excluding it from the Execute parameters
        t.Position += v.Value * DeltaTime;
    }
}
```

| Filter Type | SystemBase | IJobEntity |
|----------|------------|------------|
| **WithAll<T>** | `.WithAll<T>()` | Include in Execute parameters |
| **WithNone<T>** | `.WithNone<T>()` | Exclude from Execute parameters |
| **WithAny<T>** | `.WithAny<T>()` | Use `SystemAPI.QueryBuilder()` |

---

## 4. Checklist

- [ ] Have you added the `[BurstCompile]` attribute?  
- [ ] Have you passed captured values (time, constants, etc.) as **job fields**?  
- [ ] Are there no remaining `Entities.ForEach` calls in the project?  
- [ ] Have you verified with the Profiler that performance is maintained or improved?  
