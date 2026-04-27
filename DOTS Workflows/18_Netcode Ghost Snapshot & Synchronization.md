---
title: Netcode Ghost Snapshot & Synchronization
updated: 2026-04-27
folder: DOTS Workflows
---

# Netcode Ghost Snapshot & Synchronization
### Unity 6000.5 · Entities 6.5.0 · Netcode for Entities 6.5.0

---

## 1. Overview

A **Ghost** is a networked entity simulated on the server and replicated to clients. The server periodically sends Ghost state as snapshots, and each client applies those snapshots through interpolation or prediction.

| Mechanism | Use |
|-----------|-----|
| **Ghost** | Eventual consistency over unreliable packets with built-in replication optimization. |
| **RPC** | One-shot reliable event. |

---

## 2. Ghost vs RPC

| Use Ghost for | Use RPC for |
|---------------|-------------|
| Spatial entity state near a client. | High-level game-flow events. |
| Data that needs client prediction. | One-shot commands such as chat or setting changes. |
| State that should still exist after a disconnect/reconnect cycle. | Events with no persistent replicated state. |

---

## 3. Ghost authoring

Add `GhostAuthoringComponent` to the prefab that should be replicated.

| Property | Meaning |
|----------|---------|
| **Name** | Ghost identifier. |
| **Importance** | Snapshot priority when bandwidth is constrained. |
| **Supported Ghost Mode** | `All`, `Interpolated`, or `Predicted`. |
| **Default Ghost Mode** | `Interpolated`, `Predicted`, or `OwnerPredicted`. |
| **Optimization Mode** | `Dynamic` by default, or `Static` for objects that rarely change. |
| **MaxSendRate** | Maximum replication rate in Hz, capped by `NetworkTickRate`. |
| **UseSingleBaseline** | Single baseline delta compression option from the 1.5.0 line. |

### 3.1 Ghost modes

| Mode | Meaning |
|------|---------|
| **Interpolated** | Client interpolates between server snapshots. Visuals are smooth but delayed. |
| **Predicted** | Client simulates ahead to reduce input latency. CPU cost is higher. |
| **OwnerPredicted** | Owning client predicts; other clients interpolate. |

---

## 4. Serialization attributes

### 4.1 `GhostField`

Use `GhostField` on component fields that should be serialized.

```csharp
using Unity.Entities;
using Unity.Mathematics;
using Unity.NetCode;

public struct MyComponent : IComponentData
{
    [GhostField(Quantization = 1000, Smoothing = SmoothingAction.Interpolate)]
    public float3 Position;

    [GhostField(Quantization = 100)]
    public float Health;

    [GhostField]
    public int Score;

    [GhostField(SendData = false)]
    public float LocalOnly;
}
```

| `GhostField` property | Meaning |
|-----------------------|---------|
| `Quantization` | Float-to-int multiplier; `1000` keeps three decimal places. |
| `Smoothing` | `Clamp`, `Interpolate`, or `InterpolateAndExtrapolate`. |
| `MaxSmoothingDistance` | Distance threshold that disables interpolation for teleports. |
| `Composite` | Treat the struct as one change bit. |
| `SendData` | Set false to exclude the field from serialization. |
| `SubType` | Selects a specialized serialization rule. |

### 4.2 `GhostEnabledBit`

Use `GhostEnabledBit` to replicate the enabled state of an enableable component.

```csharp
using Unity.Entities;
using Unity.NetCode;

[GhostEnabledBit]
public struct MyEnableableComp : IComponentData, IEnableableComponent
{
    [GhostField] public int Value;
}
```

### 4.3 `GhostComponent`

Use `GhostComponent` to control prefab variants, child-entity serialization, and replication scope.

```csharp
using Unity.Entities;
using Unity.NetCode;

[GhostComponent(
    PrefabType = GhostPrefabType.All,
    SendTypeOptimization = GhostSendType.AllClients,
    SendToOwner = SendToOwnerType.SendToNonOwner)]
public struct EnemyData : IComponentData
{
    [GhostField] public float Health;
}
```

| Property | Options |
|----------|---------|
| `PrefabType` | `InterpolatedClient`, `PredictedClient`, `Client`, `Server`, `AllPredicted`, `All`. |
| `SendTypeOptimization` | `None`, `Interpolated`, `Predicted`, `All`. |
| `SendToOwner` | `SendToOwner`, `SendToNonOwner`, `All`. |
| `SendDataForChildEntity` | Replicate child-entity component data; slower than root entity data. |

---

## 5. Buffer serialization

For `IBufferElementData`, every public field must have `[GhostField]`.

```csharp
using Unity.Entities;
using Unity.NetCode;

[InternalBufferCapacity(8)]
public struct MyBuffer : IBufferElementData
{
    [GhostField] public int Value;
    [GhostField] public float Score;
}
```

Buffers are stricter than components: a buffer element needs every public field annotated, not just one serialized field.

---

## 6. Ghost component variants

Use a variant to define an alternate serialization schema without modifying the original component type.

```csharp
using Unity.Mathematics;
using Unity.NetCode;
using Unity.Transforms;

[GhostComponentVariation(typeof(LocalTransform), "Transform - 2D")]
[GhostComponent(PrefabType = GhostPrefabType.All)]
public struct Transform2dVariant
{
    [GhostField(Quantization = 1000, Smoothing = SmoothingAction.InterpolateAndExtrapolate)]
    public float3 Position;
}
```

| Variant | Use |
|---------|-----|
| `ClientOnlyVariant` | Component exists only on clients. |
| `ServerOnlyVariant` | Component exists only on the server. |
| `DontSerializeVariant` | Disable serialization entirely. |

---

## 7. Snapshot system

### 7.1 Partial snapshots

If a snapshot would exceed the MTU, Netcode sends higher-importance Ghosts first. Remaining Ghosts are sent in later ticks, so world state streams progressively.

### 7.2 `NetworkTickRate`

When `NetworkTickRate` is lower than `SimulationTickRate`, `GhostSendSystem` work is distributed over multiple simulation ticks.

```text
SimulationTickRate = 60, NetworkTickRate = 30
Tick 1: GhostSendSystem sends a snapshot
Tick 2: GhostSendSystem skips
Tick 3: GhostSendSystem sends a snapshot
...
```

---

## 8. Ghost prefab registration workflow

```text
1. Create a Ghost prefab
   ├─ Add GhostAuthoringComponent
   └─ Add the required authoring components

2. Place or reference it from a SubScene
   ├─ Put the prefab inside the SubScene, or
   └─ Let GhostCollection register the prefab

3. Bake
   ├─ Baker analyzes GhostField attributes
   └─ Source Generator emits serialization code

4. Runtime
   ├─ Server instantiates the Ghost entity
   ├─ GhostSendSystem sends snapshots
   └─ Client automatically spawns the Ghost
```

### 8.1 Prespawned Ghosts

Ghosts placed in a SubScene become **Prespawned Ghosts**:

- Server and client automatically match the same entity when the SubScene loads.
- No separate spawn message is needed; snapshot sync is enough.
- Prespawn Ghost IDs are preserved during Host Migration in the 1.6.1+ line.

Static world objects such as buildings and terrain are usually best as Prespawned Ghosts with Static Optimization Mode.

---

## 9. Checklist

- [ ] Add `GhostAuthoringComponent` to the Ghost prefab.
- [ ] Mark replicated fields with `[GhostField]`.
- [ ] Choose Ghost Mode: `Predicted`, `Interpolated`, or `OwnerPredicted`.
- [ ] Add `NetworkStreamInGame` so snapshot exchange can begin.
- [ ] Tune `Quantization` for bandwidth and precision.
- [ ] Enable `SendDataForChildEntity` only when child-entity replication is required.

---

## 10. Related docs

- [`22_Netcode Ghost Optimization · Importance · Relevancy.md`](22_Netcode Ghost Optimization · Importance · Relevancy.md) — bandwidth, relevancy, and send-rate controls.
- [`19_Netcode Prediction & Rollback.md`](19_Netcode Prediction & Rollback.md) — predicted Ghost simulation.
- [`../Changelog/Netcode for Entities 1.4 → 6.5 Key Changes.md`](../Changelog/Netcode for Entities 1.4 → 6.5 Key Changes.md) — Ghost serialization changes across versions.

---

## 11. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Ghost does not appear on the client | Add `NetworkStreamInGame`, and confirm the Ghost prefab is registered. |
| Float values jitter | `Quantization` is too low or smoothing is missing. |
| Teleport creates interpolation artifacts | Set `MaxSmoothingDistance` or use a custom smoothing path. |
| Protocol version mismatch | Client and server Ghost schemas differ. |
| Buffer does not serialize | Every public buffer element field must have `[GhostField]`. |
