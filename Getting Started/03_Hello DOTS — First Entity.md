---
title: Hello DOTS — First Entity
updated: 2026-04-21
folder: Getting Started
---

# Hello DOTS — First Entity
### Unity 6000.5 · Entities 6.5.0

---

## 1. What you will build

A cube that rotates on its Y axis, driven by an ECS system. In one sitting you will touch every core concept: **SubScene**, **authoring → baker**, **`IComponentData`**, **`ISystem`**, and the **Entities Hierarchy / Systems** windows. At the end you will swap the main-thread system for an **`IJobEntity`** to see how parallelism enters.

Prerequisite: [`01_Environment Setup.md`](01_Environment Setup (Unity 6000.5 + Entities 6.5.0).md) — a Unity 6000.5 project with a confirmed SubScene and Entities menu.

---

## 2. Create the SubScene

1. Open your empty scene.
2. In the **Hierarchy**, right-click → **New Sub Scene → Empty Scene…**
3. Save it as `HelloDots.unity` inside `Assets/Scenes/`.
4. Inside the new SubScene, right-click → **3D Object → Cube**. Rename it to `RotatingCube`.
5. Save the scene (`Ctrl/⌘ + S`). Close the SubScene by clicking the checkbox next to its name in the Hierarchy — the icon goes yellow, which means its contents are being baked to entities.

> From this point on, modifications to `RotatingCube`'s components trigger a re-bake automatically when you reopen/close the SubScene.

---

## 3. Define the component — `RotationSpeed`

Create `Assets/Scripts/RotationSpeed.cs`:

```csharp
using Unity.Entities;

public struct RotationSpeed : IComponentData
{
    public float RadiansPerSecond;
}
```

Notes:
- `IComponentData` on an **unmanaged** `struct` is the default pick — it lives directly in chunks and is Burst-friendly.
- Keep the fields blittable (primitives, `Unity.Mathematics` types, other unmanaged structs).

---

## 4. Define the authoring + baker

Create `Assets/Scripts/RotationSpeedAuthoring.cs`:

```csharp
using Unity.Entities;
using Unity.Mathematics;
using UnityEngine;

public class RotationSpeedAuthoring : MonoBehaviour
{
    public float DegreesPerSecond = 90f;

    class Baker : Baker<RotationSpeedAuthoring>
    {
        public override void Bake(RotationSpeedAuthoring authoring)
        {
            // Dynamic — this entity will have its transform written each frame.
            var entity = GetEntity(TransformUsageFlags.Dynamic);

            AddComponent(entity, new RotationSpeed
            {
                RadiansPerSecond = math.radians(authoring.DegreesPerSecond)
            });
        }
    }
}
```

### Attach the authoring to the cube

1. Select `RotatingCube` in the SubScene.
2. **Inspector → Add Component → Rotation Speed Authoring**.
3. Leave `DegreesPerSecond = 90`.

As soon as the SubScene is closed (yellow icon), the baker runs and the cube is turned into an entity with a `RotationSpeed` component on it.

### Verify the bake

- **Window → Entities → Hierarchy** — you should see `RotatingCube` listed.
- Click it. The **Inspector** on the right shows its components — look for `RotationSpeed { RadiansPerSecond: ~1.5708 }` (π/2).

---

## 5. Write the system — `RotationSystem`

Create `Assets/Scripts/RotationSystem.cs`:

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

[BurstCompile]
public partial struct RotationSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (transform, speed) in
                 SystemAPI.Query<RefRW<LocalTransform>, RefRO<RotationSpeed>>())
        {
            transform.ValueRW = transform.ValueRO.RotateY(speed.ValueRO.RadiansPerSecond * dt);
        }
    }
}
```

Notes:
- `partial struct` + `ISystem` is the **unmanaged system** form. It can be Burst-compiled and runs without GC overhead.
- `SystemAPI.Query<RefRW<A>, RefRO<B>>()` is the modern iteration API — this replaces the deprecated `Entities.ForEach`.
- `LocalTransform.RotateY(radians)` returns a rotated copy; we write it back with `transform.ValueRW = …`.

### Press Play

The cube rotates. In **Window → Entities → Systems** you can see `RotationSystem` listed under the simulation group with a per-frame timing.

---

## 6. Inspect what you built

Useful windows once you are in Play mode:

| Window | What it shows |
|--------|--------------|
| **Entities Hierarchy** | Live list of entities. Click any entity to see its chunk + components. |
| **Systems** | All running systems, grouped by their `SystemGroup`. Timing per system is visible here. |
| **Archetypes** | Chunks grouped by component signature. Useful for spotting fragmentation. |

Take a moment to confirm:
1. The `RotatingCube` entity exists and has `LocalTransform` + `RotationSpeed`.
2. `RotationSystem` is present under `SimulationSystemGroup` and accumulates frame time.

---

## 7. Swap to `IJobEntity` (parallelism)

Replace `RotationSystem.cs` with a job-based version to see how parallelism plugs in:

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

[BurstCompile]
public partial struct RotateJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref LocalTransform transform, in RotationSpeed speed)
    {
        transform = transform.RotateY(speed.RadiansPerSecond * DeltaTime);
    }
}

[BurstCompile]
public partial struct RotationSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        new RotateJob { DeltaTime = SystemAPI.Time.DeltaTime }
            .ScheduleParallel();
    }
}
```

Notes:
- `IJobEntity.Execute(ref A, in B)` is source-generated into a chunk-level job. The parameters are the components you touch.
- `.ScheduleParallel()` distributes chunks across worker threads. For a single cube you will not see a speedup, but the shape of the code scales to thousands of entities without change.
- The system's dependency is handled for you — `state.Dependency` is combined automatically when you call `ScheduleParallel()` from an `ISystem.OnUpdate`.

---

## 8. What to read next

| If you want to… | Read |
|----------------|------|
| Learn how authoring maps to entities in detail | [`DOTS Workflows/01_Baker Pattern & SubScene.md`](../DOTS Workflows/01_Baker Pattern & SubScene.md) |
| Spawn many entities at runtime | [`DOTS Workflows/02_Spawner Example.md`](../DOTS Workflows/02_Spawner Example.md) |
| Understand Entity / Component / Chunk | [`DOTS Workflows/03_ECS Core Concepts.md`](../DOTS Workflows/03_ECS Core Concepts.md) |
| Choose between `ISystem` and `SystemBase` | [`DOTS Workflows/08_System — ISystem vs SystemBase.md`](../DOTS Workflows/08_System — ISystem vs SystemBase.md) |

---

## 9. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Cube does not appear in **Entities Hierarchy** | The SubScene is still open (white icon, not yellow). Close it via the checkbox in the Hierarchy. |
| Cube appears but has no `RotationSpeed` component | `RotationSpeedAuthoring` was not attached, or the SubScene wasn't re-baked after you added it. Toggle the SubScene checkbox off and on. |
| Cube does not rotate | `RotationSystem` is not present in the Systems window. Most common causes: missing `partial` keyword on the struct, or the script contains a compile error. |
| Compile error: `TransformUsageFlags does not exist` | `using Unity.Entities;` is missing from the authoring file. |
| Compile error: `math.radians does not exist` | `using Unity.Mathematics;` is missing. |
| `SystemAPI.Query<...>` not found | Missing `using Unity.Entities;`, or the struct is not marked `partial`. |
| Inspector shows `LocalTransform` missing on the entity | The baker called `GetEntity(TransformUsageFlags.None)` instead of `Dynamic`. Change the flag. |
| `ScheduleParallel()` throws "race condition on LocalTransform" | Another system writes to `LocalTransform` in the same frame without ordering. Add `[UpdateBefore]`/`[UpdateAfter]` or use a shared `EntityCommandBuffer`. |
