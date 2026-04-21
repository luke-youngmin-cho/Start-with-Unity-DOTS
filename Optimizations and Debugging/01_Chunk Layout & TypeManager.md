---
title: Chunk Layout & TypeManager
updated: 2026-04-21
folder: Optimizations and Debugging
---

# Chunk Layout & TypeManager
### Unity 6000.5 · Entities 6.5.0

---

## 1. The 16 KB chunk

Every archetype stores its entities in **16 KB chunks**. The chunk holds:

1. A small header (metadata — archetype id, version numbers, enable-bit masks, change versions).
2. A contiguous **SoA** region: one column per component type, each column an array of values.
3. Padding to the 16 KB boundary.

Chunk capacity — how many entities fit — falls out of the math:

```
capacity ≈ (16 KB − header) / sum(sizeof(component_i))
```

A tight archetype with a few small components holds hundreds of entities. A fat archetype with many large components holds a handful. Chunks with low occupancy waste the bytes that could have held more entities.

---

## 2. TypeManager — the component registry

`TypeManager` registers every component, buffer, and system at world startup and assigns each a stable **type index**. From the registry, the runtime reads:

| Field | What it's used for |
|-------|-------------------|
| `TypeIndex` | Packed identifier used in queries and chunk metadata. |
| `SizeInChunk` | Bytes per entity for this component in a chunk. |
| `Alignment` | Column alignment within the chunk. |
| `ElementSize` | Buffer element size (for `IBufferElementData`). |
| `HasBlobReferences` / `HasEntityReferences` | Enables validation during serialization and move operations. |

You rarely touch `TypeManager` directly. What matters is that **adding new component types, renaming them, or moving them between assemblies triggers a type registry rebuild**, which is where startup time is spent in large projects.

> Entities 1.4 reported `TypeManager.Initialize` becoming ~2× faster in large projects by skipping unnecessary assembly scanning. On 6.5 that optimization is inherited.

---

## 3. Inspecting chunk layout — the Archetypes window

**Window → Entities → Archetypes** lists every archetype in the world. For each row:

- **Entities** — live count.
- **Chunks** — number of chunks in use.
- **Entities / Chunk** — average occupancy.
- **Capacity** — theoretical max per chunk.

What to watch for:

| Reading | Interpretation |
|---------|----------------|
| Many archetypes with 1 entity / chunk | Archetype explosion from over-granular tags. Consolidate. |
| One archetype with a huge entity count and high occupancy | Healthy — enemies, projectiles, bullets. |
| Entities / Chunk ≪ Capacity | Fragmentation — occupation is spotty because entities keep leaving for other archetypes (structural churn). |
| `Capacity = 1` | The archetype is too wide (components too big). Check for oversized managed/shared components. |

---

## 4. Designing archetype-friendly components

Heuristics, in rough order of impact:

1. **Keep unmanaged structs unmanaged.** A single managed `IComponentData` forces side-table storage per entity and breaks Burst.
2. **Don't pad your structs.** Put `float3` before `float`, not after — alignment holes inflate `SizeInChunk`.
3. **Split cold fields off.** If only one system reads a field, move it to a sibling component. Unused columns still cost bytes per entity.
4. **Prefer enableable components over per-frame add/remove.** See [`13_Structural Change & Safety.md`](../DOTS Workflows/13_Structural Change & Safety.md) — toggling a bit beats reshaping the archetype.
5. **Limit shared component value sets.** Each new value spawns a new chunk bucket.
6. **Tag components cost zero bytes but still add an archetype.** Don't tag every entity with a unique marker.

---

## 5. Chunk version numbers

Each component column tracks a **change version** — a 32-bit counter bumped whenever any system writes through `RefRW<T>.ValueRW`, `SetComponentData`, or ECB `SetComponent`.

Systems can skip chunks whose version hasn't changed since last run:

```csharp
var query = state.GetEntityQuery(ComponentType.ReadWrite<Health>());
query.SetChangedVersionFilter(typeof(Health));
```

When a query is iterated with a change filter, only chunks bumped since the system last ran are visited. This is how patterns like "on Health change, update UI" become cheap.

Caveats:

- The filter is **chunk-level**, not entity-level. One entity's write dirties the whole chunk.
- Reading through `RefRW<T>.ValueRW` (even if you don't actually mutate) bumps the version.
- Toggling an enableable component also bumps the version.

---

## 6. BlobAsset storage

`BlobAssetReference<T>` holds large, read-only data (meshes, LOD tables, navigation fields) **outside** the chunk. The chunk only stores a pointer. This avoids bloating chunk capacity when many entities share the same baked data.

Practical tip: if you have a component dominated by a `FixedList` or a `DynamicBuffer` that's identical across all entities of a kind, store the big payload in a blob asset referenced from a lean component instead.

---

## 7. Examples

### 7.1 A fat archetype to avoid

```csharp
public struct EnemyState : IComponentData
{
    public float3 Position;
    public quaternion Rotation;
    public float Health;
    public float HealthMax;
    public float Stamina;
    public float StaminaMax;
    public int AIState;
    public float3 LastKnownPlayerPos;
    public float TargetDistance;
    public Entity Target;
    public FixedList512Bytes<float3> PathBuffer;   // 512 bytes per entity!
}
```

One field `PathBuffer` alone eats 512 bytes per entity. Capacity per 16 KB chunk drops into low double digits. Split it:

```csharp
public struct EnemyStats : IComponentData
{
    public float Health, HealthMax, Stamina, StaminaMax;
    public int   AIState;
    public Entity Target;
}

public struct EnemyPathPoint : IBufferElementData
{
    public float3 Value;
}
```

Now the dynamic buffer lives in its own storage and chunk density recovers.

### 7.2 Change filter for cheap UI updates

```csharp
var query = state.GetEntityQuery(ComponentType.ReadWrite<Health>(),
                                 ComponentType.ReadWrite<HealthBarLink>());
query.SetChangedVersionFilter(typeof(Health));

state.Dependency = new UpdateHealthBarJob { /* ... */ }
    .ScheduleParallel(query, state.Dependency);
```

Only chunks where `Health` changed are visited.

---

## 8. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Slow startup in large project | `TypeManager.Initialize` scanning too many assemblies. Check the Profiler's startup trace; make sure you're on a recent Entities version. |
| Capacity of 1 for an archetype | Component is too big (buffer capacity, embedded blob). Move bulk data to a `BlobAssetReference<T>` or `IBufferElementData`. |
| Change filter fires every frame even when "nothing changed" | Someone's reading via `RefRW<T>.ValueRW` without writing. Switch to `RefRO<T>` where possible. |
| Archetype count keeps growing | Per-entity tags with high cardinality, or shared components keyed on continuous values. Consolidate. |
| Entities / Chunk much less than Capacity | Structural churn — adds/removes are scattering entities across archetypes. Switch to enableable components. |
| Managed component dominates CPU profile | Move the managed field to `UnityObjectRef<T>` or split the component; unmanaged paths are much faster. |
