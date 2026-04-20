# System Group & Update Order Attributes
### Explicitly control system execution order

Unity ECS determines system execution order through **System Groups** and a variety of **Update Order attributes**.
Used properly, they help you achieve predictable behavior without dependency errors or frame delays.

---

## 1. Default System Group Hierarchy

| Group | Role | Default Sub-groups |
|-------|------|--------------------|
| **InitializationSystemGroup** | Scene loading, singleton initialization | `BeginInitializationECBSystem` |
| **SimulationSystemGroup** | Game logic, physics, AI | `FixedStepSimulationSystemGroup`<br>`PhysicsSystemGroup`<br>`EndSimulationECBSystem` |
| **PresentationSystemGroup** | Prepare render data | `LateSimulationSystemGroup` |
| **FixedStepSimulationSystemGroup** | Fixed-interval updates (default 60Hz) | `BeginFixedStepECBSystem` / `EndFixedStepECBSystem` |
| **TransformSystemGroup** *(Entities Graphics)* | LocalTransform computation | Internal use |

> At **World Bootstrap** time, `DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups()` automatically creates the groups above and classifies each system into them.

---

## 2. Update Order Attributes

| Attribute | Target | Function |
|-----------|--------|----------|
| `[UpdateInGroup(typeof(G))]` | **System / Group** | Place the system inside group `G` |
| `[UpdateBefore(typeof(S))]` | System | Runs **before S** within the same group |
| `[UpdateAfter(typeof(S))]` | System | Runs **after S** within the same group |
| `[OrderFirst]` | System | Runs first in the group |
| `[OrderLast]` | System | Runs last in the group |

### Priority Resolution Rules
1. `OrderFirst` > `UpdateBefore/After` > `OrderLast` > registration order
2. If a conflict (cycle) occurs, Unity emits a warning log and sorts based on **registration order**

---

## 3. Code Examples

### 3-1 Defining & Inserting a Group
```csharp
// Custom group - for all raycast processing
[UpdateInGroup(typeof(SimulationSystemGroup))]
[OrderFirst]                         // Runs first in the Simulation group
public partial class RaycastGroup : ComponentSystemGroup { }
```

### 3-2 Placing Systems
```csharp
[UpdateInGroup(typeof(RaycastGroup))]
public partial struct CollectRaycastRequestsSystem : ISystem { /* ... */ }

[UpdateInGroup(typeof(RaycastGroup))]
[UpdateAfter(typeof(CollectRaycastRequestsSystem))]
public partial struct ExecuteRaycastsSystem : ISystem { /* ... */ }

[UpdateInGroup(typeof(RaycastGroup))]
[UpdateAfter(typeof(ExecuteRaycastsSystem))]
public partial struct ApplyRaycastResultsSystem : ISystem { /* ... */ }
```

### 3-3 OrderFirst / OrderLast
```csharp
[UpdateInGroup(typeof(SimulationSystemGroup))]
[OrderLast]                         // Remove buff effects at the end of the frame
public partial struct ExpireBuffSystem : ISystem { /* ... */ }
```

---

## 4. Using FixedStepSimulation

```csharp
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
public partial struct PhysicsStepSystem : ISystem
{
    void OnCreate(ref SystemState s)
    {
        // Custom fixed timestep (e.g., 0.02f -> 50Hz)
        s.WorldUnmanaged.ResolveSystemStateRef<FixedStepSimulationSystemGroup>()
          .Timestep = 0.02f;
    }
}
```

* **FixedStepSimulationSystemGroup** can run multiple updates when the accumulated `TimeAccumulator` exceeds the Timestep, which makes it ideal for logic driven by **FixedDeltaTime** (physics/network ticks).

---

## 5. Systems Window & Dependencies Tab

1. Open **Entities Debugger (Windows > Entities > Systems Window)**
2. **Group Tree**: Visualizes the group/system structure of the current world
3. **Dependencies tab**: Graphically displays `UpdateBefore/After` relationships and job dependency graphs
4. Conflicts/cycles are highlighted in red - adjust attributes accordingly

> **Tip** Check the **Frame Time** column next to group names during runtime to identify bottleneck systems.

---

## 6. Best Practices

| Situation | Recommended Strategy |
|-----------|---------------------|
| Bundling complex subordinate logic | Define a **custom group** and chain child systems with UpdateAfter |
| Physics -> game logic order | Place **GameplaySystemGroup** with UpdateAfter on `PhysicsSystemGroup` |
| Render preprocessing | Put render-prep systems at the front of the Presentation group (`OrderFirst`) |
| Circular dependency issues | Split the system or force priority using `OrderFirst/Last` |
| Multi-world (NetCode) | Manage `ServerSimulationSystemGroup` and `ClientSimulationSystemGroup` independently |

---

## 7. Checklist

- [ ] Is every system in the **correct group**? (`UpdateInGroup`)
- [ ] Do any **cycles** arise from `UpdateBefore/After`? (check via the Systems Window)
- [ ] Is frame-order-sensitive logic placed in `FixedStepSimulationSystemGroup`?
- [ ] After `BurstCompile`-ing or jobifying bottleneck systems, have you readjusted their order?

> **Accurate grouping and explicit execution order** greatly improve **debuggability, performance, and collaboration** quality in ECS projects. Use this guide as a basis to tidy up your own system graph.
