---
title: IJobEntity vs IJobChunk
updated: 2026-04-21
folder: DOTS Workflows
---

# IJobEntity vs IJobChunk
### Unity 6000.5 · Entities 6.5.0

---

## 1. The short answer

- **`IJobEntity`** for "do the same thing to each entity."
- **`IJobChunk`** for "do something to each chunk that isn't a simple per-entity map."

`IJobEntity` is generated on top of `IJobChunk` — they run on the same iteration machinery. `IJobChunk` just exposes the full chunk to your code.

---

## 2. `IJobEntity` — per-entity ergonomics

```csharp
[BurstCompile]
public partial struct DamageTickJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref Health health, in Burn burn)
    {
        health.Value -= burn.DamagePerSecond * DeltaTime;
    }
}
```

Strengths:
- One method, one entity view. Easy to write, easy to read.
- Source-generated plumbing: `Entity`, `[ChunkIndexInQuery]`, `[EntityIndexInQuery]` parameters are filled in for you.
- Filters via `[WithAll]` / `[WithNone]` / `[WithAny]` attributes on the struct, or via a `QueryBuilder` passed to `.Schedule*`.

Weaknesses:
- No cross-entity visibility inside `Execute`. Can't easily "sum over this chunk" without extra state.
- No access to chunk-level metadata (change version, shared component value) from inside `Execute`.

---

## 3. `IJobChunk` — full-chunk control

```csharp
[BurstCompile]
public partial struct HealthAggregateJob : IJobChunk
{
    [ReadOnly] public ComponentTypeHandle<Health> HealthHandle;
    public NativeArray<float> PerChunkTotal;  // size = chunk count

    public void Execute(in ArchetypeChunk chunk,
                        int unfilteredChunkIndex,
                        bool useEnabledMask,
                        in v128 chunkEnabledMask)
    {
        var healths = chunk.GetNativeArray(ref HealthHandle);
        float total = 0f;
        for (int i = 0; i < chunk.Count; i++)
            total += healths[i].Value;

        PerChunkTotal[unfilteredChunkIndex] = total;
    }
}
```

Strengths:
- Direct access to chunk arrays: `chunk.GetNativeArray(ref handle)`, `chunk.GetBufferAccessor(ref bufferHandle)`.
- `chunk.Has<T>()`, `chunk.GetSharedComponent<T>(...)` for conditional work.
- Sees the enabled-mask and change version per chunk.
- Perfect for reductions, spatial partitioning, batched ECB recording.

Cost:
- More boilerplate: declare `ComponentTypeHandle<T>` in the system, update them each frame with `handle.Update(ref state)`.
- You manage the inner `for` loop yourself — so more places to get wrong.

### Setting up the handles in a system

```csharp
[BurstCompile]
public partial struct HealthAggregateSystem : ISystem
{
    private EntityQuery _query;
    private ComponentTypeHandle<Health> _healthHandle;

    public void OnCreate(ref SystemState state)
    {
        _query        = state.GetEntityQuery(ComponentType.ReadOnly<Health>());
        _healthHandle = state.GetComponentTypeHandle<Health>(true);
    }

    public void OnUpdate(ref SystemState state)
    {
        _healthHandle.Update(ref state);

        var totals = CollectionHelper.CreateNativeArray<float>(
            _query.CalculateChunkCount(), state.WorldUpdateAllocator);

        state.Dependency = new HealthAggregateJob
        {
            HealthHandle   = _healthHandle,
            PerChunkTotal  = totals
        }.ScheduleParallel(_query, state.Dependency);
    }
}
```

`handle.Update(ref state)` refreshes the cached type handle each frame — forget this and Burst will hit stale indices.

---

## 4. Enabled mask (`useEnabledMask`)

For queries that include an `IEnableableComponent`, `Execute` receives `useEnabledMask = true` and a `v128` mask of enabled bits. You must respect the mask or you'll process disabled entities.

Helper:

```csharp
var enumerator = new ChunkEntityEnumerator(useEnabledMask, chunkEnabledMask, chunk.Count);
while (enumerator.NextEntityIndex(out int i))
{
    // i is the index of the next enabled entity in the chunk
}
```

`IJobEntity` applies the mask for you automatically — another reason to prefer it unless you need chunk-level visibility.

---

## 5. Decision guide

| Situation | Pick |
|-----------|------|
| Tick health, translate position, apply input — pure per-entity math | `IJobEntity` |
| Sum values across all entities in a chunk | `IJobChunk` |
| Record one ECB command per chunk instead of per entity | `IJobChunk` |
| Access a shared component value inside the loop | `IJobChunk` |
| Need the chunk's change version | `IJobChunk` |
| Iterate a `DynamicBuffer<T>` on each entity | Either works; `IJobEntity` is cleaner |
| Per-entity logic but need the entity's chunk index | `IJobEntity` with `[ChunkIndexInQuery] int chunkIndex` |
| Scheduling cost matters and the work is tiny | Benchmark — fewer chunks beats fewer entities. `IJobChunk` usually wins at thousands of chunks. |

---

## 6. Interop — using both together

A common pattern: gather per-chunk data with `IJobChunk`, then do per-entity writes with `IJobEntity`, chained through dependencies.

```csharp
var gather = new HealthAggregateJob { ... }
    .ScheduleParallel(query, state.Dependency);

var act = new HealJob { PerChunkTotal = totals }
    .ScheduleParallel(gather);

state.Dependency = act;
```

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `IJobChunk.Execute` reads garbage values | Forgot `handle.Update(ref state)` before scheduling, or the wrong query was used. |
| `ComponentTypeHandle<T>` not found | Missing `using Unity.Entities;`, or the handle was declared with the wrong access mode. Use `GetComponentTypeHandle<T>(true)` for read-only. |
| Enabled entities skipped | Mask not respected — use `ChunkEntityEnumerator` as shown. `IJobEntity` avoids this. |
| Job writes to the same `NativeArray` from two chunks | Provide a per-chunk slot (as in `HealthAggregateJob`) or use `NativeParallelHashMap` with chunk index as key. |
| Scheduling `IJobChunk` without `query` argument | `ScheduleParallel(query, ...)` requires the query explicitly. `IJobEntity.ScheduleParallel()` picks it up from attributes/source generation. |
| `v128` type not found | `using Unity.Burst.Intrinsics;` is missing. |
