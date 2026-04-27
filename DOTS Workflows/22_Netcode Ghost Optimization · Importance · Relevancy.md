---
title: Netcode Ghost Optimization · Importance · Relevancy
updated: 2026-04-27
folder: DOTS Workflows
---

# Netcode Ghost Optimization · Importance · Relevancy
### Unity 6000.5 · Entities 6.5.0 · Netcode for Entities 6.5.0

---

## 1. Overview

Large multiplayer worlds cannot send every Ghost to every client every tick. Netcode for Entities uses **Optimization Mode**, **Importance Scaling**, **Relevancy**, and per-Ghost send-rate controls to reduce CPU cost and bandwidth.

The core rule is simple: the server should spend snapshot budget on Ghosts that matter to a specific connection right now.

---

## 2. Optimization Mode

Set Optimization Mode on `GhostAuthoringComponent`.

| Mode | Meaning | Good fit |
|------|---------|----------|
| **Dynamic** | Default. Optimizes snapshot size both when data changes and when it does not. | Normal moving Ghosts. |
| **Static** | Avoids re-sending data when the Ghost has not changed. | Buildings, terrain, static props. |

The 1.11.0 line added a Static Ghost optimization that can substantially reduce `GhostUpdateSystem` timing for unchanged static Ghosts.

---

## 3. Snapshot-size limits

Snapshot budget can be limited at several levels.

| Setting | Meaning |
|---------|---------|
| `NetworkStreamSnapshotTargetSize` | Per-connection soft byte target. |
| `GhostSendSystemData.MaxSendEntities` | Maximum entities per snapshot. |
| `GhostSendSystemData.MaxSendChunks` | Maximum chunks per snapshot. |

When budget is limited, higher-importance Ghosts are sent first.

---

## 4. Importance Scaling

Importance Scaling computes per-Ghost send priority on the server.

### 4.1 Inputs

`GhostImportance` combines three kinds of data.

| Data | Meaning |
|------|---------|
| Per-connection | Connection-specific data such as player position. |
| Singleton data | Global settings such as tile size. |
| Per-chunk shared data | Chunk-level scaling parameters. |

### 4.2 Distance-based importance

```csharp
using Unity.Entities;
using Unity.Mathematics;
using Unity.NetCode;

var gridSingleton = state.EntityManager.CreateSingleton(
    new GhostDistanceData
    {
        TileSize = new int3(50, 50, 256),
        TileCenter = new int3(0, 0, 128),
        TileBorderWidth = new float3(1f, 1f, 1f),
    });

state.EntityManager.AddComponentData(gridSingleton, new GhostImportance
{
    ScaleImportanceFunction = GhostDistanceImportance.ScaleFunctionPointer,
    GhostConnectionComponentType = ComponentType.ReadOnly<GhostConnectionPosition>(),
    GhostImportanceDataType = ComponentType.ReadOnly<GhostDistanceData>(),
    GhostImportancePerChunkDataType = ComponentType.ReadOnly<GhostDistancePartitionShared>(),
});
```

Breaking change from the 1.8.0 line: `GhostDistanceImportance` scale functions no longer multiply `baseImportance` by 1000. Existing custom importance code that assumed the old multiplier needs review.

### 4.3 Automatic `GhostDistancePartitionShared`

```csharp
using Unity.NetCode;

GhostDistancePartitioningSystem.AutomaticallyAddGhostDistancePartitionSharedComponent = true;
```

### 4.4 Importance Visualizer

The PlayMode Tool includes an Importance Visualizer in the 1.8.0+ line. Use it to inspect the runtime importance distribution instead of guessing from code.

---

## 5. Ghost Relevancy

Ghost Relevancy lets the server hide or expose specific Ghosts per connection.

| Mode | Meaning |
|------|---------|
| `Disabled` | Default. No filtering; all Ghosts are candidates. |
| `SetIsRelevant` | Only listed Ghosts replicate to that connection. |
| `SetIsIrrelevant` | Listed Ghosts are excluded from that connection. |

### 5.1 Use cases

| Scenario | Implementation idea |
|----------|---------------------|
| Distance culling | Mark distant Ghosts irrelevant. |
| Fog of war | Hide entities outside vision. |
| Client-specific Ghosts | Mark a Ghost relevant only for one client. |
| Team filtering | Hide enemy-only internal data. |

### 5.2 Relevancy example

```csharp
using Unity.Entities;
using Unity.Mathematics;
using Unity.NetCode;
using Unity.Transforms;

[WorldSystemFilter(WorldSystemFilterFlags.ServerSimulation)]
[UpdateInGroup(typeof(GhostSimulationSystemGroup))]
[UpdateBefore(typeof(GhostSendSystem))]
public partial struct RelevancySystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var relevancy = SystemAPI.GetSingletonRW<GhostRelevancy>();

        relevancy.ValueRW.GhostRelevancyMode = GhostRelevancyMode.SetIsRelevant;
        relevancy.ValueRW.GhostRelevancySet.Clear();

        foreach (var (ghostInstance, transform) in
            SystemAPI.Query<RefRO<GhostInstance>, RefRO<LocalTransform>>())
        {
            foreach (var (connId, connPos) in
                SystemAPI.Query<RefRO<NetworkId>, RefRO<GhostConnectionPosition>>())
            {
                float dist = math.distance(
                    transform.ValueRO.Position,
                    connPos.ValueRO.Position);

                if (dist < 100f)
                {
                    relevancy.ValueRW.GhostRelevancySet.TryAdd(
                        new RelevantGhostForConnection(
                            connId.ValueRO.Value,
                            ghostInstance.ValueRO.ghostId),
                        1);
                }
            }
        }
    }
}
```

This simplified example is O(Ghosts × connections). For large worlds, prefer distance-based importance partitioning and reserve relevancy for rules that partitioning cannot express, such as fog of war or team visibility.

### 5.3 `DefaultRelevancyQuery`

`DefaultRelevancyQuery` defines Ghost chunks that are always relevant to every connection, such as global game-state Ghosts or UI-related network state.

### 5.4 Relevancy plus importance

The 1.6.1+ line supports combining Ghost Relevancy and Importance Scaling through `PrioChunks.isRelevant`, enabling a faster relevancy path.

---

## 6. Prediction CPU optimization

### 6.1 Prediction Switching

Switch distant or unimportant Ghosts from Predicted to Interpolated.

```csharp
using Unity.Entities;
using Unity.Mathematics;
using Unity.NetCode;

float distance = math.distance(playerPos, ghostPos);
if (distance > predictionRange)
{
    switchQueues.ConvertToInterpolatedQueue.Enqueue(new ConvertPredictionEntry
    {
        TargetEntity = ghostEntity,
        TransitionDurationSeconds = 0.5f,
    });
}
```

### 6.2 Minimize GhostField write access

Jobs with write access to GhostField components can trigger serialization checks. Use read-only access for fields that do not change in the prediction loop.

### 6.3 Physics re-simulation

For high-latency cases that require 20+ predicted re-simulation frames:

- Disable multithreaded physics through the `PhysicsStep` singleton when main-thread consolidation is cheaper.
- Use `MaxPredictionStepBatchSizeRepeatedTick` to batch repeated physics ticks.

---

## 7. `MaxIterateChunks`

`GhostSendSystemData.MaxIterateChunks` limits the number of chunks processed in a single tick.

| Setting | Meaning |
|---------|---------|
| `MaxIterateChunks` | Limits serialization CPU cost per tick. |
| `MaxSendChunks` | Limits bandwidth by capping chunks included in a snapshot. |

These settings solve different problems. `MaxIterateChunks` spreads CPU work; `MaxSendChunks` constrains snapshot size.

---

## 8. `MaxSendRate`

`GhostAuthoringComponent.MaxSendRate` limits a Ghost's send frequency in Hz.

```text
NetworkTickRate = 60, MaxSendRate = 10
→ This Ghost can be included once every 6 network ticks.
```

Use this for low-frequency or low-importance Ghosts.

---

## 9. `UseSingleBaseline`

`GhostAuthoringComponent.UseSingleBaseline = true` uses one baseline for delta compression. This can reduce CPU cost, with a possible bandwidth increase.

---

## 10. Optimization checklist

- [ ] Use **Static** Optimization Mode for Ghosts that rarely change.
- [ ] Set `NetworkStreamSnapshotTargetSize` to cap bandwidth per connection.
- [ ] Implement distance-based Importance Scaling.
- [ ] Apply Relevancy only where partitioning cannot express the rule.
- [ ] Switch distant Ghosts to Interpolated mode.
- [ ] Use `MaxSendRate` for low-frequency Ghosts.
- [ ] Avoid write access to unchanged GhostField data.
- [ ] Inspect runtime priority with the Importance Visualizer.

---

## 11. Related docs

- [`18_Netcode Ghost Snapshot & Synchronization.md`](18_Netcode Ghost Snapshot & Synchronization.md) — Ghost serialization and snapshot flow.
- [`19_Netcode Prediction & Rollback.md`](19_Netcode Prediction & Rollback.md) — prediction switching behavior.
- [`24_Netcode Profiler & Debugging.md`](24_Netcode Profiler & Debugging.md) — profiling bandwidth and prediction cost.

---

## 12. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| A Ghost takes too long to appear | Importance is too low or snapshot budget is too tight. |
| Ghosts stop sending after Importance Scaling | Confirm `GhostDistancePartitionShared` is present or use the automatic-add option. |
| Ghost flickers with Relevancy enabled | Check boundary conditions and rebuild the relevancy set consistently each frame. |
| Bandwidth is available but Ghosts are still missing | Check `MaxSendEntities` and `MaxSendChunks` limits. |
