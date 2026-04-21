# IAspect Deprecation — Migrating Off Aspects
### Unity 6000.5 · Entities 6.5.0

---

## 1. Status — deprecated, not yet removed

`IAspect` was marked **obsolete in Entities 1.4** and remains obsolete on the 6.x line. It still compiles, still runs, and is still listed as a valid `IJobEntity.Execute` parameter kind on the 6.5 API reference — but the compiler raises a deprecation warning, and Unity has stated the type will be **removed in a future major release** (Entities 2.0). New code should not use it; existing code should migrate at an opportunistic pace.

`IAspect` was introduced to let systems declare a "view" over a set of components and use it as a single argument in `Entities.ForEach` or `IJobEntity`. In practice it added a layer of source generation and a parallel type system that didn't pay for itself:

- Writing an aspect duplicated the component parameter list.
- Debuggers stepped through generated wrapper code.
- `RefRW<T>` / `RefRO<T>` on `IJobEntity` and `SystemAPI.Query` achieve the same "pass components as a bundle" ergonomics without the extra type.

Because the surface is still present, your code does not break on the Editor upgrade — this migration is about cleaning up warnings and preparing for 2.0.

---

## 2. Mechanical refactor

### 2.1 Starting point

```csharp
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

public readonly partial struct MovementAspect : IAspect
{
    public readonly RefRW<LocalTransform> Transform;
    public readonly RefRO<Velocity>       Velocity;
    public readonly RefRO<Speed>          Speed;

    public void Step(float dt)
    {
        Transform.ValueRW.Position += Velocity.ValueRO.Direction * Speed.ValueRO.Value * dt;
    }
}

public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;
        foreach (var m in SystemAPI.Query<MovementAspect>())
            m.Step(dt);
    }
}
```

### 2.2 After — direct component parameters

```csharp
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;
        foreach (var (transform, velocity, speed) in
                 SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>, RefRO<Speed>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Direction * speed.ValueRO.Value * dt;
        }
    }
}
```

Two changes:

1. The aspect type is gone. Its fields become the tuple elements of the `SystemAPI.Query`.
2. The method that lived on the aspect (`Step`) is inlined. If it's too big to inline, extract a **static helper** that takes the same parameters — that keeps it Burst-friendly:

```csharp
static void Step(ref LocalTransform t, in Velocity v, in Speed s, float dt)
{
    t.Position += v.Direction * s.Value * dt;
}
```

Calling it:

```csharp
Step(ref transform.ValueRW, velocity.ValueRO, speed.ValueRO, dt);
```

---

## 3. Aspects in `IJobEntity`

### 3.1 Before

```csharp
[BurstCompile]
public partial struct MoveJob : IJobEntity
{
    public float DeltaTime;
    void Execute(MovementAspect aspect) => aspect.Step(DeltaTime);
}
```

### 3.2 After — components as Execute parameters

```csharp
[BurstCompile]
public partial struct MoveJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref LocalTransform transform, in Velocity velocity, in Speed speed)
    {
        transform.Position += velocity.Direction * speed.Value * DeltaTime;
    }
}
```

Note that `IJobEntity` uses plain `ref` / `in` parameters, not `RefRW<T>` / `RefRO<T>`. (The generator wraps the components into chunk-array accesses under the hood.)

---

## 4. Aspects that wrapped a DynamicBuffer

```csharp
// BEFORE
public readonly partial struct PathAspect : IAspect
{
    public readonly DynamicBuffer<Waypoint> Path;
    public void Advance(int n) => Path.RemoveRange(0, n);
}

// AFTER — access buffer directly
foreach (var (path, e) in SystemAPI.Query<DynamicBuffer<Waypoint>>().WithEntityAccess())
{
    path.RemoveRange(0, 1);
}
```

In jobs, `DynamicBuffer<T>` is a valid `Execute` parameter — nothing extra needed.

---

## 5. Cross-entity aspects

Occasionally an aspect wrapped a `ComponentLookup<T>` for "I need to peek at another entity's component." Replacement:

```csharp
// BEFORE
public readonly partial struct AITargetAspect : IAspect
{
    public readonly RefRO<Target> Target;
    [ReadOnly] public readonly ComponentLookup<Health> HealthLookup;
    public float TargetHealth =>
        HealthLookup.TryGetComponent(Target.ValueRO.Entity, out var h) ? h.Value : 0f;
}

// AFTER — declare the lookup on the system / job directly
public partial struct AISystem : ISystem
{
    private ComponentLookup<Health> _healthLookup;

    public void OnCreate(ref SystemState state)
    {
        _healthLookup = state.GetComponentLookup<Health>(true);
    }

    public void OnUpdate(ref SystemState state)
    {
        _healthLookup.Update(ref state);

        foreach (var (target, e) in SystemAPI
                     .Query<RefRO<Target>>()
                     .WithEntityAccess())
        {
            float hp = _healthLookup.TryGetComponent(target.ValueRO.Entity, out var h)
                ? h.Value : 0f;
            // ...
        }
    }
}
```

This pattern is also lighter than the aspect version because the lookup is shared across entities.

---

## 6. When an aspect was complex

If the aspect had many fields, encapsulated state, or participated in non-trivial inheritance — that's a sign to consider a genuine refactor, not just a port. Often the code reads better as two systems that pass data through components, rather than one system with a god-aspect.

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Compile warning `IAspect is obsolete` | Expected on 6.5. Port as above to clear the warning; the code still runs until removal in a future major version. |
| `type or namespace IAspect not found` | You are on a version where `IAspect` has been removed (Entities 2.0+). Complete the port — no fallback remains. |
| Query tuple has too many types and doesn't compile | `SystemAPI.Query` supports a large tuple; splitting a very wide query into two queries or moving to `IJobChunk` usually reads better. |
| Job no longer Burst-compiles | An aspect method may have captured a type Burst didn't like. Re-check the `Execute` parameters directly. |
| Cross-entity aspect ported naively runs slowly | `ComponentLookup<T>.Update(ref state)` must be called in `OnUpdate` before use; forgetting it makes lookups invalid. |
| DynamicBuffer aspect won't port | `DynamicBuffer<T>` is valid as a `SystemAPI.Query` type and `IJobEntity.Execute` parameter directly — no wrapper needed. |
| Tests that asserted on the aspect type fail | Delete or rewrite those tests against the new signature; there's nothing to keep. |
