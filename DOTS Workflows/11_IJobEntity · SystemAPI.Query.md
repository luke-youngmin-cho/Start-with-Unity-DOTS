# IJobEntity · SystemAPI.Query
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

Two APIs cover 95% of component iteration in Entities 6.5:

| API | Where it runs | Use when… |
|-----|---------------|-----------|
| `SystemAPI.Query<...>()` | Main thread, inside `OnUpdate` | The work is trivial (a few hundred entities), or needs main-thread APIs. |
| `IJobEntity` | Worker threads, via `Schedule / ScheduleParallel` | The work is heavier or benefits from parallelism. |

They are the recommended replacement for the legacy `Entities.ForEach` (marked obsolete in Entities 1.4; still compiles on 6.5 with deprecation warnings; removal planned for Entities 2.0).

---

## 2. `SystemAPI.Query<T>`

Iterate matching entities on the main thread. Components are accessed via `RefRO<T>` / `RefRW<T>`.

```csharp
[BurstCompile]
public partial struct HealthRegenSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (health, regen) in
                 SystemAPI.Query<RefRW<Health>, RefRO<Regen>>())
        {
            health.ValueRW.Value = math.min(
                health.ValueRO.Value + regen.ValueRO.Rate * dt,
                health.ValueRW.Max);
        }
    }
}
```

Rules:
- The system struct must be `partial`.
- `RefRO<T>` for read-only, `RefRW<T>` for read-write. Don't write through `RefRO`.
- Add an `Entity` to the tuple if you need the id: `SystemAPI.Query<RefRW<Health>>().WithEntityAccess()`.

### Filters

```csharp
foreach (var health in SystemAPI.Query<RefRW<Health>>()
             .WithAll<Enemy>()
             .WithNone<Dead>()
             .WithAny<Burning, Poisoned>())
{
    // only entities that are Enemy AND not Dead AND (Burning OR Poisoned)
}
```

- `WithAll<T>()` — must have all of the listed components.
- `WithNone<T>()` — must not have any of them.
- `WithAny<T...>()` — must have at least one of them.
- `WithChangeFilter<T>()` — only chunks where component `T` changed since last run.
- `WithEntityAccess()` — also return the `Entity` id.

---

## 3. `IJobEntity`

A source-generated parallel iteration. The `Execute` signature declares what components you touch.

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Mathematics;

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

Signature rules:
- `void Execute(...)` — parameters map 1:1 to components.
- `ref Component` — read-write. `in Component` — read-only (no write allowed).
- `Entity entity` as a parameter is allowed — the source generator fills it in.
- Job `struct` must be `partial`.

### Schedule variants

| Call | Threading |
|------|-----------|
| `.Run()` | Main thread, completes before returning. Useful for debugging or small workloads. |
| `.Schedule()` | Single worker thread. |
| `.ScheduleParallel()` | Distributes chunks across worker threads. |

All three update `state.Dependency` automatically inside an `ISystem.OnUpdate`.

### Filters in jobs

```csharp
new HealthRegenJob { DeltaTime = dt }
    .ScheduleParallel(SystemAPI.QueryBuilder()
        .WithAll<Enemy, Regen>()
        .WithNone<Dead>()
        .Build());
```

Or via `[WithAll]`/`[WithNone]`/`[WithAny]` attributes on the job struct itself.

### Entity access in a job

```csharp
void Execute(Entity entity, ref Health health) { /* ... */ }
```

### Chunk index / `[ChunkIndexInQuery]`

Useful when pairing with a `ParallelWriter` ECB for deterministic playback — pass the chunk index as the `sortKey`. See [`15_ParallelWriter · Deterministic Playback.md`](15_ParallelWriter%20%C2%B7%20Deterministic%20Playback.md).

```csharp
void Execute([ChunkIndexInQuery] int chunkIndex, in Health health, Entity entity)
{
    if (health.Value <= 0)
        ecb.DestroyEntity(chunkIndex, entity);
}
```

---

## 4. When to use which

| Situation | Pick |
|-----------|------|
| Tens to hundreds of entities, trivial math | `SystemAPI.Query` |
| Thousands+ of entities, independent per-entity work | `IJobEntity.ScheduleParallel` |
| Per-chunk work (bounds, sums, spatial partitioning) | **`IJobChunk`** — see [`12_IJobEntity vs IJobChunk.md`](12_IJobEntity%20vs%20IJobChunk.md) |
| Need managed API inside the loop | `SystemAPI.Query` from a `SystemBase` |
| Need deferred structural changes | Either form, but write via `EntityCommandBuffer.ParallelWriter` — see [`14_EntityCommandBuffer · Deferred Entity.md`](14_EntityCommandBuffer%20%C2%B7%20Deferred%20Entity.md) |

---

## 5. Gotchas

- **Don't capture `this` or managed fields in a job.** The source generator will error if you try.
- **`RefRW<T>.ValueRW` triggers change tracking.** Reading through `ValueRW` and never writing still dirties the chunk.
- **`WithChangeFilter<T>` is chunk-level.** If any entity in the chunk writes to `T`, the whole chunk is considered changed.
- **Aspects are obsolete (but not removed).** The `IAspect` grouping interface was deprecated in 1.4 and is still listed as a valid `Execute` parameter kind on the 6.5 API reference. New code should use plain component parameters; existing aspects will still compile until Entities 2.0.

---

## 6. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Compile error: `IJobEntity must be partial` | Add `partial` to the job struct. |
| `ScheduleParallel` throws a race-condition error | Two jobs write to the same component without an ordering edge. Ensure `state.Dependency` is combined correctly, or add `[UpdateAfter]` between systems. |
| Job doesn't run on Burst | Missing `[BurstCompile]` on the job struct, or a Burst-incompatible type (managed reference, `FixedString`-less `string`) is captured. |
| `SystemAPI.Query` returns nothing | Filter combination is empty. Inspect the query in **Window → Entities → Query** to see what it actually matches. |
| Writing through `RefRO` (silent bug) | Compiler now errors on `refRO.ValueRW = …`. If it slips through older source-gen versions, switch to `RefRW`. |
| Entity-count mismatch between `SystemAPI.Query` and an `EntityQuery` you built manually | Different filter set. `SystemAPI.Query` includes enableable-component filtering by default; `EntityQueryBuilder` requires explicit configuration. |
