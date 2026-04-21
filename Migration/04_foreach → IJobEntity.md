# foreach → IJobEntity Migration
### Unity 6000.5 · Entities 6.5.0

---

## 1. Status — obsolete, not yet removed

The legacy component-iteration APIs are **obsolete** on Entities 1.4+ / 6.5 but still compile:

| Legacy | Status in 6.5 | Replacement |
|--------|--------------|-------------|
| `Entities.ForEach` | **Obsolete** since 1.4; deprecation warnings on every use; scheduled for removal in the next major version (Entities 2.0) | `SystemAPI.Query<...>` (main thread) or `IJobEntity` (jobs) |
| `Job.WithCode` | **Obsolete** alongside ForEach | Plain `IJob` struct, or inline in `OnUpdate` |
| `IJobForEach` (pre-1.0) | Removed long before 1.4 | `IJobEntity` |

You're here either because you opened a 1.x project in 6000.5+ and the compiler is warning loudly, or because you want to clear these warnings before the next major version lands. The refactor is mechanical once you see the pattern.

---

## 2. The two replacement APIs

- **`SystemAPI.Query<T, U, ...>`** — main-thread iteration inside `OnUpdate`. Simple, no job overhead, ideal for small workloads or when you need managed APIs.
- **`IJobEntity`** — source-generated parallel job. Drop-in if you're ready for multi-threaded work, which is where DOTS pays off.

Background: [`../DOTS Workflows/11_IJobEntity · SystemAPI.Query.md`](../DOTS%20Workflows/11_IJobEntity%20%C2%B7%20SystemAPI.Query.md).

---

## 3. Port — `Entities.ForEach` → `SystemAPI.Query`

### 3.1 Before

```csharp
public partial class HealthRegenSystem : SystemBase
{
    protected override void OnUpdate()
    {
        float dt = Time.DeltaTime;

        Entities
            .WithAll<Alive>()
            .WithNone<Dead>()
            .ForEach((ref Health health, in Regen regen) =>
            {
                health.Value = math.min(health.Value + regen.Rate * dt, health.Max);
            })
            .ScheduleParallel();
    }
}
```

### 3.2 After — `SystemAPI.Query` (main thread)

```csharp
[BurstCompile]
public partial struct HealthRegenSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (health, regen) in
                 SystemAPI.Query<RefRW<Health>, RefRO<Regen>>()
                          .WithAll<Alive>()
                          .WithNone<Dead>())
        {
            health.ValueRW.Value = math.min(
                health.ValueRO.Value + regen.ValueRO.Rate * dt,
                health.ValueRO.Max);
        }
    }
}
```

Notes on the shape change:
- `SystemBase` → `ISystem` (cheaper, Burst-able). You may need to keep `SystemBase` if the old code captured managed fields.
- `ref Health` → `RefRW<Health>` tuple element. Read via `.ValueRO`, write via `.ValueRW`.
- `in Regen` → `RefRO<Regen>`.
- `Time.DeltaTime` → `SystemAPI.Time.DeltaTime`.
- Filters (`WithAll`/`WithNone`) move onto the `Query` builder.

---

## 4. Port — `Entities.ForEach.ScheduleParallel` → `IJobEntity`

When the ForEach was scheduled in parallel, the job form is a direct 1:1 port and keeps the parallelism.

### 4.1 After — `IJobEntity`

```csharp
[BurstCompile]
public partial struct HealthRegenJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref Health health, in Regen regen)
    {
        health.Value = math.min(health.Value + regen.Rate * DeltaTime, health.Max);
    }
}

[BurstCompile]
public partial struct HealthRegenSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        new HealthRegenJob { DeltaTime = SystemAPI.Time.DeltaTime }
            .ScheduleParallel();
    }
}
```

Filters go on the job struct as attributes:

```csharp
[WithAll(typeof(Alive))]
[WithNone(typeof(Dead))]
public partial struct HealthRegenJob : IJobEntity { ... }
```

Or pass a query at schedule time:

```csharp
var q = SystemAPI.QueryBuilder()
    .WithAll<Health, Regen, Alive>()
    .WithNone<Dead>()
    .Build();
new HealthRegenJob { DeltaTime = dt }.ScheduleParallel(q);
```

---

## 5. Captures and their replacements

Legacy `Entities.ForEach` allowed capturing locals with `.WithReadOnly`, `.WithDisposeOnCompletion`, etc. In `IJobEntity`, captures are **fields on the job struct** — Burst knows exactly what's being passed.

```csharp
// BEFORE
var buffer = new NativeArray<int>(count, Allocator.TempJob);
Entities.ForEach((ref Foo f) => { /* use buffer */ })
    .WithDisposeOnCompletion(buffer)
    .ScheduleParallel();

// AFTER
[BurstCompile]
partial struct FooJob : IJobEntity
{
    [ReadOnly] public NativeArray<int> Buffer;

    void Execute(ref Foo f) { /* use Buffer */ }
}

var buffer = new NativeArray<int>(count, Allocator.TempJob);
state.Dependency = new FooJob { Buffer = buffer }.ScheduleParallel(state.Dependency);
buffer.Dispose(state.Dependency);   // chained dispose replaces WithDisposeOnCompletion
```

Attribute replacements:
- `.WithReadOnly(x)` → `[ReadOnly] public NativeArray<T> X;`
- `.WithDisposeOnCompletion(x)` → `x.Dispose(handle)` after scheduling.
- `.WithoutBurst()` → `[WithoutBurst]` on the job struct (but strongly consider removing the reason you needed it).
- `.WithEntityAccess()` → add `Entity entity` parameter to `Execute`.

---

## 6. When a `SystemBase` is still required

If the original system had a genuine managed field (Input System, UI Toolkit, external service), you can't mechanically convert to `ISystem`. Two options:

1. **Keep it as `SystemBase`** and use `SystemAPI.Query` inside `OnUpdate`. Single-threaded.
2. **Split** into a `SystemBase` that bridges the managed world and an `ISystem` that does the Burst work — see [`../DOTS Workflows/08_System — ISystem vs SystemBase.md`](../DOTS%20Workflows/08_System%20%E2%80%94%20ISystem%20vs%20SystemBase.md).

---

## 7. Validation

After porting a system:

1. **Unit-test the behaviour.** Port should not change logic; if output differs, the refactor introduced a bug.
2. **Profile.** Parallel ForEach and `IJobEntity.ScheduleParallel` should be within 5% of each other for equivalent work. A large regression means Burst isn't kicking in — inspect the Burst Inspector.
3. **Delete the old system file.** Resist keeping a `#if LEGACY` fallback — it rots.

---

## 8. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `warning: Entities.ForEach is obsolete` | Expected on 6.5 — the code still compiles. Port as above to clear the warning before Entities 2.0 removes it. |
| `error: 'SystemAPI' does not contain 'Query'` | System is not `partial`, or the struct doesn't implement `ISystem` correctly. |
| `IJobEntity` compiles but doesn't run | Missing `[BurstCompile]` on the struct and on `OnUpdate`; source generator doesn't schedule non-partial types. |
| Job reads stale values | Captured a local before `.Schedule*` was called. Fields must be set on the job struct before scheduling. |
| Perf regresses after port | Check for Burst fallback (Burst Inspector), or a managed type sneaked into a field. |
| Compile-time only issues after porting | Cross-check that you removed captured locals via attributes and not via lambda-closure syntax, which only applied to the ForEach API. |
