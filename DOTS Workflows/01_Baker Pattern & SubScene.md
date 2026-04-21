---
title: Baker Pattern & SubScene
updated: 2026-04-21
folder: DOTS Workflows
---

# Baker Pattern & SubScene
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

Entities cannot be authored directly in the Hierarchy. Instead, you author **GameObjects** inside a **SubScene** and a **Baker** converts them into entities when the SubScene is closed or built.

```mermaid
flowchart LR
    A["Authoring GameObject<br>+ MonoBehaviour"] --> B["Baker&lt;T&gt;<br>runs per authoring component"]
    B --> C["Entity<br>+ IComponentData"]
    C --> D["EntityScene asset<br>baked output"]
    D --> E["DefaultWorld<br>at runtime"]
```

---

## 2. SubScene lifecycle

A SubScene has three visible states:

| Icon in Hierarchy | State | What it means |
|-------------------|-------|---------------|
| ⬜ (white checkbox) | **Open** | Its GameObjects are editable; still GameObjects, no baking. |
| 🟨 (yellow checkbox) | **Closed** | Bakers have run; its contents exist as entities only. |
| ⏳ | **Baking** | Incremental bake in progress (brief). |

### Create a SubScene

- **Hierarchy → right-click → New Sub Scene → Empty Scene…** — saves a `.unity` asset and adds a `SubScene` component to the parent GameObject.
- Open/close via the checkbox next to the SubScene name. Closing triggers a bake.

### Incremental baking

When the SubScene is open and you edit a GameObject, only the Bakers that touched the modified authoring run again. Avoid expensive work in `Bake()` for this reason.

---

## 3. `Baker<T>` API

A Baker maps one authoring MonoBehaviour to one or more entities + components.

```csharp
public class EnemyAuthoring : MonoBehaviour
{
    public float MaxHealth = 100f;
    public GameObject ProjectilePrefab;

    class Baker : Baker<EnemyAuthoring>
    {
        public override void Bake(EnemyAuthoring authoring)
        {
            // Primary entity for this GameObject.
            var self = GetEntity(TransformUsageFlags.Dynamic);

            AddComponent(self, new Health { Value = authoring.MaxHealth });

            // Bake a GameObject reference into an Entity reference.
            AddComponent(self, new ProjectilePrefabRef
            {
                Value = GetEntity(authoring.ProjectilePrefab, TransformUsageFlags.Dynamic)
            });
        }
    }
}
```

Key APIs:

| API | Purpose |
|-----|---------|
| `GetEntity(TransformUsageFlags)` | Entity for the current authoring GameObject. |
| `GetEntity(Object, TransformUsageFlags)` | Entity corresponding to another GameObject/prefab (registers a prefab entity). |
| `CreateAdditionalEntity(TransformUsageFlags)` | A second entity linked to the same authoring (e.g. sub-parts). |
| `AddComponent<T>(entity, value)` | Attach component data. |
| `AddBuffer<T>(entity)` / `SetBuffer<T>` | Attach a `DynamicBuffer<T>`. |
| `AddSharedComponent<T>` | Attach an `ISharedComponentData`. |
| `DependsOn(asset)` | Declare an asset dependency so the Baker re-runs when the asset changes. |

---

## 4. `TransformUsageFlags`

Baking only adds `LocalTransform` / `LocalToWorld` / `Parent` when you ask for them via `TransformUsageFlags`. Unused transform components make chunks larger for no reason, so pick the **narrowest** flag that still works.

| Flag | Adds | Typical use |
|------|------|-------------|
| `None` | Nothing | Pure data entities (config, singletons, logic-only). |
| `ManualOverride` | Nothing; you manage transforms yourself | Custom transform layouts. |
| `Renderable` | `LocalToWorld` | Static mesh that never moves. |
| `Dynamic` | `LocalTransform`, `LocalToWorld` (+ `Parent` if parented) | Anything that moves or is written by a system. |
| `WorldSpace` | `LocalTransform`, `LocalToWorld`; prevents parenting | Entities that ignore hierarchy. |
| `NonUniformScale` | Adds `PostTransformMatrix` | Non-uniform scale (uncommon in DOTS). |

> `Dynamic` is the safe default for anything that a system will move or rotate.

---

## 5. Baking systems (advanced)

A Baker produces components from a single authoring. A **Baking System** runs **after** all Bakers and sees the full baked world — useful for cross-entity post-processing (resolving references, computing derived data).

```csharp
[WorldSystemFilter(WorldSystemFilterFlags.BakingSystem)]
public partial struct EnemyWaveLinkSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs during bake only. Query entities tagged earlier by Bakers
        // and wire them together.
    }
}
```

Use this when a Baker doesn't know everything it needs (e.g. counts, cross-references). Keep it deterministic — baking must produce the same result on every run.

---

## 6. Example — authoring a projectile spawner

**Authoring:**

```csharp
using Unity.Entities;
using UnityEngine;

public class ProjectileSpawnerAuthoring : MonoBehaviour
{
    public GameObject Prefab;
    public float Interval = 0.5f;

    class Baker : Baker<ProjectileSpawnerAuthoring>
    {
        public override void Bake(ProjectileSpawnerAuthoring authoring)
        {
            var self = GetEntity(TransformUsageFlags.Dynamic);

            AddComponent(self, new ProjectileSpawner
            {
                Prefab = GetEntity(authoring.Prefab, TransformUsageFlags.Dynamic),
                Interval = authoring.Interval,
                TimeLeft = authoring.Interval
            });
        }
    }
}
```

**Component:**

```csharp
using Unity.Entities;

public struct ProjectileSpawner : IComponentData
{
    public Entity Prefab;
    public float Interval;
    public float TimeLeft;
}
```

A runtime system can then instantiate `Prefab` on a timer. See [`02_Spawner Example.md`](02_Spawner Example.md).

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Entity appears but is missing a component | The Baker didn't run (authoring was added after the SubScene closed) — toggle the SubScene to re-bake. |
| `GetEntity(prefab, ...)` returns `Entity.Null` | The referenced GameObject isn't a prefab asset or scene object reachable from the Baker. Ensure it's a prefab or in the same SubScene. |
| Baker runs every frame in the Editor | `Bake()` has non-deterministic logic (e.g. `Random`, `Time.time`). Bakers must be pure functions of the authoring. |
| Entity has `LocalToWorld` but not `LocalTransform` | `TransformUsageFlags.Renderable` was used. Switch to `Dynamic` if the entity will move. |
| Changes to an authoring field don't re-bake | The Baker doesn't read that field or didn't call `DependsOn` for an external asset. |
| Baking is slow | Heavy work inside `Bake()`. Move cross-entity logic into a Baking System or a runtime one-shot initializer. |
