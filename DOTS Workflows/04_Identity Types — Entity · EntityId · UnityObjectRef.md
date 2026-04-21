---
title: Identity Types — Entity · EntityId · UnityObjectRef
updated: 2026-04-21
folder: DOTS Workflows
---

# Identity Types — Entity · EntityId · UnityObjectRef
### Unity 6000.5 · Entities 6.5.0

---

## 1. Three IDs, three jobs

A DOTS codebase on Unity 6000.5+ routinely touches **three** identity types that have similar names and come from different layers of Unity. Mixing them up is the most common migration mistake.

| Type | Namespace · Origin | Identifies | Lives where |
|------|-------------------|-----------|-------------|
| `Entity` | `Unity.Entities` · **Entities package** | An entity inside an ECS `World` | ECS runtime only |
| `EntityId` | `UnityEngine` · **Unity engine (6000.3+)** — NOT an Entities-package type | A `UnityEngine.Object` instance (scene object or asset); replaces `InstanceID` | Engine-wide (every Unity project) |
| `UnityObjectRef<T>` | `Unity.Entities` · **Entities package** | A weak, unmanaged reference to a `UnityEngine.Object`, usable as component data | ECS components; safe in Burst |

> The name `EntityId` is confusing but is **not** an alias for the ECS `Entity`. It is the engine-wide replacement for `UnityEngine.Object.GetInstanceID()` introduced in Unity 6000.3, shipped in the engine itself, and relevant to every Unity project — not just DOTS. It shows up in this manual because DOTS codebases often held `int` InstanceIDs in components or dictionaries.

---

## 2. `EntityId` — the new engine-level identity

Before Unity 6000.3, `UnityEngine.Object.GetInstanceID()` returned an `int` that uniquely identified the object within the Editor session. That `int` was used for logging, dictionaries, serialization, and a long tail of debugging tricks.

Unity 6000.3 introduced the `EntityId` type in `UnityEngine` and began marking legacy `int`-based APIs obsolete. Unity 6000.4 extended the obsolete range. **Unity 6000.5 is the version where the transition gets teeth** — implicit conversion between `int` and `EntityId` is marked obsolete, `FindObjectsByType(FindObjectsSortMode.InstanceID)` is deprecated, and the Editor release notes announce that `EntityId` will grow from 4 to 8 bytes.

In **Unity 6000.6α** the implicit cast is removed entirely and `InstanceIDToObject(int)` becomes a compile error. The migration is time-limited: warnings on 6000.5, errors on 6000.6.

```csharp
using UnityEngine;          // EntityId lives in UnityEngine — not Unity.Entities

UnityEngine.Object obj = ...;
EntityId id = obj.GetEntityId();   // available since Unity 6000.3; preferred on 6000.5+
```

### `EntityId` can't be treated as an `int`

The old `InstanceID` was an `int` with lots of hidden structure (sign bit = asset vs scene, sequential allocation, etc.). `EntityId` is an opaque handle — its internal representation is an implementation detail. On Unity 6000.5+ with 6000.6 incoming, the value is also scheduled to widen to 8 bytes.

Do **not** rely on any of these behaviours any more:

| Old habit | Status on 6000.5 | Expected on 6000.6+ |
|-----------|------------------|---------------------|
| Casting `(int)instanceId` or `(EntityId)anInt` | Obsolete warning | Compile error |
| Checking `id > 0` / `id < 0` to distinguish asset vs scene | Never guaranteed; now actively broken on 8-byte values | Same |
| `obj.GetHashCode()` to derive an instance id | No longer 1:1 | Same |
| `id.ToString()` → `int.Parse(...)` round-trip | Format is not parseable as `int` | Same |
| Sorting to imply creation order | `EntityId` is versioned/reused — doesn't imply order | Same |
| Packing state into the high/first bit | No contract on bit layout | Same |
| `FindObjectsByType(FindObjectsSortMode.InstanceID)` | **Deprecated on 6000.5** | Removed in future |
| `InstanceIDToObject(int)` in Editor scripts | Obsolete | **Compile error on 6000.6α** |

See the migration checklist in [`Optimizations and Debugging/04_EntityId Audit — Deprecated InstanceID Hunt.md`](../Optimizations and Debugging/04_EntityId Audit — Deprecated InstanceID Hunt.md) and the step-by-step refactor in [`Migration/03_InstanceID → EntityId.md`](../Migration/03_InstanceID → EntityId.md).

---

## 3. `Entity` — the ECS handle (Entities package)

`Entity` is a lightweight, unmanaged `struct` from `Unity.Entities`. It only exists inside an ECS `World` and has no meaning outside the Entities package.

```csharp
public readonly struct Entity : IEquatable<Entity>
{
    public int Index;
    public int Version;
}
```

Key properties:

- **Per-World.** The same `Entity` value is meaningful only inside the world that produced it.
- **Reusable slots.** Destroying an entity frees its `Index`; a newly created entity may reuse that slot but with a bumped `Version`. Old handles no longer resolve.
- **Cheap by value.** Pass entities by value everywhere; they're 8 bytes.

Checking validity:

```csharp
if (state.EntityManager.Exists(entity)) { /* ... */ }
```

---

## 4. `UnityObjectRef<T>` — a bridge for ECS components (Entities package)

`UnityObjectRef<T>` is an **Entities package** type (`Unity.Entities` namespace). ECS components must be unmanaged, but sometimes you genuinely need to reference a `UnityEngine.Object` (a mesh, a texture, an AudioClip). `UnityObjectRef<T>` wraps that reference as an unmanaged value:

```csharp
using Unity.Entities;

public struct ProjectileVisual : IComponentData
{
    public UnityObjectRef<Mesh>     Mesh;
    public UnityObjectRef<Material> Material;
}
```

Baked in a Baker:

```csharp
AddComponent(entity, new ProjectileVisual
{
    Mesh     = new UnityObjectRef<Mesh>(authoring.Mesh),
    Material = new UnityObjectRef<Material>(authoring.Material)
});
```

At runtime:

```csharp
Mesh mesh = projectile.Mesh.Value;   // dereference back to a managed object
```

Rules:
- `UnityObjectRef<T>` is **Burst-compatible** for its storage, but `.Value` touches the managed world — do that on the main thread.
- Lifetime: the target `UnityEngine.Object` must still exist when you call `.Value`; the ref itself does not keep the asset loaded.

---

## 5. Choosing the right type

| Need | Use |
|------|-----|
| Identify an entity inside an ECS world | `Entity` |
| Identify a `UnityEngine.Object` (asset, scene object) | `EntityId` |
| Put a reference to a `UnityEngine.Object` inside an ECS component | `UnityObjectRef<T>` |
| Persist across Editor sessions / serialization | None of the above — use a `GlobalObjectId` (Editor) or a content-addressed asset key (runtime) |

---

## 6. Example — cross-referencing ECS and engine objects

A hybrid case: an ECS system logs damage by the attacker's GameObject name.

```csharp
public struct Attacker : IComponentData
{
    public UnityObjectRef<UnityEngine.Object> Source;
}

// Main-thread system only — .Value dereferences managed.
public partial class DamageLogSystem : SystemBase
{
    protected override void OnUpdate()
    {
        foreach (var (attacker, health) in
                 SystemAPI.Query<RefRO<Attacker>, RefRO<Health>>())
        {
            var src = attacker.ValueRO.Source.Value;
            Debug.Log($"Damage from {(src != null ? src.name : "<destroyed>")}: hp={health.ValueRO.Value}");
        }
    }
}
```

Note: `UnityObjectRef<T>`'s target can be destroyed independently — check for `null` before using `.Value`.

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `GetInstanceID()` marked deprecated | Replace with `GetEntityId()` on `UnityEngine.Object`. |
| Dictionary keyed on `int` InstanceID no longer works | Re-key on `EntityId`, or on `UnityObjectRef<T>` if the dict is inside ECS code. |
| Save file contains old InstanceID values | Old save data is no longer portable — migrate on load using a fresh lookup by name/path. |
| `UnityObjectRef<T>.Value` returns `null` after a scene unload | The managed target was destroyed. Re-resolve the reference or guard with a null check. |
| Entity handle silently resolves to a different entity | The original entity was destroyed and the slot reused. Check `EntityManager.Exists(e)` and the `Version` field. |
| `EntityId` cast to `int` compile error | Intended — the cast is no longer valid. Replace the downstream code that needed the int. |
