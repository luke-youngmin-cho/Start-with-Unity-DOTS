# Singleton Component Pattern
### Safely handle global game state the ECS way

> A **Singleton Component** is a component that exists **exactly once** within a given **World**.
> It lets you migrate settings, scores, random seeds, and other global state that traditionally used `static` variables into a **data-oriented** form that is easy to consolidate, serialize, and test.

---

## 1. Basic Concept

| Item | Description |
|------|-------------|
| **Uniqueness** | Use `EntityQueryOptions.IncludeDisabledEntities` + **assert** to keep exactly one instance |
| **Storage** | Regular Unmanaged Component -> stored inside Chunks |
| **Access** | `SystemAPI.GetSingleton<T>()`, `SystemAPI.SetSingleton<T>()` |
| **Creation timing** | Create an Entity in `SystemBase.OnCreate()` or in a Baker |
| **Advantages** | Simple global state, DOTS serialization, job-safe access |
| **Caveats** | Destroying the singleton **Entity at runtime causes errors** -> do not Remove during gameplay |

---

## 2. Declaration & Initialization

### 2-1 Data Structure
```csharp
public struct GameSettings : IComponentData
{
    public int   MaxLives;
    public float GlobalGravity;
}
```

### 2-2 Creating the Entity (System)
```csharp
public partial class BootstrapSettingsSystem : SystemBase
{
    protected override void OnCreate()
    {
        if (!SystemAPI.HasSingleton<GameSettings>())
        {
            Entity settings = EntityManager.CreateEntity();
            EntityManager.AddComponentData(settings, new GameSettings
            {
                MaxLives       = 3,
                GlobalGravity  = -9.81f
            });
        }
    }
    protected override void OnUpdate() { }
}
```

> **Tip** When multiple worlds exist (e.g., server/client), create a separate singleton for each world.

---

## 3. Read / Write Patterns

### 3-1 Read-only
```csharp
public partial struct ApplyGravityJob : IJobEntity
{
    [ReadOnly] public GameSettings Settings;

    void Execute(ref Velocity vel)
    {
        vel.Value.y += Settings.GlobalGravity * SystemAPI.Time.DeltaTime;
    }
}

// System
var job = new ApplyGravityJob
{
    Settings = SystemAPI.GetSingleton<GameSettings>()
}.ScheduleParallel();
```

### 3-2 Write
```csharp
public partial struct DebugKeyInputSystem : ISystem
{
    void OnUpdate(ref SystemState state)
    {
        if (Input.GetKeyDown(KeyCode.L))
        {
            var settings = SystemAPI.GetSingleton<GameSettings>();
            settings.MaxLives++;
            SystemAPI.SetSingleton(settings);
        }
    }
}
```

* **Note**: `SetSingleton` cannot be called inside a Job -> handle it in a main-thread system.

---

## 4. Multiple Fields vs Multiple Singletons

| Situation | Recommended Approach |
|-----------|---------------------|
| <= 5 values & low update frequency | Group into a **single singleton** (struct fields) |
| >= 10 fields & varied change frequency | **Split singletons by domain** (e.g., `GameConfig`, `AudioSettings`) |
| Large arrays / lists | Use a separate **BlobAsset** or DynamicBuffer |

---

## 5. Testing & Refactoring

1. **World-level testing**
   * Create a test world and configure the environment with `SystemAPI.SetSingleton`.
2. **Data-driven design**
   * Convert singleton initial values from ScriptableObject via Baker.
3. **Runtime mutability**
   * Register as a NetCode **Ghost Component** when network synchronization is required.

---

## 6. Checklist

- [ ] Prevent duplicate creation with `SystemAPI.HasSingleton<T>()`
- [ ] Inside jobs, use only `GetSingletonRO<T>()` (read-only)
- [ ] Do not destroy the singleton Entity - overwrite its fields to reinitialize instead
- [ ] Maintain an independent singleton per world (multi-world architectures)
- [ ] If values change frequently, consider replacing with **IEnableableComponent**, Buffer, or Event

> Applying the Singleton Component pattern correctly lets you safely share **data-oriented global state** while reducing `static` code dependencies, improving testability and multithreading compatibility.
