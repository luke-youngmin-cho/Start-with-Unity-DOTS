# ParallelWriter · Deterministic Playback
### Safely record Structural Changes from multiple threads while guaranteeing a predictable replay order

`EntityCommandBuffer (ECB)` provides **ParallelWriter** for recording structural changes from multi-threaded Jobs.  
Multiple threads can append commands to the same ECB without **data races**,  
and Playback preserves a **deterministic order**, ensuring reproducible game logic.

---

## 1. ParallelWriter Basics

| Item | Description |
|------|------|
| **Creation** | `ecb.AsParallelWriter()` – slot per thread using **ChunkIndexInQuery** |
| **Thread Safety** | Uses per-thread **NativeQueue** internally → lock-free |
| **Supported Commands** | `Instantiate`, `DestroyEntity`, `Add/Remove/SetComponent`, `AddBuffer`, ... |
| **Index Parameter** | Requires a deterministic input value (`chunkIndex` or `sortKey`) |

### 1-1 Example
```csharp
var ecb = _endSimEcbSystem
            .CreateCommandBuffer(state.WorldUnmanaged)
            .AsParallelWriter();

[BurstCompile]
partial struct SpawnJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;
    public Entity Prefab;

    void Execute([ChunkIndexInQuery] int ciq, in SpawnRequest req)
    {
        Entity e = Ecb.Instantiate(ciq, Prefab);
        Ecb.SetComponent(ciq, e, new LocalTransform { Position = req.Position });
    }
}

new SpawnJob { Ecb = ecb, Prefab = prefabEntity }
    .ScheduleParallel();
```

---

## 2. How Deterministic Playback Works

| Stage | Details |
|------|------|
| **Record** | Store commands in buckets keyed by `sortKey` (usually `chunkIndex` or `entityInQueryIndex`) |
| **Sort** | Right before Playback, the ECB sorts commands by **sortKey ASC** |
| **Execute** | Commands with the same SortKey are FIFO; the order across all worker threads is **fixed** |

> **Important**: the `chunkIndex` value depends on the **order of query results** → if the data layout changes, the SortKey changes too.  
> For maximum reproducibility, consider a **stable key** (`entityInQueryIndex` or a custom ID).

### 2-1 Using a Custom sortKey
```csharp
int sortKey = unchecked((int) (entity.Index & 0xFFFF_FFFF)); // Stable ID
Ecb.DestroyEntity(sortKey, entity);
```

---

## 3. Practical Usage Patterns

| Pattern | Recommended Key | Description |
|------|--------|------|
| **Bulk Prefab Instantiate** | `chunkIndex` | Optimal for chunk-level batching |
| **Per-entity destruction** | `entityInQueryIndex` | Deterministic order via unique ID |
| **Hierarchy (parent-child) creation** | Same sortKey | Preserves parent→child creation order |
| **Event component additions** | Combination with `NativeThreadIndex` | Thread discriminator + local index |

---

## 4. Debugging & Profiling

1. **Entities Timeline**  
   * Expand the `Playback (EntityCommandBuffer)` event → check command count and duration  
2. **ECB Statistics (Editor mode)**  
   * Debug window: **Jobs ▸ Entity Debugger ▸ ECB Stats**  
3. **Safety Checks**  
   * Detects **write/read conflicts** in `ENABLE_UNITY_COLLECTIONS_CHECKS` builds

---

## 5. Caveats & Best Practices

| Topic | Guideline |
|------|--------|
| **BurstCompile** | Add `[BurstCompile]` to the Job struct — ECB calls themselves are Burst-compatible |
| **Capacity Estimation** | Use the `capacity` argument of `CreateCommandBuffer` when anticipating many commands |
| **Playback Timing** | End of the Simulation group (`EndSimulationECBSystem`) → guarantees system dependencies |
| **Nested ECB** | Multiple ECB Playbacks per frame possible — watch ordering (`OrderBefore/After`) |
| **Cost** | Sort & memcpy costs exist, but a single SyncPoint typically offsets them |

---

## 6. Checklist

- [ ] Is a **stable sortKey** used in ParallelWriter calls?  
- [ ] No **duplicate commands** on the same entity (e.g., Add & Remove of the same component)?  
- [ ] Is the Playback group positioned as intended? (`UpdateInGroup`, `OrderAfter`)  
- [ ] Has the profiler confirmed whether ECB **Sort + Playback** time is a bottleneck?  
- [ ] Does the configured Capacity prevent **resize spikes**?  

> Used correctly, **ParallelWriter + Deterministic Playback** lets you record structural changes in parallel across multiple cores while satisfying both **order consistency** and **performance**.
