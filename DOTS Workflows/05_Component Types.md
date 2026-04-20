# Complete Guide to Component Types  
### Unmanaged · Managed · Buffer · Chunk · Shared · Tag · Cleanup
---

## 1. At-a-Glance Component Classification

| Category | Interface | Memory Location | Burst Support | Typical Use |
|----------|-----------|-----------------|---------------|-------------|
| **Unmanaged Component** | `IComponentData` *(struct)* | SoA inside Chunk | ✅ | Frequently updated data such as position, velocity, health |
| **Managed Component** | `ManagedComponent<T>` (class) | GC Heap | ❌ | UnityEngine object references, strings, large Asset handles |
| **Dynamic Buffer** | `IBufferElementData` *(struct)* | Variable region at end of Chunk | ✅ | Path nodes, inventories, effect lists – variable length |
| **Chunk Component** | `IComponentData` + `[ChunkSerializable]` | Chunk Header | ✅ | Chunk-wide settings (LOD, Section ID) |
| **Shared Component** | `ISharedComponentData` *(struct/class)* | External management table | ❌ | Shared rendering data, AI group ID – filtering only |
| **Tag Component** | `IComponentData` (no fields) | Inside Chunk | ✅ | State flags ("Dead", "Selected") |
| **Cleanup Component** | `ICleanupComponentData` *(struct)* | Inside Chunk | ✅ | Notifies systems immediately after entity destruction |
| *(For reference)* Enableable | `IEnableableComponent` | Chunk Enable Mask | ✅ | On/Off toggle (without Structural Change) |

---

## 2. Unmanaged vs Managed

| Item | Unmanaged (`IComponentData`) | Managed (class component) |
|------|------------------------------|---------------------------|
| **Storage Location** | Inside Chunk (Struct-of-Arrays) | GC Heap |
| **Burst** | Fully supported | Not supported |
| **GC Cost** | None | Allocation / collection cost |
| **Serialization** | Fast binary copy | Reflection-based |
| **Recommended Size** | ≤ 16 KB (recommended) | Keep small; avoid heavy use |
| **Example** | `float3 Position`, `int Health` | `Mesh mesh`, `string Name` |

> As project scale grows, **minimizing Managed Components** becomes decisive for performance and GC stability.

---

## 3. Dynamic Buffer (`IBufferElementData`)

* **Structure**: A variable-length array laid out contiguously at the back of the Chunk.  
* When **capacity grows**, the buffer moves to a new Chunk → triggers a **Structural Change**.  
* Use **ResizeUninitialized()** to pre-reserve capacity and reduce migration costs.  
* Internal elements must be Burst-supported **Unmanaged structs**.

```csharp
public struct Waypoint : IBufferElementData
{
    public float3 Position;
}
// Usage
var buffer = entityManager.GetBuffer<Waypoint>(e);
buffer.Add(new Waypoint { Position = p });
```

---

## 4. Chunk Component (`IComponentData` + `[ChunkSerializable]`)

* **Only one exists per Chunk** – shared by all entities of the same Archetype.  
* Used for **Chunk-scoped settings** such as render section IDs and LOD masks.  
* Can be filtered in queries via `.AddComponent<ChunkComponent>()`.

---

## 5. Shared Component (`ISharedComponentData`)

| Characteristic | Details |
|----------------|---------|
| **Archetype Split** | Entities with different values go into separate Chunks → O(1) filtering |
| **ECS Query** | Fast group selection via `WithSharedComponentFilter()` |
| **Value Change Cost** | Chunk migration (StructuralChange) + COW; use with care |
| **Burst** | The value itself can be Unmanaged, but table management introduces overhead |

```csharp
entityManager.SetSharedComponent(entity,
    new RenderMaterialID { Value = 3 });
```

---

## 6. Tag Component

* A **field-less struct** → 0 bytes, occupying only the Enable Mask.  
* State is expressed purely by presence/absence.  
* `AddComponent<TagDead>(entity)` → "dead" state; remove with `RemoveComponent<TagDead>`.

---

## 7. Cleanup Component (`ICleanupComponentData`)

* Not removed immediately when the entity is **Destroyed**; **persists for one additional frame**.  
* A system querying `ICleanupComponentData` can perform cleanup work right after destruction.  
* Automatically removed on the next frame, so no memory leak occurs.

---

## 8. Design Guidelines

1. **Frequent updates** → Unmanaged Components.  
2. **On/Off toggles** → avoid Structural Changes via `IEnableableComponent`.  
3. **Bulk filtering criteria** → Shared Component, but keep value changes to a minimum.  
4. **Chunk-wide values** → use a Chunk Component to deduplicate data.  
5. **Temporary / post-destroy handling** → use a Cleanup Component.  
6. **GC-sensitive logic** → avoid Managed Components; replace with Asset IDs (numbers).

---

## 9. Quick Checklist

- [ ] Frequently updated data isn't sitting on the GC Heap, is it?  
- [ ] Buffer capacity growth isn't causing excessive StructuralChanges, is it?  
- [ ] SharedComponent value changes aren't happening every frame, are they?  
- [ ] Are you Remove/Add-ing fields that could simply be toggled with `IEnableableComponent`?  

> Choosing the right component type based on this table optimizes **cache efficiency, GC cost, and StructuralChange frequency** all at once.
