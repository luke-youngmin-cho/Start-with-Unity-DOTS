# Spawner Example
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

A spawner is a single entity that creates more entities — at startup, on a timer, or on an event. This page builds one end to end, covering:

- Authoring the spawner in a SubScene.
- Storing a **prefab entity** reference (baked from a GameObject prefab).
- Instantiating it at runtime through an **EntityCommandBuffer** so structural changes happen safely.

Reading [`01_Baker Pattern & SubScene.md`](01_Baker%20Pattern%20%26%20SubScene.md) first will make this easier.

---

## 2. Data model

```csharp
using Unity.Entities;
using Unity.Mathematics;

public struct Spawner : IComponentData
{
    public Entity  Prefab;       // baked prefab entity
    public float3  SpawnPoint;
    public float   Interval;
    public float   TimeLeft;
    public int     RemainingCount;  // stop when 0
}
```

Two design choices worth calling out:

- `Prefab` is an **`Entity`**, not a `GameObject`. Bakers convert the GameObject prefab into a prefab entity; the runtime only sees entities.
- `TimeLeft` and `RemainingCount` live on the spawner entity so the system stays stateless.

---

## 3. Authoring + Baker

```csharp
using Unity.Entities;
using Unity.Mathematics;
using UnityEngine;

public class SpawnerAuthoring : MonoBehaviour
{
    public GameObject Prefab;
    public float      Interval = 0.5f;
    public int        Count    = 100;

    class Baker : Baker<SpawnerAuthoring>
    {
        public override void Bake(SpawnerAuthoring authoring)
        {
            var self = GetEntity(TransformUsageFlags.Dynamic);

            AddComponent(self, new Spawner
            {
                Prefab        = GetEntity(authoring.Prefab, TransformUsageFlags.Dynamic),
                SpawnPoint    = authoring.transform.position,
                Interval      = authoring.Interval,
                TimeLeft      = 0f,            // fires immediately on first update
                RemainingCount = authoring.Count
            });
        }
    }
}
```

Drop `SpawnerAuthoring` on an empty GameObject in the SubScene, assign a Prefab, set a count. Close the SubScene — the spawner is now an entity.

---

## 4. Runtime system

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

[BurstCompile]
public partial struct SpawnerSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<Spawner>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        // Use the EndSimulation ECB so structural changes flush at a safe point.
        var ecb = SystemAPI
            .GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>()
            .CreateCommandBuffer(state.WorldUnmanaged);

        foreach (var (spawnerRW, spawnerEntity) in
                 SystemAPI.Query<RefRW<Spawner>>().WithEntityAccess())
        {
            ref var spawner = ref spawnerRW.ValueRW;

            if (spawner.RemainingCount <= 0)
            {
                ecb.DestroyEntity(spawnerEntity);   // done — remove the spawner
                continue;
            }

            spawner.TimeLeft -= dt;
            if (spawner.TimeLeft > 0f) continue;

            spawner.TimeLeft = spawner.Interval;
            spawner.RemainingCount--;

            var instance = ecb.Instantiate(spawner.Prefab);
            ecb.SetComponent(instance, LocalTransform.FromPosition(spawner.SpawnPoint));
        }
    }
}
```

What matters:

- `RequireForUpdate<Spawner>()` skips the system entirely when no spawner exists.
- The `EndSimulationEntityCommandBufferSystem` singleton provides a shared ECB that flushes at end-of-frame — safer than mutating through `EntityManager` mid-loop.
- `ecb.Instantiate(prefab)` creates a **deferred entity**; `ecb.SetComponent(instance, …)` schedules component writes on it. The `instance` handle is only valid inside this ECB's playback. See [`14_EntityCommandBuffer · Deferred Entity.md`](14_EntityCommandBuffer%20%C2%B7%20Deferred%20Entity.md).

---

## 5. Scaling up — parallel spawner

If you need to spawn thousands per frame from many spawners, convert to an `IJobEntity` with a parallel ECB writer:

```csharp
[BurstCompile]
public partial struct SpawnJob : IJobEntity
{
    public float                               DeltaTime;
    public EntityCommandBuffer.ParallelWriter  ECB;

    void Execute([ChunkIndexInQuery] int chunkIndex,
                 Entity entity,
                 ref Spawner spawner)
    {
        if (spawner.RemainingCount <= 0)
        {
            ECB.DestroyEntity(chunkIndex, entity);
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
```

And in the system:

```csharp
var ecb = SystemAPI
    .GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>()
    .CreateCommandBuffer(state.WorldUnmanaged)
    .AsParallelWriter();

new SpawnJob { DeltaTime = SystemAPI.Time.DeltaTime, ECB = ecb }
    .ScheduleParallel();
```

The `chunkIndex` sort key is what keeps parallel playback deterministic. See [`15_ParallelWriter · Deterministic Playback.md`](15_ParallelWriter%20%C2%B7%20Deterministic%20Playback.md).

---

## 6. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Nothing spawns | `Spawner` entity doesn't exist — open the SubScene, verify `SpawnerAuthoring` is attached and the SubScene is closed (yellow icon). |
| Prefab spawns at the world origin | The ECB never set `LocalTransform`. Either add `ecb.SetComponent(instance, LocalTransform.FromPosition(...))` or ensure the prefab already has a transform with the desired position. |
| Prefab is visible but not moving | Prefab was baked with `TransformUsageFlags.Renderable` instead of `Dynamic`. |
| `SystemAPI.GetSingleton<...ECBS.Singleton>()` throws | The ECB system isn't registered — happens in very stripped worlds. Fall back to `state.EntityManager` + manual ECB, or register the ECB system in your bootstrap. |
| Spawner never stops | `RemainingCount` decremented against a cached `Spawner` instead of `spawnerRW.ValueRW`. Always write through `RefRW.ValueRW`. |
| `ecb.SetComponent` on a deferred instance is ignored | Using a different ECB for `Instantiate` and `SetComponent`. Both calls must share the same ECB instance. |
