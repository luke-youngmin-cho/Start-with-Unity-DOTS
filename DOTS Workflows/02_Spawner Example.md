# ECS Workflow Quick Start — **Spawner** Tutorial

> This tutorial is a step-by-step example for learning the ECS workflow through **entity creation, component read/write, and runtime instantiation**. The overall flow performs the following five tasks in order.

| Step | Task | Goal |
|------|------|------|
| 1 | Create a SubScene | Create a SubScene to hold the example content |
| 2 | Create a component | Define the **Spawner** component used by the spawner logic |
| 3 | Create an entity | Convert GameObject → Entity using Authoring + Baker |
| 4 | Write a system | Implement a single-threaded `SpawnerSystem` |
| 5 | Optimize the system | Implement a Burst-Job based **parallel** `OptimizedSpawnerSystem` |

---

## 1. Creating a SubScene
1. Right-click in the Hierarchy → select **New Sub Scene ▸ Empty Scene**, then save.  
2. Or select an existing GameObject and convert it via **New Sub Scene ▸ From Selection**.  
3. You can attach a **SubScene** component to an empty GameObject and assign a *Scene Asset* to reference an existing SubScene.

> **Tip** Enabling `Auto Load Scene` makes the SubScene stream automatically in Play mode.

---

## 2. Writing the Spawner Component
```csharp
using Unity.Entities;
using Unity.Mathematics;

public struct Spawner : IComponentData
{
    public Entity Prefab;
    public float3 SpawnPosition;
    public float  NextSpawnTime;
    public float  SpawnRate;
}
```
* **Prefab**  : Template entity to be duplicated at runtime  
* **SpawnPosition** : Spawn location  
* **SpawnRate**  : Spawn interval (seconds)  
* **NextSpawnTime** : Next spawn time (updated at runtime)
---

## 3. Creating the Spawner Entity (Authoring + Baker)

### 3‑1 Authoring Component
```csharp
using UnityEngine;
using Unity.Entities;

class SpawnerAuthoring : MonoBehaviour
{
    public GameObject Prefab;
    public float SpawnRate;
}
```

### 3‑2 Baker
```csharp
class SpawnerBaker : Baker<SpawnerAuthoring>
{
    public override void Bake(SpawnerAuthoring authoring)
    {
        var entity = GetEntity(TransformUsageFlags.Dynamic);
        AddComponent(entity, new Spawner
        {
            Prefab        = GetEntity(authoring.Prefab, TransformUsageFlags.Dynamic),
            SpawnPosition = authoring.transform.position,
            NextSpawnTime = 0f,
            SpawnRate     = authoring.SpawnRate
        });
    }
}
``` 

### 3‑3 Editor Setup
1. Create an empty GameObject named **Spawner** in the SubScene.  
2. Add **SpawnerAuthoring** and set the *Prefab* and *Spawn Rate* (e.g. 2 seconds).  
3. Set the **Entities Hierarchy** window to *Runtime* or *Mixed* mode to inspect the baked Spawner entity and its component values directly.

---

## 4. Writing the Spawner System (single-threaded)

```csharp
using Unity.Entities;
using Unity.Transforms;
using Unity.Burst;

[BurstCompile]
public partial struct SpawnerSystem : ISystem
{
    public void OnCreate (ref SystemState state) { }
    public void OnDestroy(ref SystemState state) { }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (RefRW<Spawner> spawner in SystemAPI.Query<RefRW<Spawner>>())
        {
            ProcessSpawner(ref state, spawner);
        }
    }

    void ProcessSpawner(ref SystemState state, RefRW<Spawner> spawner)
    {
        if (spawner.ValueRO.NextSpawnTime < SystemAPI.Time.ElapsedTime)
        {
            // Verify that the Prefab is valid
            if (spawner.ValueRO.Prefab == Entity.Null)
            {
                UnityEngine.Debug.LogError("Spawner's Prefab has not been set!");
                return;
            }

            Entity e = state.EntityManager.Instantiate(spawner.ValueRO.Prefab);
            state.EntityManager.SetComponentData(
                e,
                LocalTransform.FromPosition(spawner.ValueRO.SpawnPosition));

            spawner.ValueRW.NextSpawnTime =
                (float)SystemAPI.Time.ElapsedTime + spawner.ValueRO.SpawnRate;
        }
    }
}
```

* `RefRW<Spawner>` : for **read/write** access to the component.  
* The system runs on a **single thread**, so it is suited to processing a small number of entities.

---

## 5. Optimizing the Spawner System (Burst + Parallel Job)

```csharp
using Unity.Collections;
using Unity.Entities;
using Unity.Transforms;
using Unity.Burst;

[BurstCompile]
public partial struct OptimizedSpawnerSystem : ISystem
{
    public void OnCreate (ref SystemState state) { }
    public void OnDestroy(ref SystemState state) { }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = GetEntityCommandBuffer(ref state);

        new ProcessSpawnerJob
        {
            ElapsedTime = SystemAPI.Time.ElapsedTime,
            Ecb         = ecb
        }.ScheduleParallel();
    }

    EntityCommandBuffer.ParallelWriter GetEntityCommandBuffer(ref SystemState state)
    {
        // Retrieve the BeginSimulationEntityCommandBufferSystem and create an ECB
        var ecbSystem = state.World.GetExistingSystemManaged<BeginSimulationEntityCommandBufferSystem>();
        var ecb = ecbSystem.CreateCommandBuffer(state.WorldUnmanaged);
        return ecb.AsParallelWriter();
    }
}

[BurstCompile]
public partial struct ProcessSpawnerJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;
    public double ElapsedTime;

    void Execute([ChunkIndexInQuery] int chunkIndex, ref Spawner spawner)
    {
        if (spawner.NextSpawnTime < ElapsedTime)
        {
            // Verify that the Prefab is valid
            if (spawner.Prefab == Entity.Null)
                return;

            Entity e = Ecb.Instantiate(chunkIndex, spawner.Prefab);
            Ecb.SetComponent(
                chunkIndex, e,
                LocalTransform.FromPosition(spawner.SpawnPosition));

            spawner.NextSpawnTime =
                (float)ElapsedTime + spawner.SpawnRate;
        }
    }
}
```

* **`IJobEntity` + `ScheduleParallel()`** : processes many entities across multiple threads.  
* **EntityCommandBuffer** : records create/set operations off the main thread and plays them back later.  
* **BurstCompile** : compiles to native code, boosting CPU performance.  

> **Note** When the number of entities is small, the **scheduling overhead** may exceed the performance gain. Measure with the Profiler before applying.

---

## 6. Verifying in Play Mode
* When entering Play mode, verify in the Scene / Game view that the **Prefab** spawns at the configured interval.  
* New entities will appear in real time in the `Entities Hierarchy` window.
* To see the benefits of multithreading, duplicate the Spawner object multiple times and run **many spawners at once**.

---

### Wrap-up
With this **Spawner example** you have practiced the basic ECS workflow of SubScene → Baker → System → Job optimization. As a next step, try combining **Entities Graphics**, **Netcode for Entities**, and similar packages to build more complex runtime logic.
