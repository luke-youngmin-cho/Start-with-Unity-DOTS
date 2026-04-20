# ECS Core Concepts – Entity / Component / System / World / Archetype

> Basic terminology and structures organized **based on Entities 1.4**.  
> Understanding each concept accurately lets you design optimal data layouts and system structures.

---

## 1. Entity
| Characteristic | Description |
|----------------|-------------|
| **ID + Tag** | In memory, a 64-bit *(index + version)* integer. It has no data of its own and **serves as a container for components**. |
| **Lightweight object** | Cheap to create/destroy. By **separating data from logic**, it is less heavy than a GameObject. |
| **Version management** | When an index is reused after deletion, the version is incremented → reference integrity is guaranteed.|

```text
┌────────────┐
│  Entity 42 │  ← Only an ID
└────────────┘
    ▲  ▲
    │  └──── Version (incremented on change)
    └─────── Index (position within the Chunk)
```

### Examples of Using the Entity Version Field

The Version field of an Entity is used to guarantee reference integrity and to detect stale references.

#### 1-1 Reference Validation
```csharp
partial struct ReferenceSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (entityRef, transform) in 
                 SystemAPI.Query<RefRO<EntityReference>, RefRW<LocalTransform>>())
        {
            Entity target = entityRef.ValueRO.Target;
            
            // Validation via Version
            if (!state.EntityManager.Exists(target))
            {
                // The referenced Entity has been destroyed - handle safely
                Debug.Log($"Invalid reference detected: {target}");
                continue;
            }
            
            // Safe component access
            if (state.EntityManager.HasComponent<LocalTransform>(target))
            {
                var targetPos = state.EntityManager.GetComponentData<LocalTransform>(target);
                transform.ValueRW.Position = targetPos.Position;
            }
        }
    }
}
```

#### 1-2 Entity Caching and Version Validation
```csharp
public partial struct EntityCacheSystem : ISystem, ISystemStartStop
{
    private Entity cachedTarget;
    
    public void OnStartRunning(ref SystemState state)
    {
        // Cache a specific Entity
        cachedTarget = SystemAPI.GetSingletonEntity<PlayerTag>();
    }
    
    public void OnUpdate(ref SystemState state)
    {
        // Validate the cached Entity
        if (cachedTarget == Entity.Null || !state.EntityManager.Exists(cachedTarget))
        {
            // Cache needs to be refreshed
            var query = SystemAPI.QueryBuilder().WithAll<PlayerTag>().Build();
            if (query.TryGetSingletonEntity<PlayerTag>(out cachedTarget))
            {
                Debug.Log($"Updated cached entity: {cachedTarget} (v{cachedTarget.Version})");
            }
        }
        
        // Use the cached Entity
        if (cachedTarget != Entity.Null)
        {
            var playerPos = state.EntityManager.GetComponentData<LocalTransform>(cachedTarget);
            // Logic based on player position...
        }
    }
    
    public void OnStopRunning(ref SystemState state) { }
}
```

#### 1-3 Tracking Entity Lifecycle
```csharp
[System.Serializable]
public struct EntityTracker : IComponentData
{
    public Entity TrackedEntity;
    public int LastKnownVersion;  // Version tracking
}

partial struct EntityTrackingSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (tracker, entity) in 
                 SystemAPI.Query<RefRW<EntityTracker>>().WithEntityAccess())
        {
            var currentEntity = tracker.ValueRO.TrackedEntity;
            
            if (state.EntityManager.Exists(currentEntity))
            {
                int currentVersion = currentEntity.Version;
                
                // Detect version change (confirms re-creation)
                if (currentVersion != tracker.ValueRO.LastKnownVersion)
                {
                    tracker.ValueRW.LastKnownVersion = currentVersion;
                    Debug.Log($"Entity {currentEntity.Index} was recreated (v{currentVersion})");
                }
            }
            else
            {
                // The Entity has been destroyed
                Debug.Log($"Tracked entity {currentEntity} no longer exists");
                tracker.ValueRW.TrackedEntity = Entity.Null;
                tracker.ValueRW.LastKnownVersion = 0;
            }
        }
    }
}
```

> **Version field usage principles**
> 1. Check reference validity with **Entity.Exists()**
> 2. Detect Entity re-creation via **Version comparison**
> 3. Guarantee the integrity of **cached Entity references**

---

## 2. Component
| Category | Example | Memory Characteristics |
|----------|---------|------------------------|
| **IComponentData**<br>(Unmanaged) | `LocalTransform`, `Velocity` | Struct ≤ 16 KB, Burst-friendly |
| **IEnableableComponent** | `PhysicsCollider` (on/off) | Toggles Enable state via BitMask (no StructuralChange) |
| **IBufferElementData** | `DynamicBuffer<Waypoint>` | Variable-length array at the end of the Chunk |
| **ISharedComponentData** | `RenderMeshArray` index | Entities with identical values are split into separate Chunks (fast filtering) |
| **ManagedComponent** | `UnityEngine.Mesh` reference | GC Heap, not Burst-compatible – minimize use |

> **Invariant rule**  
> *Components must hold only pure data* and must contain no logic.

---

## 3. System
| Type | Characteristics | Typical Use |
|------|-----------------|-------------|
| **ISystem** (struct) | Unmanaged, Burst supported by default, few state fields | Respawn, movement calculations |
| **SystemBase** (class) | Managed state, rich API, supports OnCreate/Destroy | NetCode, streaming |
| **SystemGroup** | Subgroup with `UpdateInGroup`, `OrderFirst/Last` attributes | `SimulationSystemGroup` |
| **BakingSystem** | Dedicated to the baking pipeline, runs in the Editor only | Geometry optimization |

### System Execution Order Flow
```text
InitializationSystemGroup
    ├─ [..]
SimulationSystemGroup
    ├─ PhysicsSystemGroup
    │   ├─ PhysicsBuild
    │   └─ PhysicsStep
    ├─ MyGameplaySystem
    └─ EndSimulationEntityCommandBufferSystem
PresentationSystemGroup
```

---

## 4. World
| Property | Description |
|----------|-------------|
| **Container of entities and systems** | Each World contains an independent EntityDB + SystemGroup set |
| **Multiple Worlds** | PlayMode defaults: `DefaultWorld` (game), `ConversionWorld` (editor bake) |
| **Singleton data** | Global settings can be passed via `SystemAPI.GetSingleton<T>()` |

> **Tip** Directly moving entities between Worlds is not possible. EntityScene streaming or duplication is required.

---

## 5. Archetype & Chunk
### 5‑1 Archetype
* **The signature of a component set**  
  Entities with the same list of components are grouped into the same **Archetype**.
* **Dedicated memory pool** – changing the classification triggers a Structural Change.

### 5‑2 Chunk
| Item | Value |
|------|-------|
| **Size** | 16 KB by default (varies by platform) |
| **Layout** | Entities of the same Archetype are laid out contiguously in **SoA** form |
| **Access** | Burst-friendly access via the `ArchetypeChunk` struct |
| **Chunk Header** | Contains metadata such as Enable Mask and Version |

```text
┌───────────────────────────────┐  ◄ Chunk (16 KB)
│ Entity0 | Position0 | Velocity0│
│ Entity1 | Position1 | Velocity1│  ← stored as columns, one per component
│   ...   |    ...    |    ...   │
└───────────────────────────────┘
```

#### Cache Optimization via SoA Memory Layout

**Actual memory layout (Position + Velocity Archetype)**
```text
┌─────────── Chunk Header (64-128B) ───────────┐
│ Entity Count | Enable Masks | Version Info  │
├─────────────── Component Arrays ─────────────┤
│ Position[0] Position[1] ... Position[N]     │ ← contiguous layout
│ Velocity[0] Velocity[1] ... Velocity[N]     │ ← contiguous layout
│ DynamicBuffer pointers (if needed)          │
└──────────────────────────────────────────────┘

Example memory addresses:
Position[0]: 0x1000  Position[1]: 0x100C  Position[2]: 0x1018
Velocity[0]: 0x2000  Velocity[1]: 0x200C  Velocity[2]: 0x2018
         ↑ Same components stored contiguously (SIMD optimization possible)
```

**Direct SoA access in IJobChunk**
```csharp
public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
{
    var positions = chunk.GetNativeArray(ref positionHandle);
    var velocities = chunk.GetNativeArray(ref velocityHandle);
    
    // Cache-friendly: iterate sequentially over arrays of the same type
    for (int i = 0; i < chunk.Count; i++)
    {
        positions[i] = new LocalTransform
        {
            Position = positions[i].Position + velocities[i].Value
        };
    }
}
```

> **Performance principles**  
> 1. **Contiguous memory access** → maximizes cache efficiency  
> 2. **SIMD vectorization** → can process 4–8 elements simultaneously
> 3. **Queries** filter by Archetype → minimizes unnecessary checks

---

## 6. Key Takeaways
1. An **Entity** is an ID, a **Component** is pure data, and a **System** is logic.  
2. A **World** is the container that holds all of these and also the scheduler that runs the systems.  
3. The **Archetype + Chunk** structure groups entities with similar data to maximize CPU cache hit rates.  
4. For performance, avoid **unnecessary Managed Components** and design so that Queries exploit **Archetype-contiguous memory**.  
5. Define execution order and dependencies clearly with `SystemGroup`, `UpdateBefore/After` to minimize **contention and Sync Points**.
