---
title: Structural Change & Safety
updated: 2026-04-21
folder: DOTS Workflows
---

# Structural Change & Safety
### Unity 6000.5 · Entities 6.5.0

---

## 1. What counts as a structural change

A **structural change** is any operation that can move entities between chunks or alter the chunk inventory. All of the following are structural:

| Operation | Why it's structural |
|-----------|---------------------|
| `CreateEntity` / `DestroyEntity` | Allocates or frees a chunk slot. |
| `AddComponent<T>` / `RemoveComponent<T>` | Entity's archetype changes, so it physically moves to a different chunk. |
| `SetSharedComponent<T>` | Different shared-component value → different chunk (groups by value). |
| `AddChunkComponent<T>` / `RemoveChunkComponent<T>` | Changes the chunk's own metadata. |
| `SetName(entity, ...)` | Touches entity metadata in the EntityManager. |
| `MoveEntitiesFrom` | Copies entities between worlds. |
| `Instantiate(prefab)` | Allocates a new entity (may pre-fill an archetype). |

Writing a component's **value** via `SetComponentData` / `RefRW<T>.ValueRW` is **not** structural — the component already lives in the chunk; you're just changing bytes.

---

## 2. Why structural changes are dangerous

When the archetype changes, anything holding a position-based reference into a chunk goes stale:

- In-flight query iterators (`SystemAPI.Query<...>`) may skip or revisit entities.
- `ComponentLookup<T>` entries become invalid for affected entities.
- `ArchetypeChunk` handles in a job point to the **old** chunk.
- Other systems running concurrently observe inconsistent state.

The Safety System catches most of these with an `InvalidOperationException` in the Editor. In a build, they become crashes or corrupt entities.

---

## 3. Rule of thumb

> **Do not perform structural changes while iterating a query in the same system update.**

Two compliant patterns, in order of preference:

1. **Use an `EntityCommandBuffer`.** Record the operation during iteration, play it back after. See [`14_EntityCommandBuffer · Deferred Entity.md`](14_EntityCommandBuffer · Deferred Entity.md).
2. **Iterate into a `NativeList<Entity>`, then mutate after the loop.** Simpler for small cases; no ECB overhead.

Example of pattern 2:

```csharp
var toKill = new NativeList<Entity>(state.WorldUpdateAllocator);

foreach (var (health, e) in SystemAPI
             .Query<RefRO<Health>>()
             .WithEntityAccess())
{
    if (health.ValueRO.Value <= 0f)
        toKill.Add(e);
}

state.EntityManager.DestroyEntity(toKill.AsArray());
```

---

## 4. Enableable components — the escape hatch

If you find yourself adding and removing a component every frame, it should almost certainly be an **enableable component** instead — toggling the enable bit is **not** structural. See [`06_Enableable Component.md`](06_Enableable Component.md).

---

## 5. Jobs and structural changes

Structural changes are **main-thread operations**. You cannot perform them inside a job.

The idiomatic ways to mutate from a job:

| Approach | When |
|----------|------|
| `EntityCommandBuffer.ParallelWriter` recorded in the job, played back on the main thread | Per-entity structural changes from parallel code. |
| `EntityCommandBuffer` (single-thread) recorded in an `IJob` or `IJobChunk`, played back on main thread | Batched / chunk-scoped changes. |
| Toggle an enableable component via `EnabledRefRW<T>` inside a job | Per-entity "activate/deactivate" without structure. |

An ECB captures the operation as a command; playback happens at a known point on the main thread, usually at an `EntityCommandBufferSystem` boundary.

---

## 6. Combining structural changes and reads

If two systems run back-to-back and system A performs a structural change that system B depends on, make the ordering explicit:

```csharp
[UpdateInGroup(typeof(SimulationSystemGroup))]
[UpdateBefore(typeof(MovementSystem))]
public partial struct SpawnSystem : ISystem { /* uses ECB */ }
```

When the ECB system playback sits between A and B (e.g. `EndSimulationECBS` with a suitable `[UpdateAfter]` chain), the structural change lands before B reads.

---

## 7. Cost model

Rough ranking for a single change:

```
SetComponentData (value only)          ~ 1x
SetComponentEnabled (enableable bit)   ~ 1x (per-chunk bitmask write)
AddComponent (structural)              ~ 10x   (entity moves chunk)
SetSharedComponent (structural)        ~ 10x
DestroyEntity (structural)             ~ 10x
Per-frame add + remove pattern         pathological
```

Orders of magnitude; exact numbers depend on archetype size, chunk occupancy, and allocator pressure.

---

## 8. Diagnosing structural churn

Symptoms:
- `Window → Entities → Systems` shows a system with disproportionate timing for its logic.
- Profiler marker `EntityManager.SetArchetype` or `Move Entity` dominates frames.
- Archetype count grows over time.

Tactics:
- Audit `AddComponent` / `RemoveComponent` calls — are any per-frame? Replace with enableable components.
- Inspect shared-component value sets — any accidentally per-entity keys?
- Check the Archetypes window for one-entity chunks; that's fragmentation from over-granular component sets.

---

## 9. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `InvalidOperationException: EntityCommandBuffer has already been played back` | Recording into an ECB after its playback. Each ECB is single-shot — get a fresh one from the ECBS each frame. |
| `InvalidOperationException: … invalidated …` during foreach | Structural change inside the loop. Move to an ECB or buffer-then-apply. |
| Entities silently disappear | `DestroyEntity` happened but other systems still hold the old `Entity` handle. Check handles with `EntityManager.Exists`. |
| Add/Remove pattern churns frame time | Replace with enableable components. |
| `RemoveComponent` triggers "cleanup component present" error | The entity has a cleanup component. Handle cleanup first (see [`05_Component Types.md`](05_Component Types.md)). |
| Shared component value set explodes (hundreds of distinct values) | The value is too granular. Bucket it (e.g. rounded to the nearest N) or store the variable piece as a regular component. |
