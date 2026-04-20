# IJobEntity · SystemAPI.Query · RequireMatchingQueries
### Essential Guide to High-Performance Entity Queries & Jobs

In Entities 1.4, **`IJobEntity`** and **`SystemAPI.Query`** are the primary APIs that replace `Entities.ForEach`.  
The `RequireMatchingQueries` attribute improves safety by **preventing missing or erroneous queries at compile time**.

---

## 1. IJobEntity Basics

| Item | Description |
|------|------|
| **Definition** | `public partial struct MyJob : IJobEntity { void Execute(…); }` |
| **Auto-generation** | Source generator converts it to `IJobChunk` code → Burst-optimized |
| **Scheduling** | `.Schedule()`, `.ScheduleParallel()`, `.Run()` |
| **Burst** | Declare `[BurstCompile]` directly |
| **Parameters** | Supports `in`/`ref` components and attributes like `[ChunkIndexInQuery]` |

### 1-1 Example: Movement System
```csharp
[BurstCompile]
public partial struct MoveJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref LocalTransform t, in Velocity v)
    {
        t.Position += v.Value * DeltaTime;
    }
}

// Invocation within a system
new MoveJob { DeltaTime = SystemAPI.Time.DeltaTime }
    .ScheduleParallel();
```

> **Tip** Pass values captured by the job as **public fields** (Burst-safe).

---

## 2. SystemAPI.Query Patterns

| Pattern | Description |
|-----------|------|
| `foreach (var (rw, ro) in SystemAPI.Query<RefRW<A>, RefRO<B>>())` | Returns `Ref*` tuples |
| `.WithNone<C>()`, `.WithAll<D>()`, `.WithDisabled<E>()` | Query filter chain |
| `.ForEach((RefRW<A> a, in B b) => { … })` *(lambda)* | Convenient for small entity counts |
| `SystemAPI.QueryBuilder().WithAspect<SomeAspect>()` | Aspect filter |

### 2-1 Example: Destroying Entities with Health <= 0
```csharp
foreach (var (e, health) in
        SystemAPI.Query<Entity, RefRO<Health>>()
                 .WithNone<DeadTag>())
{
    if (health.ValueRO.Value <= 0)
    {
        state.EntityManager.AddComponent<DeadTag>(e);
    }
}
```

---

## 3. RequireMatchingQueries Attribute

```csharp
[RequireMatchingQueriesForUpdate]
public partial struct DamageSystem : ISystem
{
    void OnUpdate(ref SystemState state)
    {
        // Query example
        foreach (var hp in SystemAPI.Query<RefRW<Health>>()) { … }
    }
}
```

| Effect | Description |
|------|------|
| **Empty Query Prevention** | If every query returns 0 results, **Update is skipped** |
| **Performance** | Eliminates unnecessary system calls and saves frame time |
| **Caution** | If a query only uses `.WithNone<>` and nothing matches, the system may never run at all |

---

## 4. IJobEntity vs SystemAPI.Query Comparison

| Criterion | **IJobEntity** | **SystemAPI.Query (foreach)** |
|------|----------------|--------------------------------|
| **Threading** | Multi-core via `ScheduleParallel()` | Single-threaded by default |
| **Overhead** | Scheduling cost exists; efficient for large entity counts | Simple for small/conditional logic |
| **Code Length** | Requires a separate struct definition | Concise inside the method |
| **Burst** | Entire struct Burst-compiled | Burst inside the loop (when possible) |
| **When to Use** | Repeated calculations like physics or movement | Events and rare condition checks |

---

## 5. Best Practices

1. **Iterating thousands of entities** → use `IJobEntity.ScheduleParallel()`.  
2. **Entity count < 100** & simple logic → use `SystemAPI.Query` + `foreach`.  
3. Add `[RequireMatchingQueriesForUpdate]` to systems to **avoid empty-frame calls**.  
4. Use `EntityCommandBuffer.ParallelWriter` inside `IJobEntity` to **record StructuralChanges**.  
5. **Profile**: compare Job Schedule vs main-thread loop times and confirm overhead does not outweigh the gain.

---

## 6. Checklist

- [ ] Does the `Execute` method avoid Burst-incompatible types?  
- [ ] Are filters like `.WithAll<SimulationTag>()` clearly added to `SystemAPI.Query`?  
- [ ] Did `RequireMatchingQueriesForUpdate` reduce unnecessary per-frame calls?  
- [ ] When using `ComponentLookup` in a job, is the `isReadOnly` flag set correctly?  

> Choosing the right query/job pattern and leveraging `RequireMatchingQueries` simultaneously improves **CPU frame time** and **code safety**. Measure entity counts and change frequency per system and apply the best-fit approach.
