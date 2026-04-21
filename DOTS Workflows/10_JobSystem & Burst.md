---
title: JobSystem & Burst
updated: 2026-04-21
folder: DOTS Workflows
---

# JobSystem & Burst
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

DOTS performance rests on two engine features that existed before Entities and still underpin it:

- **Job System** — schedule work onto worker threads with dependency graphs.
- **Burst Compiler** — compile C# jobs to tight native code (SIMD, no GC, no virtual dispatch).

Neither is an Entities-specific concept. `IJobEntity` and `IJobChunk` are thin layers on top of `IJob` + Burst. Understanding the base layer makes every higher-level pattern read clearly.

---

## 2. Job System — the core types

| Interface | Shape | Use when… |
|-----------|-------|-----------|
| `IJob` | `void Execute()` | One-shot single-threaded work. |
| `IJobParallelFor` | `void Execute(int index)` | Loop from 0 to N in parallel. |
| `IJobParallelForBatch` | `void Execute(int start, int count)` | Loop in parallel batches (cache-friendlier than per-index). |
| `IJobChunk` | `void Execute(ArchetypeChunk chunk, int chunkIndex, bool useEnabledMask, in v128 enabledMask)` | Iterate ECS chunks in parallel. |
| `IJobEntity` | `void Execute([component params])` | ECS entity iteration — source-generated into a chunk job. |

All of these are **structs** and all support Burst.

### Minimal `IJobParallelFor` example

```csharp
using Unity.Burst;
using Unity.Collections;
using Unity.Jobs;

[BurstCompile]
struct ScaleJob : IJobParallelFor
{
    public NativeArray<float> Data;
    public float Factor;

    public void Execute(int index) => Data[index] *= Factor;
}

// Scheduling:
var handle = new ScaleJob { Data = arr, Factor = 2f }.Schedule(arr.Length, 64);
handle.Complete();
```

Notes:
- Fields must be blittable.
- Write-access to the same `NativeArray` from multiple jobs requires dependency chaining (below).

---

## 3. Scheduling and dependencies

```csharp
JobHandle a = jobA.Schedule();
JobHandle b = jobB.Schedule(a);              // b runs after a
JobHandle c = jobC.Schedule(JobHandle.CombineDependencies(a, b));
c.Complete();
```

Rules:

- Every `Schedule()` returns a `JobHandle`. Pass the previous handle in as a dependency if the new job reads or writes shared data.
- **Never skip `.Complete()`** on the final handle before the main thread reads the results (the Safety System enforces this).
- In an `ISystem`, `state.Dependency` is the system-level handle; scheduled jobs automatically combine with it if you use `.Schedule(state.Dependency)` or rely on source-generated helpers like `IJobEntity.ScheduleParallel(...)`.

### Inside an `ISystem`

```csharp
[BurstCompile]
public void OnUpdate(ref SystemState state)
{
    var job = new MyJob { Dt = SystemAPI.Time.DeltaTime };
    state.Dependency = job.ScheduleParallel(state.Dependency);
}
```

`IJobEntity.ScheduleParallel()` without an explicit dependency will combine with `state.Dependency` implicitly. Be explicit when chaining multiple jobs.

---

## 4. Burst — what and why

Burst compiles a subset of C# (`[BurstCompile]`-marked methods, no GC allocations, blittable data) to highly optimised native code via LLVM.

### What Burst compiles well

- Struct `IJob*` / `ISystem` methods tagged `[BurstCompile]`.
- Math on `Unity.Mathematics` types (`float3`, `quaternion`, `int4`, etc.).
- Tight loops over `NativeArray<T>`, `NativeList<T>`, `DynamicBuffer<T>`, `ArchetypeChunk` column arrays.

### What Burst cannot compile

- Managed references (`class`, `string`, `List<T>`, anything not unmanaged).
- Exceptions (you can catch them, but generally avoid).
- Dynamic dispatch through interfaces (explicit generic instantiation is fine).
- `System.IO` / `Reflection` / most of BCL beyond math/primitives.

### Enabling Burst

```csharp
using Unity.Burst;

[BurstCompile]
public partial struct MyJob : IJobEntity
{
    // attribute on the struct marks the type for Burst
    // methods need [BurstCompile] too if you want each compiled
    [BurstCompile]
    void Execute(ref Velocity v) { /* ... */ }
}
```

For systems:

```csharp
[BurstCompile]
public partial struct MySystem : ISystem
{
    [BurstCompile] public void OnCreate(ref SystemState state) { }
    [BurstCompile] public void OnUpdate(ref SystemState state) { }
}
```

> The attribute on the type is **not** enough — each method you want compiled also needs `[BurstCompile]`.

### Burst Inspector

**Window → Analysis → Burst Inspector** shows the LLVM IR, assembly, and vector usage per method. Use it when "why isn't this fast" is the question.

---

## 5. Native containers

Burst and the Safety System expect **native containers** instead of managed collections:

| Managed | Native equivalent |
|---------|-------------------|
| `T[]` | `NativeArray<T>` |
| `List<T>` | `NativeList<T>` |
| `Dictionary<K,V>` | `NativeHashMap<K,V>`, `NativeParallelHashMap<K,V>` |
| `HashSet<T>` | `NativeHashSet<T>`, `NativeParallelHashSet<T>` |
| `Queue<T>` | `NativeQueue<T>` |
| `string` | `FixedString32Bytes` / `FixedString64Bytes` / etc. |

Allocation:

```csharp
var arr = new NativeArray<int>(1024, Allocator.TempJob);
// ...
arr.Dispose();
```

Lifetime matters: Burst + Safety verifies that containers are disposed and not accessed after a job's lifetime.

---

## 6. Allocators

| Allocator | Lifetime | Typical use |
|-----------|----------|-------------|
| `Allocator.Temp` | Within the current frame, main thread only | Scratch containers in `OnUpdate` |
| `Allocator.TempJob` | Across one job, ≤4 frames | Containers passed to jobs |
| `Allocator.Persistent` | Manual `Dispose()` required | System-owned containers that live across frames |
| `state.WorldUpdateAllocator` | Cleared at the end of the system group update | Preferred for per-system transient data |

Prefer `state.WorldUpdateAllocator` (or its Rewindable allocator) inside systems — it avoids manual `Dispose()` bookkeeping.

---

## 7. Safety System

In the Editor with "Jobs > Leak Detection" and "Jobs > Safety Checks" on, Unity validates that:

- Two jobs don't write the same native container without a dependency edge.
- A main-thread read doesn't race with a running write.
- Native containers aren't used after `Dispose()` or past their allocator's lifetime.

Safety errors surface as exceptions with a clear "job A and job B both access X" message. **Do not disable safety checks in development** — performance wins are small and debugging cost is high.

---

## 8. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Burst warning: "method not burst compiled" | Missing `[BurstCompile]` on the method; or Burst found an unsupported type (check the Burst Inspector for the failing line). |
| `InvalidOperationException: … writes to … without declaring dependency` | Missing dependency edge — pass the previous `JobHandle` or use `state.Dependency` chaining. |
| `ObjectDisposedException` on a NativeArray | Disposed before the job finished. Use `arr.Dispose(handle)` or wait on the handle first. |
| Scheduled job never completes | Missing `.Complete()` on a chain, or a circular dependency. |
| Performance regression after adding `[BurstCompile]` | Method is being recompiled each domain reload; check Burst cache. Or a Burst-incompatible type slipped in — the method silently fell back to IL. |
| `NativeHashMap` enumeration throws | Enumeration doesn't play well with concurrent writes — enumerate a `NativeArray<K>` copy instead. |
| Main-thread stall waiting for a job | Called `.Complete()` too early. Move the completion point as late as possible in the frame. |
