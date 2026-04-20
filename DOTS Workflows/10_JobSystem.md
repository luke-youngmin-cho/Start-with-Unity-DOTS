# Job System Overview, Dependencies, and Scheduling Overhead
### Achieving high-performance multithreading with Entities 1.4

Unity's **Job System** uses multi-core CPUs to process ECS data in parallel.
It automatically tracks **data dependencies** between jobs to ensure safety, and keeping scheduling overhead to a minimum is the key.

---

## 1. Job System Overview

| Element | Description |
|---------|-------------|
| **Job** | Implementations of interfaces such as `IJob`, `IJobFor`, `IJobParallelFor`, `IJobEntity`, `IJobChunk` |
| **Scheduler** | Unity JobWorker thread pool pulls jobs from the queue and executes them |
| **Burst** | Compiles to native code with C++-level optimizations |
| **Dependency** | A **JobHandle** graph that tracks Read/Write conflicts between jobs |
| **Complete()** | Wait for a JobHandle to finish (blocks the main thread) |

---

## 2. Job Type Comparison

| Interface | Target | Characteristics |
|-----------|--------|-----------------|
| `IJob` | Single task | Simple / short logic |
| `IJobParallelFor` | Index array | Static work-stealing partitioning |
| `IJobEntity` | ECS entities | Source generator emits `IJobChunk` code |
| `IJobChunk` | Chunk-level | Low-level, highest performance (manual loop) |
| `Entities.ForEach` *(Obsolete)* | Entities | Recommend replacing with `IJobEntity` |

---

## 3. Dependency Management

### 3-1 Automatic Tracking
* `SystemAPI.Query<RefRO<A>, RefRW<B>>()` -> dependencies are computed from read/write flags
* Concurrent **write** requests on the same component cause jobs to be **serialized**.

### 3-2 JobHandle Chaining
```csharp
JobHandle h1 = job1.Schedule();
JobHandle h2 = job2.Schedule(h1);           // Dependency link
state.Dependency = h2;                      // System-wide Dependency
```

### 3-3 `CombineDependencies`
```csharp
state.Dependency = JobHandle.CombineDependencies(handleA, handleB);
```
* Use when you need to wait on multiple jobs simultaneously.

### 3-4 Full Wait (`Complete()`)
```csharp
state.Dependency.Complete();  // When subsequent code needs the result
```
* **Caution**: Blocks the main thread -> frequent usage causes frame drops.

---

## 4. Scheduling Overhead

| Situation | Cause of Overhead | Solution |
|-----------|-------------------|----------|
| **Few entities per job** | Scheduling + context switch cost exceeds the work cost | Switch to `Run()` or a single-threaded `foreach` |
| **Empty jobs** | Query matches 0 -> scheduling unnecessary | `[RequireMatchingQueriesForUpdate]` |
| **Excessive Complete calls** | Main-thread waits | Minimize `Complete()` by chaining dependencies |
| **Burst off inside a job** | Calls to managed code -> increases JobWorker thread load | Use struct-only data and Burst-compatible APIs |
| **Too-fine chunk splitting** | `ScheduleParallel` split (stride) not tuned | Measure with `state.GetProcessedChunkCountWithoutFilter()` and tune |

### 4-1 Overhead Metrics
* **Profiler > Timeline > Worker Threads** - check **JobFence** (waiting) / **JobFlush** (submit) events
* **System Inspector > Frame Time** - `Schedule + WaitForJobGroupID` span

---

## 5. Best Practices

1. **Entities >= 500** or heavy computation -> **`ScheduleParallel()`**
2. **Entities < 200** -> **`Run()`** or `foreach`
3. **Minimize job captures**: pass `NativeArray`, `ComponentLookup`, `float` values, etc. as **struct fields**
4. Use **BurstCompile** together with `Unity.Mathematics` -> leverage CPU SIMD
5. Record StructuralChanges via **EntityCommandBuffer.ParallelWriter**
6. **Iteration stride**: tune `batchCount` for `IJobChunk` (4~8 chunks)
7. Keep **`state.CompleteDependency()`** calls to **at most once per frame boundary**

---

## 6. Practical Checklist

- [ ] Before calling `ScheduleParallel`, is the `Dependency` chain set up correctly?
- [ ] Are `Complete()` calls placed only where absolutely required (right before accessing data)?
- [ ] Is **MainThread WaitForJobGroup** time in the Profiler not excessive?
- [ ] Are there no Burst-off paths (managed code) running on Job Worker threads?
- [ ] Are you scheduling a job every frame even when the query returns 0 entities? -> `RequireMatchingQueries`

> To fully reap the benefits of the Job System, the **scheduling gain > overhead** condition must hold.
> Measure entity counts and workload complexity, and choose appropriately between **Run <-> Schedule <-> ScheduleParallel**.
