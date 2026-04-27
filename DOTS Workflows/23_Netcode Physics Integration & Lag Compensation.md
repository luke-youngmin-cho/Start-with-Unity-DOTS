---
title: Netcode Physics Integration & Lag Compensation
updated: 2026-04-27
folder: DOTS Workflows
---

# Netcode Physics Integration & Lag Compensation
### Unity 6000.5 · Entities 6.5.0 · Netcode for Entities 6.5.0

---

## 1. Overview

Netcode for Entities integrates with **Unity Physics** for networked physics simulation. Physics behavior depends on whether the Ghost is Interpolated or Predicted.

At least one Predicted Ghost must exist in the scene for predicted physics to run.

---

## 2. Interpolated vs Predicted physics

### 2.1 Interpolated Ghosts on clients

- Position and rotation come from server snapshots.
- `Simulate` is disabled, so the physics object behaves as kinematic.
- `PhysicsVelocity` is ignored.

### 2.2 Predicted Ghosts on clients

- Physics runs inside the prediction loop and can execute multiple times per rendered frame.
- `PhysicsSystemGroup` runs inside `PredictedFixedStepSimulationSystemGroup`.
- The client is expected to produce the same simulation result as the server for predicted physics.

```text
Netcode initialization
├─ PhysicsSystemGroup moves under PredictedFixedStepSimulationSystemGroup
└─ FixedStepSimulationSystemGroup moves under PredictedFixedStepSimulationSystemGroup
```

---

## 3. Predicted physics performance

Predicted physics is expensive because rollback can execute physics repeatedly.

### 3.1 Server batching

```csharp
using Unity.NetCode;

ClientServerTickRate tickRate = default;
tickRate.MaxSimulationStepBatchSize = 4;
tickRate.MaxSimulationStepsPerFrame = 4;
```

### 3.2 Client batching

```csharp
using Unity.NetCode;

ClientTickRate clientRate = default;
clientRate.MaxPredictionStepBatchSizeFirstTimeTick = 2;
clientRate.MaxPredictionStepBatchSizeRepeatedTick = 4;
```

Batching can increase the risk of misprediction, so use it as a performance tradeoff rather than a default.

### 3.3 Quantization

The default Transform / Velocity quantization for physics Ghosts is `1000`.

| Higher quantization | Lower quantization |
|---------------------|--------------------|
| More precise simulation. | Lower bandwidth. |
| Higher bandwidth. | More precision loss. |

---

## 4. Lag compensation

Because clients predict ahead of the server, the authoritative server sometimes needs to query an older collision world that matches the client's action time.

### 4.1 Enable lag compensation

Add `NetCodePhysicsConfig` to a SubScene and set `EnableLagCompensation = true`.

### 4.2 Query an old collision world

```csharp
using Unity.NetCode;
using Unity.Physics;

var physicsHistory = SystemAPI.GetSingleton<PhysicsWorldHistorySingleton>();

physicsHistory.GetCollisionWorldFromTick(
    serverTick,
    interpolationDelay,
    ref physicsWorld,
    out var collisionWorld);

// Use collisionWorld for raycasts or other physics queries.
```

### 4.3 Collider deep cloning

| Setting | Default | Meaning |
|---------|---------|---------|
| `DeepCopyDynamicColliders` | `true` | Deep clone dynamic colliders; default in 6.5.0. |
| `DeepCopyStaticColliders` | `false` | Deep clone static colliders. |

The 6.5.0 line enables dynamic Ghost collider deep cloning by default to avoid blob asset assertions.

### 4.4 Physics history buffer

| Version | History size |
|---------|--------------|
| Up to 1.4.x | 16 ticks. |
| 1.5.0+ | 32 ticks. |
| Custom | `GhostSystemConstants.SnapshotHistorySize` via compiler define. |

---

## 5. Multi Physics World

Use a separate physics world for client-only effects such as VFX or particles that do not participate in networked simulation.

### 5.1 Setup

1. Add `NetCodePhysicsConfig` to the SubScene.
2. Set **Client Non Ghost World** to a non-zero value, such as `1`.
3. Add `PhysicsWorldIndex` with the same value to client-only physics objects.

### 5.2 Custom physics system group

```csharp
using Unity.Entities;
using Unity.NetCode;
using Unity.Physics.Systems;

[WorldSystemFilter(WorldSystemFilterFlags.ClientSimulation)]
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
public partial class VfxPhysicsGroup : CustomPhysicsSystemGroup
{
    public const int WorldIndex = 1;

    public VfxPhysicsGroup() : base(WorldIndex, shareStaticColliders: true)
    {
    }
}
```

`shareStaticColliders: true` lets the custom physics world share static colliders with the main physics world.

---

## 6. Physics Proxy

Use a Physics Proxy when a Predicted Ghost must interact with a client-only physics entity.

- `CustomPhysicsProxyAuthoring` creates the proxy entity.
- `CustomPhysicsProxyDriver` links the proxy to the Predicted Ghost.
- `SyncCustomPhysicsProxySystem` synchronizes position and rotation.

---

## 7. `PredictedFixedStepSimulationSystemGroup`

`PredictedFixedStepSimulationSystemGroup` updates only when all required conditions are met.

| Requirement | Meaning |
|-------------|---------|
| `NetworkStreamInGame` singleton exists | The connection is in-game. |
| At least one Predicted Ghost exists | Predicted physics has something to simulate. |
| Fixed tick rate runs | Fixed-timestep simulation is active. |

Before a network connection is in-game or before predicted Ghosts stream in, predicted physics does not run.

### 7.1 Physics before connection

If physics must run before a network connection exists, disable predicted physics setup so normal `FixedStepSimulationSystemGroup` physics can run.

```csharp
using Unity.Entities;
using Unity.NetCode;

[UpdateInGroup(typeof(InitializationSystemGroup))]
[CreateAfter(typeof(PredictedPhysicsConfigSystem))]
public partial struct DisablePredictedPhysics : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        state.World.GetExistingSystemManaged<PredictedPhysicsConfigSystem>()
            .Enabled = false;
    }
}
```

---

## 8. Limitations

| Limitation | Meaning |
|------------|---------|
| Partial tick unsupported | Physics does not use partial ticks; use Physics Interpolation. |
| Multi-world debug limits | Unity Physics debug systems do not fully support multi-world setups. |
| Minimum Predicted Ghost | At least one Predicted Ghost is required for predicted physics to run. |

---

## 9. Related docs

- [`19_Netcode Prediction & Rollback.md`](19_Netcode Prediction & Rollback.md) — prediction loop behavior.
- [`22_Netcode Ghost Optimization · Importance · Relevancy.md`](22_Netcode Ghost Optimization · Importance · Relevancy.md) — switching Ghosts away from prediction.
- [`24_Netcode Profiler & Debugging.md`](24_Netcode Profiler & Debugging.md) — profiling predicted physics.

---

## 10. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Physics does not run | Check for a Predicted Ghost and `NetworkStreamInGame`. |
| Physics jitters | Check quantization and prediction smoothing. |
| Lag-compensation raycast fails | Enable `EnableLagCompensation` and confirm history buffer coverage. |
| Blob asset assertion | Check collider deep-clone settings; dynamic clone is enabled by default in 6.5.0. |
| Client-only physics does not run | Match `PhysicsWorldIndex` to the **Client Non Ghost World** value. |
| Predicted physics CPU is too high | Enable batching and switch unnecessary Ghosts to Interpolated. |
