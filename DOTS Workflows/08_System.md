# System Authoring Guide – **SystemBase (Managed) vs ISystem (Unmanaged)**

From Unity Entities 1.4 onward, both **SystemBase** (class-based) and **ISystem** (struct-based) styles are supported.
Use the comparison table and examples below to choose based on **performance**, **API convenience**, and **GC cost**.

---

## 1. Structural Comparison at a Glance

| Item | **SystemBase** (class) | **ISystem** (struct) |
|------|------------------------|----------------------|
| **Runtime type** | Managed object | Unmanaged value type |
| **State fields** | `class` member fields (GC Heap) | **`SystemState`** accessed via `ref SystemState state` |
| **OnCreate / OnDestroy** | Virtual methods (`protected override`) | Optional methods (`void OnCreate(ref SystemState)`, etc.) |
| **Burst default** | No (system itself cannot be Bursted; only its internal jobs) | Yes (the struct code itself is a Burst target) |
| **GC overhead** | Each field uses the GC heap | None |
| **Reflection API** | Free to use UnityEngine / Managed APIs | Unmanaged restrictions (prefer Burst-friendly APIs) |
| **Entities.ForEach** | Supported (Obsolete) | Not available |
| **IJobEntity / Query** | Fully supported | Fully supported |
| **Compile speed** | Slow (source generator + class) | Fast |
| **Use cases** | Complex gameplay, systems that depend on UnityEngine | Hot loops, heavy computation, server-side, NetCode |

---

## 2. Basic Skeletons

### 2-1 SystemBase Example
```csharp
using Unity.Entities;
using Unity.Burst;

public partial class MoveSystem : SystemBase
{
    protected override void OnCreate()
    {
        // Managed fields are allowed
        random = new Unity.Mathematics.Random(1);
    }

    protected override void OnUpdate()
    {
        float dt = Time.DeltaTime;

        Entities
            .WithNone<Paused>()
            .ForEach((ref LocalTransform t, in Velocity v) =>
            {
                t.Position += v.Value * dt;
            })
            .ScheduleParallel();
    }

    private Unity.Mathematics.Random random; // example
}
```

### 2-2 ISystem Example
```csharp
using Unity.Entities;
using Unity.Burst;

[BurstCompile]
public partial struct MoveSystemIS : ISystem
{
    public void OnCreate(ref SystemState state) { }

    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (t, v) in
                 SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>()
                          .WithNone<Paused>())
        {
            t.ValueRW.Position += v.ValueRO.Value * dt;
        }
    }

    public void OnDestroy(ref SystemState state) { }
}
```

> **Note** Inside `ISystem`, access queries, time, and singletons through **`SystemAPI`**.

---

## 3. State Management Differences

| Topic | SystemBase | ISystem |
|-------|------------|---------|
| **Member fields** | Add freely (GC heap) | Unmanaged types only (struct fields) |
| **Storage** | `this.field` | Struct fields or **Singleton Component** storage |
| **Inspector exposure** | Fields not visible in Entities Debugger | Same (no fields to expose) |

### 3-1 Storing State in ISystem
```csharp
// In ISystem, fields can be stored directly on the struct
[BurstCompile]
public partial struct MoveSystemWithState : ISystem
{
    // Store state as struct fields (only Unmanaged types allowed)
    private Unity.Mathematics.Random random;
    private float elapsedTime;
    
    public void OnCreate(ref SystemState state)
    {
        // Initialize struct fields
        random = new Unity.Mathematics.Random(1);
        elapsedTime = 0f;
    }

    public void OnUpdate(ref SystemState state)
    {
        // Use struct fields
        elapsedTime += SystemAPI.Time.DeltaTime;
        
        // For complex state, use a Singleton component
        // e.g., SystemAPI.GetSingletonRW<GameState>()
    }
}

// Or manage state via a Singleton Component
public struct SystemState : IComponentData 
{ 
    public int TickCount;
    public float AccumulatedTime;
}
```

---

## 4. Burst & Performance

* The entire `ISystem` is a **BurstCompile** target -> native optimization even with many branches and loops.
* `SystemBase` C# class code lives **outside Burst**, so hot loops must be offloaded into Burst jobs.
* For code that is called frequently or has tight frame budgets (>90 fps), prefer ISystem.

---

## 5. API Restrictions

| Category | SystemBase | ISystem |
|----------|-----------|---------|
| **UnityEngine API** | Available (`Time`, `Random.Range`, `Debug.Log`) | **Not available**, or requires Burst off |
| **Job.WithCode / Entities.ForEach** | Legacy supported | Not supported |
| **Capturing managed objects** | Can be captured inside lambdas (GC) | Not allowed, or requires Burst off |
| **NetCode Ghost** | `ISystem` preferred in Server/Client worlds | Both are possible |

---

## 6. Decision Guide

1. **Hot loops / compute-heavy** -> **ISystem**
2. Need **complex state or class references** -> **SystemBase**
3. Prioritize **convenience (fast prototyping)** -> SystemBase, then refactor to ISystem for optimization
4. Following **package sample code** -> NetCode/Physics samples are mostly ISystem-based
5. **Editor-only** - BakingSystem, Conversion phase -> use ISystem (BakingSystem)

---

## 7. Mixing Strategies

* **High-level ordering**: SystemBase and ISystem can coexist within a SystemGroup (`[UpdateInGroup]`, `[OrderFirst/Last]`).
* **Multiple modes**: GameLogicSystem (SystemBase) produces **events** -> Physics/Animation computations run in separate ISystem jobs.
* **Gradual migration**: Convert individual performance bottleneck Systems to ISystem and leave the rest unchanged.

---

## 8. Step-by-Step Guide: Migrating SystemBase to ISystem

### 8-1 Preparing for Migration

#### Step 1: Analyze the Existing SystemBase Code
```csharp
// Original SystemBase code example
public partial class WeaponSystem : SystemBase
{
    private float lastShotTime;          // State field 1
    private Dictionary<Entity, float> cooldowns; // State field 2
    private Random random;               // State field 3

    protected override void OnCreate()
    {
        random = new Random(42);
        cooldowns = new Dictionary<Entity, float>();
    }

    protected override void OnUpdate()
    {
        float currentTime = Time.ElapsedTime;
        
        // Using UnityEngine API
        if (Input.GetKeyDown(KeyCode.Space))
        {
            FireWeapon(currentTime);
        }
    }

    private void FireWeapon(float time)
    {
        // Complex state-dependent logic...
    }
}
```

#### Step 2: Analyze State & Create Singleton Components
```csharp
// Extract state into a Singleton Component
public struct WeaponSystemState : IComponentData
{
    public float LastShotTime;
    public Unity.Mathematics.Random Random;
}

public struct WeaponCooldown : IComponentData
{
    public float CooldownTime;
    public float LastFireTime;
}
```

#### Step 3: Basic Conversion to ISystem
```csharp
[BurstCompile]
public partial struct WeaponSystemIS : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        // Create the singleton entity
        var systemStateEntity = state.EntityManager.CreateEntity();
        state.EntityManager.AddComponentData(systemStateEntity, new WeaponSystemState
        {
            LastShotTime = 0f,
            Random = new Unity.Mathematics.Random(42)
        });
    }

    public void OnUpdate(ref SystemState state)
    {
        // Access singleton state
        var weaponState = SystemAPI.GetSingletonRW<WeaponSystemState>();
        float currentTime = (float)SystemAPI.Time.ElapsedTime;

        // TODO: Input handling needs to live in a separate system
        
        // Handle cooldowns via query
        foreach (var (cooldown, entity) in 
                 SystemAPI.Query<RefRW<WeaponCooldown>>().WithEntityAccess())
        {
            if (currentTime - cooldown.ValueRO.LastFireTime >= cooldown.ValueRO.CooldownTime)
            {
                // Fire logic
                FireWeapon(ref weaponState.ValueRW, cooldown.ValueRW, currentTime);
            }
        }
    }

    private void FireWeapon(ref WeaponSystemState state, RefRW<WeaponCooldown> cooldown, float time)
    {
        state.LastShotTime = time;
        cooldown.ValueRW.LastFireTime = time;
        // Additional firing logic...
    }
}
```

### 8-2 Advanced Migration Patterns

#### Complex State Management: Entity Storage Pattern
```csharp
// Store complex state on a dedicated Entity
[BurstCompile]
public partial struct ComplexSystemIS : ISystem
{
    private Entity systemDataEntity;

    public void OnCreate(ref SystemState state)
    {
        systemDataEntity = state.EntityManager.CreateEntity();
        
        // Complex data structures can be stored as Components
        state.EntityManager.AddBuffer<StateBuffer>(systemDataEntity);
        state.EntityManager.AddComponentData(systemDataEntity, new SystemConfig
        {
            MaxEntities = 1000,
            UpdateRate = 60f
        });
    }

    public void OnUpdate(ref SystemState state)
    {
        // Reference the stored Entity through SystemState
        if (!state.EntityManager.Exists(systemDataEntity))
            return;

        var config = state.EntityManager.GetComponentData<SystemConfig>(systemDataEntity);
        var stateBuffer = state.EntityManager.GetBuffer<StateBuffer>(systemDataEntity);
        
        // Perform complex logic...
    }
}
```

#### Separating Managed API Dependencies
```csharp
// Move input handling into a dedicated SystemBase
public partial class InputSystem : SystemBase
{
    protected override void OnUpdate()
    {
        // Use the UnityEngine Input API
        bool firePressed = Input.GetKeyDown(KeyCode.Space);
        
        if (firePressed)
        {
            // Deliver input events via Components
            var inputEntity = GetSingletonEntity<PlayerInputSingleton>();
            EntityManager.AddComponent<FireInputEvent>(inputEntity);
        }
    }
}

// Actual gameplay logic lives in an ISystem
[BurstCompile]
public partial struct WeaponFireSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Check for input events
        foreach (var (fireEvent, entity) in 
                 SystemAPI.Query<RefRO<FireInputEvent>>().WithEntityAccess())
        {
            // Execute firing logic
            ProcessWeaponFire(ref state);
            
            // Remove the event
            state.EntityManager.RemoveComponent<FireInputEvent>(entity);
        }
    }

    private void ProcessWeaponFire(ref SystemState state)
    {
        // Burst-compiled firing logic...
    }
}
```

### 8-3 Migration Verification Checklist

#### Compilation & Execution Checks
- [ ] No compile errors after adding the `[BurstCompile]` attribute
- [ ] All query/time/singleton access converted to `SystemAPI`
- [ ] UnityEngine API usage extracted into a separate SystemBase
- [ ] All state fields moved to Singleton Components or struct fields

#### Measuring & Comparing Performance
```csharp
// Performance measurement utility
[BurstCompile]
public partial struct PerformanceMeasureSystem : ISystem
{
    private long lastMeasureTime;

    public void OnUpdate(ref SystemState state)
    {
        long startTime = System.Diagnostics.Stopwatch.GetTimestamp();
        
        // System logic to be measured...
        
        long endTime = System.Diagnostics.Stopwatch.GetTimestamp();
        float elapsedMs = (endTime - startTime) / (float)System.Diagnostics.Stopwatch.Frequency * 1000f;
        
        // Inspect in the Unity Profiler
        Unity.Profiling.ProfilerUnsafeUtility.BeginSample("SystemPerformance");
        // Record result...
        Unity.Profiling.ProfilerUnsafeUtility.EndSample();
    }
}
```

### 8-4 Common Migration Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| **Burst compile fails** | Managed type references | Use `[BurstDiscard]` or extract to a separate SystemBase |
| **State loss** | Missing struct field initialization | Initialize explicitly in `OnCreate` |
| **Singleton access fails** | Singleton Entity not created | Verify singleton creation in `OnCreate` |
| **Performance regression** | Excessive struct copying | Use `ref` parameters |

---

## 9. Final Checklist

- [ ] Are there no Burst compile errors? (`Jobs > Burst > Safety Checks`)
- [ ] Does the SystemGroup order behave as intended? (`Dependencies` tab)
- [ ] Has GC Alloc decreased? (`Profiler > Memory`)
- [ ] Does the test world preserve the same functionality?

> **Conclusion**: **SystemBase** is about convenience; **ISystem** is about performance.
> Running a **hybrid** configuration based on project scale and performance requirements is usually the most practical approach.
