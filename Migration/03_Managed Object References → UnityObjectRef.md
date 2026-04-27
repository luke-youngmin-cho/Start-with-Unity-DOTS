---
title: Managed Object References → UnityObjectRef
updated: 2026-04-27
folder: Migration
---

# Managed Object References → UnityObjectRef
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

When migrating older ECS code, a common problem is managed object references leaking into runtime component data. In Entities 6.5, keep gameplay components unmanaged where possible and use `UnityObjectRef<T>` when ECS data must point at a Unity object such as a mesh, material, texture, audio clip, or authored ScriptableObject.

This page is scoped to DOTS component design. It does not cover engine-wide Unity object identifier APIs.

---

## 2. What to look for

Search for components that store managed Unity objects directly:

| Pattern | Why to review it |
|---------|------------------|
| `class` components implementing `IComponentData` | Managed components are allowed but slower and main-thread constrained. |
| `UnityEngine.Object` fields inside component data | Usually indicates a managed component or invalid unmanaged component design. |
| `GameObject`, `Transform`, `Material`, `Mesh`, `Texture`, `AudioClip` fields | Decide whether this belongs in authoring, presentation, or `UnityObjectRef<T>`. |
| Runtime systems calling `Resources.Load` per entity | Move asset resolution to baking or a central content system. |

---

## 3. Before and after

### Before — managed component

```csharp
using Unity.Entities;
using UnityEngine;

public class ProjectileVisual : IComponentData
{
    public Mesh Mesh;
    public Material Material;
}
```

This works, but it makes the component managed. Managed components reduce Burst/job flexibility and are usually a poor fit for per-entity gameplay data.

### After — unmanaged component with `UnityObjectRef<T>`

```csharp
using Unity.Entities;
using UnityEngine;

public struct ProjectileVisual : IComponentData
{
    public UnityObjectRef<Mesh> Mesh;
    public UnityObjectRef<Material> Material;
}
```

Bake the references once:

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

---

## 4. Runtime access pattern

Keep simulation jobs purely data-oriented. Dereference Unity objects only where managed Unity APIs are allowed.

```csharp
using Unity.Entities;
using UnityEngine;

public partial class ProjectileVisualUploadSystem : SystemBase
{
    protected override void OnUpdate()
    {
        foreach (var visual in SystemAPI.Query<RefRO<ProjectileVisual>>())
        {
            Mesh mesh = visual.ValueRO.Mesh.Value;
            Material material = visual.ValueRO.Material.Value;

            if (mesh == null || material == null)
                continue;

            // Upload to a rendering registry, material property block cache, etc.
        }
    }
}
```

For high-volume rendering, avoid per-frame dereferencing per entity. Cache resolved objects in a presentation-side registry keyed by a compact ECS value.

---

## 5. When not to use `UnityObjectRef<T>`

| Case | Better choice |
|------|---------------|
| Pure gameplay data | Store primitive / math / blob data directly in `IComponentData`. |
| Large read-only tables | `BlobAssetReference<T>`. |
| Entity prefab baked into the same SubScene | `Entity` prefab reference. |
| Prefab streamed through content workflows | `EntityPrefabReference`. |
| Complex managed behavior per entity | Keep it outside the simulation path or isolate it in a managed presentation system. |

---

## 6. Migration checklist

- [ ] Find managed component data used only to store Unity object references.
- [ ] Replace those fields with `UnityObjectRef<T>` where an unmanaged component is appropriate.
- [ ] Move object assignment into Bakers.
- [ ] Keep `.Value` dereferencing out of Burst jobs.
- [ ] Cache dereferenced objects when many entities share the same asset.
- [ ] Use `BlobAssetReference<T>` for large immutable gameplay data instead of ScriptableObject reads at runtime.

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Component cannot be used in a Burst job | It is still managed. Convert Unity object fields to `UnityObjectRef<T>` or split presentation data out. |
| `.Value` is null | The referenced asset is not loaded or was destroyed. Check the authoring reference and content loading path. |
| System allocates every frame | You are resolving or loading assets per entity. Bake references and cache resolved managed objects. |
| Gameplay system depends on a ScriptableObject at runtime | Bake the needed values into component data or a blob asset. |
| Build strips an asset | Ensure the asset is referenced through baking/content workflows, not only by a runtime string path. |
