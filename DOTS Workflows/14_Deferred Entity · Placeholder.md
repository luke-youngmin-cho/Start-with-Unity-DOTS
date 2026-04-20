# Deferred Entity · Placeholder Concept
### Safely referencing **entities that do not yet exist** during baking or runtime

In Entities workflows, the **EntityCommandBuffer (ECB)** or **Baker**  
uses a **deferred** approach when calling `Instantiate()`, `CreateAdditionalEntity()`, `GetEntity(prefab)`, etc.:  
> "**Do not create the entity right now; create all of them together later**"  
The value they return is exactly the **Deferred Entity / Placeholder**.

---

## 1. Why Is It Needed?

| Situation | Problem | Deferred Entity Solution |
|------|------|---------------------|
| Bulk Instantiate **inside Job threads** | Structural changes cannot happen immediately | Returns only a Placeholder ID, then one Sync Point later |
| **Baker** creates several additional entities that reference each other | The real Entity does not exist yet | Pre-link via Placeholders within the same ECB/Baker |
| Add components after **Prefab instantiation** | Need an EntityHandle right after Instantiate | Obtain the ID via Placeholder, then record `AddComponent()` |

---

## 2. How It Works

```text
1) Instantiate() called
   └─ Entity {index = -1, version = uniqueID}  ◄ Placeholder returned
2) Commands in the same ECB use the Placeholder
3) At Playback, the real entity is allocated
   └─ Placeholder → Real Entity remap
```

* **Index = -1** or similar negative value → indicates it is not yet in a Chunk  
* The **Version** field is matched via a unique key  
* During `EntityCommandBufferPlayback`, every Placeholder is remapped to a real Entity

---

## 3. Code Examples

### 3-1 Creating a Parent-Child Pair Inside a Job
```csharp
[BurstCompile]
partial struct SpawnHierarchyJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;
    public Entity Prefab;

    void Execute([ChunkIndexInQuery] int ciq)
    {
        // (1) Instantiate the parent prefab → Placeholder p
        Entity parent = Ecb.Instantiate(ciq, Prefab);

        // (2) Add a child
        Entity child  = Ecb.CreateEntity(ciq); // New Placeholder
        Ecb.AddComponent(ciq, child, new Parent { Value = parent });
        Ecb.AddComponent(ciq, child, new LocalTransform { /* ... */ });
    }
}
```
At *Playback* time, `Parent.Value` is automatically replaced with the real parent Entity.

### 3-2 Linking Additional Entities in a Baker
```csharp
public class SpawnerBaker : Baker<SpawnerAuthoring>
{
    public override void Bake(SpawnerAuthoring authoring)
    {
        var spawner = GetEntity(TransformUsageFlags.Dynamic);

        // Additional entity
        var counter = CreateAdditionalEntity(TransformUsageFlags.None, "Counter");
        AddComponent(counter, new LinkedEntityGroup { Value = spawner });
        // Placeholder-to-Placeholder references → remapped to real Entities after Bake
    }
}
```

---

## 4. Constraints

| Item | Details |
|------|------|
| **No component lookup** | Calling `GetComponentData<T>()` on a Placeholder throws an error |
| **Internal prefab links** | Placeholders are only resolved within the same ECB/Baker; not shared externally |
| **Runtime access** | Until Playback completes, systems cannot use it (`SystemAPI.Exists()` returns false) |
| **Version collisions** | A version conflict between Placeholders raises a runtime exception → prefer a single ECB |

---

## 5. Best Practices

1. Create and link all related entities within the **same unit of work** (Job or Baker).  
2. Only **write data** to Placeholder entities — read it later from another system after Playback.  
3. When multiple systems must share an ECB,  
   explicitly order the **`Begin/EndSimulationECBSystem`** (`UpdateBefore/After`).  
4. Use the **Profiler ▸ Entity Timeline** "CreateDeferredEntity" event to verify count and location.  
5. Placeholders alone cannot compute **Hierarchy Transforms**, so verify local/world transform updates on the frame after Playback.

---

## 6. Practical Checklist

- [ ] No `GetComponentData` access on Placeholder entities?  
- [ ] Multiple ECBs do not duplicate-reference the same Placeholder?  
- [ ] Pre/post-Playback timing is clearly distinguished? (debug log timing)  
- [ ] `Entity.Null` and Placeholder are not being confused?  
- [ ] SyncPoint count and duration monitored in the profiler?  

> The Deferred Entity / Placeholder pattern is a core technique that enables **bulk instantiation**  
> and **complex entity relationship setup** via a single SyncPoint.  
> Used correctly, it minimizes structural-change cost while guaranteeing stable dependency and reference setup.
