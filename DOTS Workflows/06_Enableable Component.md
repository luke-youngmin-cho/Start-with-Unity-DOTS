# Enableable Component
### Unity 6000.5 · Entities 6.5.0

---

## 1. Why enableable components exist

Adding or removing a component is a **structural change**: the entity moves to a different chunk, queries must re-resolve, references may be invalidated. If you do this every frame per entity (e.g. toggling "Stunned" on and off), the cost dominates the frame.

An **enableable component** gives you a way to turn behaviour on and off **without** a structural change. The component is always present on the chunk; a bit in a per-chunk mask decides whether each entity "has" it from a query's point of view.

---

## 2. The interface

Implement both `IComponentData` and `IEnableableComponent`:

```csharp
using Unity.Entities;

public struct Stunned : IComponentData, IEnableableComponent
{
    public float Duration;
}
```

You can also make an `IBufferElementData` enableable.

---

## 3. Toggling

From a system:

```csharp
// Turn Stunned off without removing the component
state.EntityManager.SetComponentEnabled<Stunned>(entity, false);

// From an ECB (deferred, safe mid-iteration)
ecb.SetComponentEnabled<Stunned>(entity, true);

// From a job with a ComponentLookup
var stunnedLookup = SystemAPI.GetComponentLookup<Stunned>();
stunnedLookup.SetComponentEnabled(entity, false);
```

Reading the current state:

```csharp
bool isStunned = state.EntityManager.IsComponentEnabled<Stunned>(entity);
```

All three paths operate on a bitmask; none of them trigger a structural change.

---

## 4. Query behaviour

`SystemAPI.Query<...>` and `EntityQuery`-based iteration **skip** entities whose enableable components are currently disabled — by default, you only see entities where every listed component is enabled.

```csharp
foreach (var stunned in SystemAPI.Query<RefRW<Stunned>>())
{
    // only fires for entities where Stunned is enabled
}
```

Override this if you need to see disabled entities too:

```csharp
foreach (var (stunnedRW, entity) in SystemAPI
             .Query<RefRW<Stunned>>()
             .WithEntityAccess()
             .WithOptions(EntityQueryOptions.IgnoreComponentEnabledState))
{
    // fires for ALL entities with the component, enabled or not
}
```

---

## 5. Example — stun that expires

A single `IJobEntity` cannot receive both `ref Stunned` (a `RefRW<T>`) **and** `EnabledRefRW<Stunned>` for the same component — Entities explicitly disallows wrapping the same component in both forms in one job. Two valid patterns:

### 5.1 Read + write value, toggle enabled bit via ECB

```csharp
[BurstCompile]
public partial struct StunTickJob : IJobEntity
{
    public float                               DeltaTime;
    public EntityCommandBuffer.ParallelWriter  ECB;

    void Execute([ChunkIndexInQuery] int chunkIndex,
                 Entity entity,
                 ref Stunned stunned)
    {
        stunned.Duration -= DeltaTime;
        if (stunned.Duration <= 0f)
            ECB.SetComponentEnabled<Stunned>(chunkIndex, entity, false);
    }
}
```

Playback happens at the next ECB boundary (e.g. `EndSimulationECBS`) — see [`14_EntityCommandBuffer · Deferred Entity.md`](14_EntityCommandBuffer%20%C2%B7%20Deferred%20Entity.md). No structural change; just a bitmask flip at a known point.

### 5.2 Toggle the enabled bit directly when you do not need the component value

```csharp
[BurstCompile]
public partial struct StunDisableJob : IJobEntity
{
    void Execute(EnabledRefRW<Stunned> stunnedEnabled)
    {
        stunnedEnabled.ValueRW = false;   // no structural change
    }
}
```

`EnabledRefRW<T>` is the only `Stunned` reference on this job, so it compiles. Use this shape when a system is purely a switcher.

> Want to both read the value **and** toggle the enabled bit on the same entity in one pass? Do it from the main thread via `EntityManager.SetComponentEnabled<T>(entity, false)`, or split across two jobs chained through `state.Dependency`.

---

## 6. When to prefer enableable over add/remove

| Situation | Pick |
|-----------|------|
| High-frequency toggling (per-frame, per-event) | **Enableable** |
| Adds state that isn't always present (per-entity modifier with variable fields) | **Enableable** |
| Low-frequency archetype change (e.g. adding a permanent capability) | Add/remove the component as usual |
| Filtering many systems uniformly | **Enableable** (queries pick it up automatically) |

General rule: if the "has this component" bit flips multiple times in an entity's lifetime, make it enableable.

---

## 7. Interaction with other features

| Feature | Behaviour |
|---------|-----------|
| `WithChangeFilter<T>` | Toggling the enable bit also bumps the component's change version. |
| `ICleanupComponentData` | A cleanup component cannot also be enableable. |
| Netcode for Entities | Enableable components can be ghost-synced, but toggles must go through the authoritative side. |
| Serialisation / world export | The enable bits are preserved. |

---

## 8. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Query includes entities that should be "off" | Component isn't implementing `IEnableableComponent`, or `WithOptions(EntityQueryOptions.IgnoreComponentEnabledState)` is set. |
| `SetComponentEnabled<T>` throws "type T does not support being enableable" | The struct only implements `IComponentData` — add `IEnableableComponent`. |
| `EnabledRefRW<T>` parameter is rejected by source generator | Job or system is missing `partial`, or the component isn't enableable. |
| Source generator error "cannot have both RefRW and EnabledRefRW for the same component" | Split the logic — use `ref T` **or** `EnabledRefRW<T>` per job, not both on the same component. See §5 for the two patterns. |
| Change filter fires spuriously | Toggling `Enabled` counts as a change for the purposes of `WithChangeFilter`. Expected — combine with `EnabledRefRO` reads rather than toggling. |
| ECB `SetComponentEnabled` is ignored | You called it on a deferred entity before the ECB that created it flushed. Both must be in the same ECB cycle. |
