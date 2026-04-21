---
title: ParallelWriter · Deterministic Playback
updated: 2026-04-21
folder: DOTS Workflows
---

# ParallelWriter · Deterministic Playback
### Unity 6000.5 · Entities 6.5.0

---

## 1. Why there is a separate parallel writer

`EntityCommandBuffer` is single-writer. Two threads recording into the same ECB would race on its internal chain. `EntityCommandBuffer.ParallelWriter` solves this by giving each caller a **thread-local chain** and requiring a **sort key** per command.

At playback time the ECB system:

1. Flattens all thread-local chains.
2. Sorts commands by `sortKey` (stable sort).
3. Executes them on the main thread.

The result is deterministic as long as your sort keys are deterministic.

---

## 2. Getting a ParallelWriter

```csharp
var ecb = SystemAPI
    .GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>()
    .CreateCommandBuffer(state.WorldUnmanaged)
    .AsParallelWriter();
```

`AsParallelWriter()` returns a `struct EntityCommandBuffer.ParallelWriter` you can copy into an `IJobEntity`, `IJobChunk`, or `IJobParallelFor`.

---

## 3. The sortKey contract

Every parallel ECB call takes a `sortKey` as its first argument:

```csharp
ecb.Instantiate(sortKey, prefab);
ecb.SetComponent(sortKey, entity, value);
ecb.DestroyEntity(sortKey, entity);
ecb.AddComponent(sortKey, entity, component);
```

Rules for a **good** sort key:

- **Deterministic.** Same input → same key, no matter how the scheduler distributes work.
- **Unique enough.** Two commands on the same entity should not collide unless you want playback order to be ambiguous.
- **Cheap.** An `int` you already have, not a hash.

In practice, the go-to choices are:

| Source | How to get it | When to use |
|--------|---------------|-------------|
| `[ChunkIndexInQuery]` on `IJobEntity` | Source generator fills it in | 99% of cases |
| `unfilteredChunkIndex` in `IJobChunk.Execute` | Provided by the API | `IJobChunk` jobs |
| `[EntityIndexInQuery]` on `IJobEntity` | Source generator | When commands per entity must be ordered individually |
| A manually-computed deterministic index | You compute it | Rare; custom job shapes |

---

## 4. Full example — parallel spawn

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

[BurstCompile]
public partial struct SpawnParallelJob : IJobEntity
{
    public float                               DeltaTime;
    public EntityCommandBuffer.ParallelWriter  ECB;

    void Execute([ChunkIndexInQuery] int chunkIndex,
                 Entity spawnerEntity,
                 ref Spawner spawner)
    {
        if (spawner.RemainingCount <= 0)
        {
            ECB.DestroyEntity(chunkIndex, spawnerEntity);
            return;
        }

        spawner.TimeLeft -= DeltaTime;
        if (spawner.TimeLeft > 0f) return;

        spawner.TimeLeft = spawner.Interval;
        spawner.RemainingCount--;

        var inst = ECB.Instantiate(chunkIndex, spawner.Prefab);
        ECB.SetComponent(chunkIndex, inst, LocalTransform.FromPosition(spawner.SpawnPoint));
    }
}

[BurstCompile]
public partial struct SpawnerParallelSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = SystemAPI
            .GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>()
            .CreateCommandBuffer(state.WorldUnmanaged)
            .AsParallelWriter();

        new SpawnParallelJob
        {
            DeltaTime = SystemAPI.Time.DeltaTime,
            ECB       = ecb
        }.ScheduleParallel();
    }
}
```

Passing `chunkIndex` to both `Instantiate` and the subsequent `SetComponent` makes sure the two commands sort to the same position and therefore the deferred-entity reference from `Instantiate` resolves to the right target.

---

## 5. Determinism — what it guarantees, what it doesn't

**Guaranteed:** Given the same ECB contents, playback produces identical resulting state — regardless of which worker thread recorded which command.

**Not guaranteed** automatically:

- **Entity IDs.** Two runs might produce entities with different `Index` values if prior commands differed.
- **Chunk ordering across systems.** If system A produces entities in parallel and system B reads them in order of `ChunkIndexInQuery`, the iteration order can still depend on archetype creation history.

If you need bit-for-bit determinism across runs (replays, Netcode rollback), combine:

1. Parallel ECB with deterministic sort keys.
2. Fixed-step systems inside `FixedStepSimulationSystemGroup`.
3. No reliance on `Entity.Index`; key on gameplay IDs you control.

---

## 6. Sort-key patterns

### 6.1 Per-chunk (the default)

Use `[ChunkIndexInQuery] int chunkIndex`. Collisions are fine if every entity in the chunk generates an identical command (e.g. a whole chunk being marked stunned).

### 6.2 Per-entity

Use `[EntityIndexInQuery] int entityIndex` when you need strict per-entity ordering (e.g. recording per-entity damage events that must play back in entity order).

### 6.3 Custom deterministic index

Very rarely, you build a flat index from inputs yourself — e.g. `int sortKey = baseOffset + localIndex;`. Only do this when the common two don't fit; custom indices are a frequent source of determinism bugs.

---

## 7. Avoid these patterns

- **Using `JobsUtility.ThreadIndex`** as a sortKey. Not deterministic.
- **Using `Random` to build the key.** Not deterministic.
- **Mixing ECBs.** Record into two different ECBs and you lose the total ordering.
- **Recording to a per-system ECB from multiple systems.** One ECB, one recorder (or one parallel-writer wrapping one ECB).

---

## 8. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Deferred entity resolves to the wrong target | `Instantiate` and the follow-up `SetComponent` used different sort keys. Keep them the same. |
| Commands fire in different order each run | Non-deterministic sort key. Switch to `[ChunkIndexInQuery]`. |
| "JobEntity must have partial" compile error | Add `partial` to the job struct. |
| `ECB.SetComponent` in a parallel job throws "wrong thread" | Using the non-parallel `EntityCommandBuffer` inside a parallel job. Call `.AsParallelWriter()` first and use the parallel API. |
| Netcode replay desyncs | A parallel ECB somewhere doesn't use a deterministic sort key, or an `IJobEntity` captured a non-deterministic value. Audit the chain. |
| `AsParallelWriter()` on a disposed ECB | ECB has already been played back — get a fresh one this frame. |
