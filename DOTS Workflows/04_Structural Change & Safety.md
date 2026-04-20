# Structural Change & Safety

> A **Structural Change** refers to any operation that changes an entity's **Archetype (component composition)**.  
> This includes adding/removing components, creating/destroying entities, `AddBuffer`, `SetSharedComponent`, and so on.  
> Because these changes trigger **Chunk reorganization**, they must be used with care.

---

## 1. Situations That Cause a Structural Change

| Operation | Example Methods | Description |
|-----------|-----------------|-------------|
| **Entity create / destroy** | `EntityManager.Instantiate`, `EntityManager.DestroyEntity` | Creates or removes a slot in a Chunk |
| **Component add / remove** | `AddComponent`, `RemoveComponent`, `SetSharedComponent` | If the Archetype changes, the entity moves between Chunks |
| **Dynamic Buffer growth** | `DynamicBuffer<T>.Add()` – exceeding capacity causes Chunk reorganization |
| **SharedComponent value change** | `SetSharedComponent` | Changing the value moves the entity to a different Chunk |
| **EntityCommandBuffer.Playback** | All Structural Changes recorded in the ECB |

> **Note** Toggling Enable/Disable (`IEnableableComponent`) is **not a Structural Change**. It only flips a bitmask, so the cost is very low.

---

## 2. Sync Points & Dependencies

* A Structural Change can conflict with all **reader/writer jobs**, triggering a **Sync Point** (the main thread is paused).  
* Calling `EntityManager` APIs **outside of a job** (on the main thread) causes an immediate Sync Point.  
* To record a Structural Change from inside a job, use an **EntityCommandBuffer (ECB)**; the Sync Point then occurs at **Playback** time.

---

## 3. Handling Structural Changes Safely

### 3‑1 EntityCommandBuffer Usage Pattern
```csharp
[BurstCompile]
public partial struct SpawnJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;
    void Execute([ChunkIndexInQuery] int chunkIndex, ref Spawner spawner)
    {
        Entity e = Ecb.Instantiate(chunkIndex, spawner.Prefab);
        Ecb.AddComponent(chunkIndex, e, new Velocity { Value = 1 });
    }
}

// System body
// Obtain a reference to the ECB system
var ecbSystem = state.World.GetExistingSystemManaged<EndSimulationEntityCommandBufferSystem>();
var ecb = ecbSystem.CreateCommandBuffer(state.WorldUnmanaged)
                   .AsParallelWriter();
new SpawnJob { Ecb = ecb }.ScheduleParallel();
```
* **Recording phase**: Record StructuralChanges inside the job via `ParallelWriter`  
* **Playback phase**: `Playback` runs at the end of the system group (`EndSimulationEntityCommandBufferSystem`)

### 3‑2 Macros to Block Structural Changes
```csharp
// First line of OnUpdate
state.CompleteDependency();  // when needed
state.EntityManager.CompleteEntityQueries(); // for Editor validation
```
* During development, you can conditionally validate Structural Change locations with `ENABLE_UNITY_COLLECTIONS_CHECKS`.

### 3‑3 Optimizing Bulk Changes
| Strategy | Description |
|----------|-------------|
| **Batch Instantiate** | Use the `NativeArray<Entity>` overload of `EntityCommandBuffer.Instantiate` |
| **Minimize SharedComponents** | Value changes split Chunks → prefer Unmanaged fields where possible |
| **Use Enable/Disable** | For temporary deactivation, use `IEnableableComponent` to avoid Structural Changes |
| **Prefab + LinkedEntityGroup** | Batch-duplicate/destroy sub-entities to reduce the number of changes |

---

## 4. Safety System

| Safety Mechanism | Purpose |
|------------------|---------|
| **Versioning** | Records component write versions → detects incorrect reads |
| **Dependency Tracking** | Automatically resolves read/write conflicts between jobs (wait/complete) |
| **Atomic Safety Handle** | Detects collection misuse at runtime (e.g. reading after write) |
| **Enable Unity Collections Checks** | Detects index/bounds errors in Editor and Development builds |
| **Burst Safety** | Analyzes invalid pointers/aliasing at compile time |

### 4‑1 Representative Error Messages
| Message | Cause / Solution |
|---------|------------------|
| `Entity has already been destroyed` | Using a stale Entity value after a StructuralChange → check **`entity.IsAlive`** |
| `InvalidOperationException: The previously scheduled job XYZ writes to ...` | Job dependency not declared → use `Schedule` instead of `ScheduleParallel`, or chain the `Dependency` |
| `ArgumentException: A component with type XXX has not been added` | `AddComponent` missing during StructuralChange, or wrong order of `GetComponent<T>` calls |

---

## 5. Practical Checklist

1. Move `EntityManager` StructuralChange code that is **called repeatedly on the main thread** to an **ECB**.  
2. Record from inside jobs via **ParallelWriter** and use `ScheduleParallel()`.  
3. Minimize StructuralChanges with **Enable/Disable** and **Buffer.ResizeUninitialized**.  
4. Inspect the Profiler's **Entity Timeline → Sync Point** events to eliminate bottlenecks.  
5. Validate at runtime with **Collections Checks** enabled in Development builds → disable in performance builds.

---

## 6. Summary
* A StructuralChange is costly due to **Archetype migration** and causes a Sync Point.  
* Reducing the **frequency and scope** of these changes using ECB patterns, Enableable components, and Prefab duplication is key.  
* Leveraging Unity's **safety system** (versioning, dependency tracking, Atomic Safety) lets you maximize the benefits of multithreading without data races or crashes.
