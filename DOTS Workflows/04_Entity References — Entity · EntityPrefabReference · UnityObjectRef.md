---
title: Entity References — Entity · EntityPrefabReference · UnityObjectRef
updated: 2026-04-27
folder: DOTS Workflows
---

# Entity References — Entity · EntityPrefabReference · UnityObjectRef
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

DOTS code uses a small set of reference types. They are easy to mix up because they all point at "something", but they live at different points in the authoring-to-runtime pipeline.

| Type | Namespace | What it references | Typical use |
|------|-----------|--------------------|-------------|
| `Entity` | `Unity.Entities` | An entity inside one ECS `World`. | Runtime relationships and component fields. |
| `EntityPrefabReference` | `Unity.Entities.Serialization` | A baked entity prefab asset that can be loaded later. | Streaming and content workflows. |
| `UnityObjectRef<T>` | `Unity.Entities` | A `UnityEngine.Object` asset/object through an unmanaged wrapper. | Referencing meshes, materials, textures, audio clips, and authored assets from ECS data. |

This page is only about DOTS / Entities reference types. Engine-wide `UnityEngine.Object` identifier APIs are outside the scope of this manual.

---

## 2. `Entity`

`Entity` is the runtime ECS handle. It is an unmanaged value that identifies a row of component data inside one `World`.

```csharp
using Unity.Entities;

public struct Target : IComponentData
{
    public Entity Value;
}
```

Important rules:

- An `Entity` value is only meaningful inside the `World` that created it.
- Destroying an entity invalidates the old handle.
- Entity indices can be reused; use `EntityManager.Exists(entity)` before dereferencing a stored handle you did not just query.
- Do not save raw `Entity` values to disk or send them across independent worlds as stable IDs.

```csharp
if (state.EntityManager.Exists(target.Value))
{
    var transform = state.EntityManager.GetComponentData<LocalTransform>(target.Value);
}
```

---

## 3. Entity references in baking

When a Baker converts authoring objects into ECS data, use `GetEntity` to convert a referenced authoring object or prefab into an `Entity` reference.

```csharp
using Unity.Entities;
using UnityEngine;

public class SpawnerAuthoring : MonoBehaviour
{
    public GameObject Prefab;
}

public struct Spawner : IComponentData
{
    public Entity Prefab;
}

public class SpawnerBaker : Baker<SpawnerAuthoring>
{
    public override void Bake(SpawnerAuthoring authoring)
    {
        var entity = GetEntity(TransformUsageFlags.None);
        AddComponent(entity, new Spawner
        {
            Prefab = GetEntity(authoring.Prefab, TransformUsageFlags.Dynamic)
        });
    }
}
```

The result is a runtime ECS reference. It is not a managed `GameObject` reference.

---

## 4. `EntityPrefabReference`

`EntityPrefabReference` is for prefab content that should be loaded through the Entities content pipeline instead of being directly embedded as an `Entity` reference in the current scene.

```csharp
using Unity.Entities;
using Unity.Entities.Serialization;
using UnityEngine;

public class ProjectileLibraryAuthoring : MonoBehaviour
{
    public GameObject ProjectilePrefab;
}

public struct ProjectileLibrary : IComponentData
{
    public EntityPrefabReference ProjectilePrefab;
}

public class ProjectileLibraryBaker : Baker<ProjectileLibraryAuthoring>
{
    public override void Bake(ProjectileLibraryAuthoring authoring)
    {
        var entity = GetEntity(TransformUsageFlags.None);
        AddComponent(entity, new ProjectileLibrary
        {
            ProjectilePrefab = new EntityPrefabReference(authoring.ProjectilePrefab)
        });
    }
}
```

Use `EntityPrefabReference` when the prefab belongs to a content-management or streaming workflow. Use a plain `Entity` prefab reference when the prefab is baked into the same SubScene and is available with the scene.

---

## 5. `UnityObjectRef<T>`

`UnityObjectRef<T>` lets unmanaged ECS data reference a `UnityEngine.Object` without turning the component into a managed component.

```csharp
using Unity.Entities;
using UnityEngine;

public struct ProjectileVisual : IComponentData
{
    public UnityObjectRef<Mesh> Mesh;
    public UnityObjectRef<Material> Material;
}
```

Bake it from authoring data:

```csharp
using Unity.Entities;
using UnityEngine;

public class ProjectileVisualAuthoring : MonoBehaviour
{
    public Mesh Mesh;
    public Material Material;
}

public class ProjectileVisualBaker : Baker<ProjectileVisualAuthoring>
{
    public override void Bake(ProjectileVisualAuthoring authoring)
    {
        var entity = GetEntity(TransformUsageFlags.Dynamic);
        AddComponent(entity, new ProjectileVisual
        {
            Mesh = new UnityObjectRef<Mesh>(authoring.Mesh),
            Material = new UnityObjectRef<Material>(authoring.Material)
        });
    }
}
```

Dereference `.Value` only on the main thread or in managed presentation code:

```csharp
Mesh mesh = visual.Mesh.Value;
Material material = visual.Material.Value;
```

Rules:

- The stored wrapper is unmanaged and can live inside `IComponentData`.
- The target object is still a managed Unity object; `.Value` is not a Burst job operation.
- The referenced object must still be loaded when you dereference it.

---

## 6. Choosing the right reference

| Need | Use |
|------|-----|
| Point from one entity to another in the same world | `Entity` |
| Store a prefab baked into the same SubScene | `Entity` prefab reference from `GetEntity(authoring.Prefab, ...)` |
| Store a prefab for content streaming / deferred loading | `EntityPrefabReference` |
| Store a mesh, material, texture, audio clip, or ScriptableObject reference in unmanaged ECS data | `UnityObjectRef<T>` |
| Persist an entity across sessions | A game-owned stable ID, not a raw `Entity` |

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Stored `Entity` no longer exists | The entity was destroyed or belongs to another world. Re-resolve it or check `EntityManager.Exists`. |
| Prefab reference is `Entity.Null` after baking | The Baker did not call `GetEntity` on the prefab, or the prefab was not included in a baked scene/content workflow. |
| `UnityObjectRef<T>.Value` returns `null` | The referenced Unity object is not loaded or was destroyed. Guard the dereference and reload/re-resolve the asset. |
| Burst job cannot use a mesh/material reference | `UnityObjectRef<T>` storage is unmanaged, but `.Value` touches managed Unity objects. Move dereferencing to main-thread presentation code. |
| Save file stores raw `Entity` values | Raw entity handles are world-local and not stable. Store a game-owned ID or content key instead. |
