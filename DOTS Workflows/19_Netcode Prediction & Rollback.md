---
title: Netcode Prediction & Rollback
updated: 2026-04-27
folder: DOTS Workflows
---

# Netcode Prediction & Rollback
### Unity 6000.5 · Entities 6.5.0 · Netcode for Entities 6.5.0

---

## 1. Overview

Client prediction runs the same simulation code on the client before the authoritative server result arrives. This reduces input latency and makes player-owned movement feel immediate.

Prediction has a real CPU cost: when an authoritative snapshot arrives, the client can roll back to an older tick and re-simulate forward to the current predicted tick.

---

## 2. Core components and systems

| Component / system | Meaning |
|--------------------|---------|
| `PredictedGhost` | Added to predicted Ghosts on clients and to all Ghosts on the server. |
| `Simulate` | Enableable tag set on entities that should simulate for the current prediction tick. |
| `PredictedSimulationSystemGroup` | Fixed-timestep group that runs the prediction loop. |
| `NetworkTime` | Holds server tick, prediction tick, and related timing data. |

---

## 3. Client prediction flow

```text
1. Receive a server snapshot
   └─ Apply the latest authoritative state to predicted entities

2. Find the oldest applied tick
   └─ Use the oldest tick across affected predicted entities

3. Roll back and re-simulate
   └─ PredictedSimulationSystemGroup repeats from oldest tick to target tick
   └─ TimeData and NetworkTime update for each tick

4. Render
   └─ Present the predicted result
```

High latency multiplies the cost. A 300 ms connection can require roughly 20+ frames of re-simulation depending on tick rate and prediction settings.

---

## 4. Writing prediction systems

### 4.1 Use the `Simulate` tag

At the start of the prediction loop, Netcode disables `Simulate` on predicted Ghosts and then enables it only for entities that should run on the current tick.

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Mathematics;
using Unity.NetCode;
using Unity.Transforms;

[UpdateInGroup(typeof(PredictedSimulationSystemGroup))]
public partial struct MovementSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var speed = 5f;
        var dt = SystemAPI.Time.DeltaTime;

        foreach (var (transform, input) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<PlayerInput>>()
                .WithAll<Simulate>())
        {
            var move = new float3(input.ValueRO.Horizontal, 0, input.ValueRO.Vertical);
            transform.ValueRW.Position += move * speed * dt;
        }
    }
}
```

Always include `.WithAll<Simulate>()` in prediction queries. Without it, the system can re-simulate entities for ticks that are already done.

### 4.2 Server-side behavior

On the server, the prediction loop runs exactly once per server tick:

- `TimeData` contains normal server tick data.
- `Simulate` is always enabled.
- The same system code can run on both client and server.

---

## 5. Remote-player prediction

If another player's command data is serialized, that remote player can also be predicted.

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Mathematics;
using Unity.NetCode;
using Unity.Transforms;

[UpdateInGroup(typeof(PredictedSimulationSystemGroup))]
public partial struct RemotePlayerMovement : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, input) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<PlayerInput>>()
                .WithAll<PredictedGhost, Simulate>())
        {
            transform.ValueRW.Position +=
                new float3(input.ValueRO.Horizontal, 0, input.ValueRO.Vertical)
                * SystemAPI.Time.DeltaTime;
        }
    }
}
```

With `IInputComponentData`, the input system provides the input for the current tick automatically.

---

## 6. Prediction smoothing

Prediction smoothing reduces visible correction when the server result differs from the predicted result.

```csharp
using Unity.NetCode;
using Unity.Transforms;

GhostPredictionSmoothing.RegisterSmoothingAction<LocalTransform>(
    state.EntityManager,
    CustomSmoothing.Action);
```

```csharp
using Unity.Burst;
using Unity.Collections.LowLevel.Unsafe;
using Unity.Mathematics;
using Unity.NetCode;
using Unity.Transforms;

[BurstCompile]
public static unsafe class CustomSmoothing
{
    public static readonly PortableFunctionPointer<
        GhostPredictionSmoothing.SmoothingActionDelegate> Action =
        new(SmoothingAction);

    [BurstCompile(DisableDirectCall = true)]
    private static void SmoothingAction(
        void* currentData, void* previousData, void* userData)
    {
        ref var current = ref UnsafeUtility.AsRef<LocalTransform>(currentData);
        ref var backup = ref UnsafeUtility.AsRef<LocalTransform>(previousData);

        var dist = math.distance(current.Position, backup.Position);
        if (dist > 0)
        {
            current.Position = math.lerp(backup.Position, current.Position, 0.5f);
        }
    }
}
```

---

## 7. Prediction switching

Prediction is expensive, so a Ghost can switch dynamically between `Predicted` and `Interpolated`.

### 7.1 Switch prerequisites

- Supported Ghost Modes must be `All`.
- Current mode must not be `OwnerPredicted`.
- The Ghost must not already be switching.

### 7.2 Queueing switches

```csharp
using Unity.NetCode;

ref var switchQueues = ref SystemAPI
    .GetSingletonRW<GhostPredictionSwitchingQueues>()
    .ValueRW;

switchQueues.ConvertToPredictedQueue.Enqueue(new ConvertPredictionEntry
{
    TargetEntity = entity,
    TransitionDurationSeconds = 1f,
});

switchQueues.ConvertToInterpolatedQueue.Enqueue(new ConvertPredictionEntry
{
    TargetEntity = entity,
    TransitionDurationSeconds = 0f,
});
```

Interpolated and Predicted Ghosts run on different timelines, often separated by roughly two ping intervals. `SwitchPredictionSmoothing` interpolates position and rotation over the configured duration to avoid visible teleports.

---

## 8. Prediction performance controls

| Technique | Effect |
|-----------|--------|
| **Prediction Switching** | Switch distant or low-importance Ghosts to Interpolated. |
| **Restrict GhostField write access** | Use read-only access for fields that do not change in the prediction loop. |
| **Physics batching** | Use `MaxPredictionStepBatchSizeRepeatedTick` for repeated physics re-simulation. |
| **Tick-rate tuning** | Lower `SimulationTickRate` reduces re-simulation count, trading off responsiveness. |

---

## 9. Related docs

- [`20_Netcode Command Stream & Input.md`](20_Netcode Command Stream & Input.md) — input data used by prediction systems.
- [`22_Netcode Ghost Optimization · Importance · Relevancy.md`](22_Netcode Ghost Optimization · Importance · Relevancy.md) — prediction switching as an optimization strategy.
- [`23_Netcode Physics Integration & Lag Compensation.md`](23_Netcode Physics Integration & Lag Compensation.md) — predicted physics behavior.

---

## 10. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Prediction does not run | Check that the Ghost mode is Predicted and that prediction queries include `Simulate`. |
| Heavy jitter | Register prediction smoothing and confirm quantization matches gameplay needs. |
| Entity teleports on server correction | Set `MaxSmoothingDistance` or add custom smoothing. |
| Prediction CPU is too high | Use prediction switching, reduce tick rate, and batch physics re-simulation where appropriate. |
| Entity simulates more than once per tick | A prediction query is missing `.WithAll<Simulate>()`. |
