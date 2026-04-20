# ECS Workflow Quick Start  
### SubScene Creation → Baker Conversion Flow

---

## 1. Overview
* A **SubScene** is a container for converting (baking) GameObject-based "Authoring data" into ECS data.  
* A **Baker** reads values from MonoBehaviour (Authoring Component) and creates/attaches **Entities** and **ECS Components**.  
* Core workflow steps:  
  1. Place GameObjects inside a SubScene  
  2. Close the SubScene → automatic baking  
  3. Baker code creates Entities / Components  
  4. (Optional) Baking System runs post-processing  
  5. The result is saved to an Entity Scene file or applied to the live world  

---

## 2. Creating a SubScene

| Scenario | Procedure |
|----------|-----------|
| **Add a new SubScene** | Right-click in the Hierarchy window → select **New Sub Scene ▸ Empty Scene**, then save |
| **Create a SubScene from an existing GameObject** | Select the target GameObject in the Hierarchy → right-click → **New Sub Scene ▸ From Selection** |
| **Add a reference to an existing SubScene** | Create an empty GameObject → add a **SubScene** component → assign a **Scene Asset** |

> **Tip** Enabling the **Auto Load Scene** checkbox on the SubScene component automatically streams it at playtime.

### Open / Close
* **Open** state  
  * The Authoring GameObject tree is shown in the Hierarchy  
  * Runtime data can be inspected depending on the Scene View Mode (Entities / GameObjects)  
  * After the initial bake, an **Incremental Bake runs on every change**  
* **Closed** state  
  * The baked Entity Scene is streamed and collapsed in the Hierarchy  
  * Entities become available a few frames after entering Play mode  
  * Unlike the Open state, there is streaming latency

---

## 3. Understanding the Baking Process

```text
Authoring GameObject
        │  (SubScene closed)
        ▼
┌─────────────┐
│ Entity created │  (metadata only)
└─────────────┘
        ▼
┌─────────────┐
│  Baker stage  │  (MonoBehaviour → ECS Component)
└─────────────┘
        ▼
┌─────────────┐
│ Baking Systems│ (optional post-processing)
└─────────────┘
        ▼
Save to Entity Scene / apply to live world
```

* **Incremental Bake**  
  * Only changed GameObjects and their dependencies are re-baked → real-time preview  
  * To guarantee result consistency, Bakers/Baking Systems must **not cache state**; they must always compute results solely from inputs  
* **Full Bake**  
  * Converts the entire scene from scratch (disk round-trip)

---

## 4. Writing an Authoring Component & Baker

### 4‑1 Basic Example

```csharp
// MonoBehaviour Authoring
public class RotationSpeedAuthoring : MonoBehaviour
{
    public float DegreesPerSecond;
}

// ECS Component
public struct RotationSpeed : IComponentData
{
    public float RadiansPerSecond;
}

// Additional Component definition
public struct AdditionalEntity : IComponentData
{
    public int SomeValue;
}

// Baker
public class SimpleBaker : Baker<RotationSpeedAuthoring>
{
    public override void Bake(RotationSpeedAuthoring authoring)
    {
        // Authoring GameObject → Entity
        var entity = GetEntity(TransformUsageFlags.Dynamic);

        // 1) Add a component to the primary Entity
        AddComponent(entity, new RotationSpeed {
            RadiansPerSecond = math.radians(authoring.DegreesPerSecond)
        });

        // 2) Create an additional Entity & attach a component (optional)
        var child = CreateAdditionalEntity(TransformUsageFlags.Dynamic, "Child");
        AddComponent(child, new AdditionalEntity { SomeValue = 123 });
    }
}
```

### 4‑2 Core Rules
| Item | Description |
|------|-------------|
| **No state preservation** | A Baker instance is invoked repeatedly for multiple Authoring Components, so **do not cache in member variables** |
| **Modify only your own Entity** | Do not change Entities/Components created by other Bakers |
| **Declare dependencies** | External reference data (other GameObjects, Assets, etc.) must be **tracked** via `DependsOn()` or the Baker-specific `GetComponent<T>()` call |
| **Static / unordered execution** | Baker execution order is undefined, so **do not write logic with mutual dependencies** |

#### Dependency Example (summary)
```csharp
public class DependentDataAuthoring : MonoBehaviour
{
    public GameObject Other;
    public Mesh Mesh;
}

public class GetComponentBaker : Baker<DependentDataAuthoring>
{
    public override void Bake(DependentDataAuthoring authoring)
    {
        DependsOn(authoring.Other);
        DependsOn(authoring.Mesh);

        if (authoring.Other == null || authoring.Mesh == null) return;

        var entity = GetEntity(TransformUsageFlags.Dynamic);
        // ...
    }
}
```

---

## 5. Hands-on Checklist

1. **Create a SubScene** and place GameObjects inside  
2. Close the SubScene and verify the bake result (Entity Scene)  
3. Write an Authoring Component + Baker script → apply the scripts  
4. Inspect Entities and the Baking Preview in the **Entities Hierarchy** window  
5. Modify values → verify **Incremental Bake** behavior  
6. In Play mode, check that entities exist and behave correctly  

---

## 6. Common Issues & Solutions

| Symptom | Cause / Solution |
|---------|------------------|
| Baker does not re-run | Missing `DependsOn()` when accessing external data |
| Values reset after entering Play mode | Changes in SubScene Open state apply in real time; when Closed, expect a few frames of startup delay |
| Duplicate components created | Baker calls `AddComponent` multiple times – check conditional statements |
| Performance degradation | Too much data in a single SubScene → split into multiple SubScenes |

---

### Done!  
Follow this guide to experience the **SubScene → Baker → Entity** conversion flow. After completing a small example, progressively apply component/system extensions, Scene Streaming, and Baking Systems so you can apply them directly to real projects.
