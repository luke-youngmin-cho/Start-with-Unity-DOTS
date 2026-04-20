# Structural Change & EntityCommandBuffer  
### Comparing **Direct EntityManager Changes** vs **Deferred ECB Changes**

A Structural Change alters an entity's Archetype, which incurs the cost of **Chunk relocation + a Sync Point**.  
Understand the difference between immediate `EntityManager` calls and the deferred **EntityCommandBuffer (ECB)** so you only pay the necessary cost.

---

## 1. Immediate Changes: Direct `EntityManager` Calls

| Pros | Cons |
|------|------|
| Simple to implement, minimal code | Triggers a **Sync Point** on call — the main thread waits |
| Easy state debugging | Cannot be called from worker Jobs |
| `Add/RemoveComponent`, `Instantiate`, `DestroyEntity` all supported | Performance drops with repeated in-frame calls |

### 1-1 Example
```csharp
// Single-threaded, immediate destruction
if (hp.ValueRO.Value <= 0)
{
    state.EntityManager.DestroyEntity(entity);
}
```
The **WaitForJobGroupID → StructuralChange** event in the *Profiler* can grow longer.

---

## 2. Deferred Changes: **EntityCommandBuffer (ECB)**

| Pros | Cons |
|------|------|
| StructuralChanges can be recorded safely inside multi-threaded Jobs | A **Sync Point** happens at Playback (once, globally) |
| Processes bulk changes in one pass → **minimizes cache misses** | Slightly more complex code, requires an ECB system |
| `AsParallelWriter()` records without cross-thread contention | Playback order must be considered |

### 2-1 Basic Pattern
```csharp
// (1) Create an ECB in the system
var ecb = _endSimEcb.CreateCommandBuffer(state.WorldUnmanaged)
                    .AsParallelWriter();

// (2) Record StructuralChanges in a Job
[BurstCompile]
partial struct DestroyJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;

    void Execute([ChunkIndexInQuery] int chunkIndex,
                 Entity entity, in Health hp)
    {
        if (hp.Value <= 0)
            Ecb.DestroyEntity(chunkIndex, entity);
    }
}

new DestroyJob { Ecb = ecb }.ScheduleParallel();
```
* Use `Begin/EndSimulationEntityCommandBufferSystem` to Playback at the **end of the group**.

### 2-2 Playback Process
1. `Playback()` is invoked at the end of the system group  
2. Commands inside the ECB execute via the **EntityManager API**  
3. One Sync Point → all queries/jobs are rewritten  

---

## 3. Performance Comparison Between the Two Approaches

| Scenario | Direct EntityManager | Deferred ECB |
|----------|------------------|----------|
| **≤ 100 StructuralChanges / frame** | Simple, negligible difference | Overhead (ECB creation & Playback) |
| **Thousands of destroys / spawns** | Long SyncPoint causes frame spikes | One SyncPoint handles all → stable |
| **Inside multi-threaded Jobs** | Not callable (Safety error) | Safe recording via `ParallelWriter` |
| **Debugging** | Easy step-by-step inspection | Must consider Playback timing |

---

## 4. Recommended Best Practices

1. **Bulk/parallel** StructuralChanges → **ECB**  
2. **Small/infrequent** changes & main-thread logic → direct `EntityManager`  
3. **Playback timing**  
   * State-listener systems (`ICleanupComponent`) → adjust placement around **EndSimulationECB**  
4. Use the correct **ParallelWriter index** (`chunkIndex` / `entityInQueryIndex`)  
5. Mixing **direct changes + ECB** in the same system can cause 2 SyncPoints — prefer to separate them

---

## 5. Practical Checklist

- [ ] Any remaining EntityManager calls inside Jobs? → move them to ECB  
- [ ] Does the Playback group's position match the actual dependency order? (`OrderBefore/After`)  
- [ ] Profiler **Entity Timeline → SyncPoint** events ≤ 1 per frame?  
- [ ] No GC or Realloc spikes from ECB capacity (NativeList) growth?  
- [ ] Any errors surface with **Burst Safety Checks** enabled?

> **Summary**  
> *Small batches — direct; large batches — ECB.*  
> Matching the pattern lets you handle Structural Changes reliably without frame spikes.
