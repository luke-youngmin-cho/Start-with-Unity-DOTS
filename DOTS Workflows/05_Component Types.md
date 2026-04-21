---
title: Component Types
updated: 2026-04-21
folder: DOTS Workflows
---

# Component Types
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

A "component" in ECS is any data container attached to an entity. Entities exposes several flavours with different storage, lifetime, and cost profiles. Picking the right one is half the work of designing an ECS system.

Quick map:

| Interface / Form | Storage | Structural change on mutate? | Typical use |
|------------------|---------|------------------------------|-------------|
| `IComponentData` (unmanaged `struct`) | In chunk, SoA | No (on value change) | Default component |
| `IComponentData` (managed `class`) | Side table | No | Reference types (meshes, GameObject refs) |
| `IBufferElementData` | In chunk + heap overflow | No | Per-entity dynamic arrays |
| `ISharedComponentData` | Per-chunk | Yes (moves entity to new chunk) | Grouping entities that share config |
| `ICleanupComponentData` | In chunk | Normal | Entity-destroy cleanup protocol |
| `ICleanupSharedComponentData` | Per-chunk | Normal | Shared-resource cleanup protocol |
| Tag (empty `IComponentData`) | Zero-size in chunk | No | Filter marker, no data |
| Chunk component | Per-chunk | Yes | Metadata computed once per chunk |
| Enableable (`IEnableableComponent`) | In chunk + bit mask | No | Turn behaviour on/off per entity |

The rest of this page expands each row.

---

## 2. Unmanaged `IComponentData`

The default. Lives directly in the chunk as tightly-packed SoA data. Burst-friendly.

```csharp
public struct Velocity : IComponentData
{
    public float3 Value;
}
```

Rules:
- Must be a `struct` — no reference fields (no `class`, no `string`, no `List<T>`).
- Allowed field types: primitives, `Unity.Mathematics`, other unmanaged structs, `Entity`, `FixedString*`, `BlobAssetReference<T>`, `UnityObjectRef<T>`.
- Read via `RefRO<T>`, write via `RefRW<T>` from `SystemAPI.Query`.

---

## 3. Managed `IComponentData`

Lives in a side table indexed by entity. Works with reference types but costs a GC allocation and breaks Burst compatibility for that component.

```csharp
public class EnemyAI : IComponentData
{
    public AIModel Brain;      // a managed class
    public List<Entity> Allies;
}
```

Use when you genuinely need a managed reference (mesh, texture, authored ScriptableObject). Prefer `UnityObjectRef<T>` inside an unmanaged component when you just need a weak reference to a `UnityEngine.Object`.

---

## 4. `IBufferElementData` — dynamic buffers

A `DynamicBuffer<T>` is a per-entity resizable array.

```csharp
public struct Waypoint : IBufferElementData
{
    public float3 Position;
}

var buffer = state.EntityManager.AddBuffer<Waypoint>(entity);
buffer.Add(new Waypoint { Position = new float3(1, 0, 0) });
```

Storage: up to `InternalBufferCapacity` elements live in-chunk; beyond that the buffer spills to the heap. Tune capacity with `[InternalBufferCapacity(N)]`.

---

## 5. `ISharedComponentData` — grouping

Shared components carry a **value that groups chunks**. Two entities with the same `ISharedComponentData` value end up in the same chunk; different values produce different chunks.

```csharp
public struct RenderMeshGroup : ISharedComponentData
{
    public int MaterialID;
}
```

Consequences:
- Changing the value moves the entity to another chunk → **structural change**, expensive.
- Great for data that doesn't change frequently but varies per group (material sets, team IDs, bake regions).
- Values are stored in a managed table if the type contains references (`ISharedComponentData` can be managed).

---

## 6. Cleanup components — `ICleanupComponentData`

Mark an entity so that, when it is "destroyed," it stays alive with only its cleanup components until your system processes them.

```csharp
public struct ConnectionCleanup : ICleanupComponentData
{
    public int ConnectionHandle;
}

// DestroyEntity(e) removes all normal components but leaves ConnectionCleanup
// behind so you can close the socket in a dedicated system, then destroy the
// cleanup component explicitly.
```

The sibling `ICleanupSharedComponentData` applies the same protocol to shared resources.

Use cases: releasing native resources, un-registering from external systems, staged destruction across frames.

---

## 7. Tag components

A **tag** is an empty `IComponentData`:

```csharp
public struct Enemy : IComponentData {}
```

Zero bytes in a chunk, queryable like any other component. Ideal for "is a type of X" markers. Overusing tags (hundreds of unique ones) fragments archetypes — use enableable components or shared components for high-cardinality states.

---

## 8. Chunk components

A chunk component carries **one value per chunk**, not per entity.

```csharp
state.EntityManager.AddChunkComponentData<RegionBounds>(query, new RegionBounds { ... });
```

Typical use: precomputed data that applies to all entities in the chunk (bounds, region ID, partition key). Mutating a chunk component causes a structural change.

---

## 9. Enableable components (`IEnableableComponent`)

See the dedicated page [`06_Enableable Component.md`](06_Enableable Component.md) for depth. Summary here: a component that can be toggled on/off per entity **without** a structural change, via a per-chunk bitmask.

```csharp
public struct Stunned : IComponentData, IEnableableComponent
{
    public float Duration;
}

state.EntityManager.SetComponentEnabled<Stunned>(entity, false);  // no structural change
```

Queries automatically skip entities whose enableable components are disabled — a cheap, common way to temporarily exclude entities from a system.

---

## 10. Decision guide

| I want to… | Use |
|-----------|-----|
| Store per-entity numeric data | Unmanaged `IComponentData` |
| Hold a managed reference per entity | Managed `IComponentData` or `UnityObjectRef<T>` |
| Store a variable-length list per entity | `IBufferElementData` |
| Group entities for batching | `ISharedComponentData` |
| Toggle a component on/off per frame | Enableable component |
| Just mark a type | Tag (empty `IComponentData`) |
| Precompute data for a whole chunk | Chunk component |
| Run code when an entity is destroyed | Cleanup component |

---

## 11. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `InvalidOperationException: The type T must be unmanaged` | You used a managed type where unmanaged is required. Switch the component or the API (e.g. `ComponentLookup<T>` vs `ManagedAPI.GetComponent<T>`). |
| Setting a shared component value stutters | Each change is a structural change. Batch with an ECB or accept that this value is rarely mutated. |
| `DynamicBuffer<T>` spills to the heap | `[InternalBufferCapacity(N)]` is too small for your usage. Increase it or keep sizes bounded. |
| Entity reappears after `DestroyEntity` | A cleanup component is still attached. Remove it explicitly once cleanup is done. |
| Too many archetypes | Tag explosion. Convert unique-per-entity "states" to enableable components. |
| Tag component isn't filtering a query | Tag components have no data, so you can't request a `RefRW<Tag>` / `RefRO<Tag>` in the `SystemAPI.Query<...>` tuple. Apply them via `.WithAll<Tag>()` / `.WithNone<Tag>()` instead. |
