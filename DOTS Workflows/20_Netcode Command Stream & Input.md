---
title: Netcode Command Stream & Input
updated: 2026-04-27
folder: DOTS Workflows
---

# Netcode Command Stream & Input
### Unity 6000.5 · Entities 6.5.0 · Netcode for Entities 6.5.0

---

## 1. Overview

Once a client is in-game, it sends a continuous **command stream** to the server. The stream includes input data and snapshot ACK data; it continues even when the player is not actively pressing input so the connection remains synchronized.

---

## 2. `ICommandData` vs `IInputComponentData`

| | `ICommandData` | `IInputComponentData` |
|---|---|---|
| Use | Low-level command control. | High-level input workflow. |
| Buffer management | Manual, via `AddCommandData`. | Automatic through generated code. |
| Tick mapping | Manual, via `GetDataAtTick`. | Automatic; current tick input is provided. |
| `InputEvent` | Not supported. | Supported for one-shot events. |
| Maximum payload | 1024 bytes. | 1024 bytes. |
| Best fit | Custom command pipelines. | Normal player input. |

---

## 3. `IInputComponentData`

### 3.1 Define input

```csharp
using Unity.NetCode;

public struct PlayerInput : IInputComponentData
{
    public int Horizontal;
    public int Vertical;
    public InputEvent Jump;
}
```

`InputEvent` preserves one-shot events such as `GetKeyDown` so they register exactly once on the server even if packet delivery drops a redundant command packet.

### 3.2 Gather input on the client

```csharp
using Unity.Entities;
using Unity.NetCode;
using UnityEngine;

[UpdateInGroup(typeof(GhostInputSystemGroup))]
public partial struct GatherInputSystem : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<PlayerInput>();
    }

    public void OnUpdate(ref SystemState state)
    {
        bool jump = Input.GetKeyDown(KeyCode.Space);
        bool left = Input.GetKey(KeyCode.A);
        bool right = Input.GetKey(KeyCode.D);

        foreach (var input in
            SystemAPI.Query<RefRW<PlayerInput>>()
                .WithAll<GhostOwnerIsLocal>())
        {
            input.ValueRW = default;
            if (jump) input.ValueRW.Jump.Set();
            if (left) input.ValueRW.Horizontal -= 1;
            if (right) input.ValueRW.Horizontal += 1;
        }
    }
}
```

### 3.3 Process input on server and prediction loop

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Mathematics;
using Unity.NetCode;
using Unity.Transforms;

[UpdateInGroup(typeof(PredictedSimulationSystemGroup))]
public partial struct ProcessInputSystem : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<PlayerInput>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var speed = 3f;
        var dt = SystemAPI.Time.DeltaTime;

        foreach (var (input, transform) in
            SystemAPI.Query<RefRO<PlayerInput>, RefRW<LocalTransform>>()
                .WithAll<Simulate>())
        {
            if (input.ValueRO.Jump.IsSet)
            {
                // Jump logic; runs once on the input tick.
            }

            transform.ValueRW.Position +=
                new float3(input.ValueRO.Horizontal, 0, 0) * speed * dt;
        }
    }
}
```

The prediction loop can execute this system many times while re-simulating. `InputEvent.IsSet` is true only for the correct tick.

---

## 4. `ICommandData`

### 4.1 Define a command

```csharp
using Unity.Mathematics;
using Unity.NetCode;

public struct MyCommand : ICommandData
{
    public NetworkTick Tick { get; set; }
    public int Action;
    public float2 Direction;
}
```

### 4.2 Add and read command data

```csharp
using Unity.Entities;
using Unity.NetCode;

DynamicBuffer<MyCommand> buffer = default;
NetworkTick networkTick = default;

buffer.AddCommandData(new MyCommand
{
    Tick = networkTick,
    Action = 1,
    Direction = new float2(1, 0),
});

if (buffer.GetDataAtTick(networkTick, out var cmd))
{
    // Use cmd.Action and cmd.Direction.
}
```

---

## 5. Command-send mechanism

```text
Command packet
├─ Tick, Command
├─ Delta(Tick - 1)
├─ Delta(Tick - 2)
└─ Delta(Tick - 3)

The last three inputs are redundantly sent with delta compression.
Maximum payload: 1024 bytes.
```

`CommandSendPacketSystem` flushes at `SimulationTickRate`, and `CommandReceiveSystem` receives the command data on the server and forwards it to the target entity.

---

## 6. `AutoCommandTarget`

If a Ghost has `AutoCommandTarget`, commands are sent automatically.

| Requirement | Meaning |
|-------------|---------|
| Ghost has owner enabled | Enabled on `GhostAuthoringComponent`. |
| Support Auto Command Target enabled | Enabled on `GhostAuthoringComponent`. |
| Client is the owner | Server sets `GhostOwner.NetworkId` to the client's `NetworkId.Value`. |
| Ghost is Predicted or OwnerPredicted | Command data targets prediction. |
| `AutoCommandTarget.Enabled` is true | Can be disabled at runtime. |

Without `AutoCommandTarget`, set `CommandTarget` on the connection entity manually.

```csharp
using Unity.Entities;
using Unity.NetCode;

Entity connectionEntity = default;
Entity playerEntity = default;

state.EntityManager.SetComponentData(connectionEntity,
    new CommandTarget { targetEntity = playerEntity });
```

---

## 7. Identifying locally owned entities

### 7.1 `GhostOwnerIsLocal`

```csharp
using Unity.Entities;
using Unity.NetCode;
using Unity.Transforms;

foreach (var (input, transform) in
    SystemAPI.Query<RefRW<PlayerInput>, RefRW<LocalTransform>>()
        .WithAll<GhostOwnerIsLocal>())
{
    // Local-owned entity only.
}
```

### 7.2 Manual owner check

```csharp
using Unity.Entities;
using Unity.NetCode;
using Unity.Transforms;

var localId = SystemAPI.GetSingleton<NetworkId>().Value;

foreach (var (owner, transform) in
    SystemAPI.Query<RefRO<GhostOwner>, RefRW<LocalTransform>>())
{
    if (owner.ValueRO.NetworkId == localId)
    {
        // Local-owned entity.
    }
}
```

---

## 8. Forced input latency

`ClientTickRate.ForcedInputLatencyTicks` adds intentional input delay.

| Setting | Meaning |
|---------|---------|
| `ForcedInputLatencyTicks` | Additional input latency in ticks; default is 0. |
| Use case | Give all players the same input delay in genres such as fighting games. |
| Effect | Included automatically in `MaxPredictAheadTimeMS` calculations. |

### 8.1 `InputTargetTick` vs `ServerTick`

In input systems, use `NetworkTime.InputTargetTick` rather than `NetworkTime.ServerTick` when forced input latency is active.

```csharp
using Unity.NetCode;

var inputTick = SystemAPI.GetSingleton<NetworkTime>().InputTargetTick;
var serverTick = SystemAPI.GetSingleton<NetworkTime>().ServerTick;
```

`InputTargetTick` and `ServerTick` differ when `ForcedInputLatencyTicks` is non-zero. Input should target `InputTargetTick`.

---

## 9. Related docs

- [`19_Netcode Prediction & Rollback.md`](19_Netcode Prediction & Rollback.md) — prediction systems that consume input.
- [`17_Netcode Network Connection & Approval.md`](17_Netcode Network Connection & Approval.md) — `NetworkStreamInGame` and connection entities.
- [`21_Netcode RPC.md`](21_Netcode RPC.md) — reliable one-shot messages, not continuous input.

---

## 10. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Server receives no input | Add `NetworkStreamInGame`, set `CommandTarget`, or satisfy `AutoCommandTarget` requirements. |
| Jump fires more than once | Use `InputEvent.Set()` / `.IsSet` instead of a normal bool. |
| Payload-size errors | Keep command structs at or below 1024 bytes. |
| Input missing during prediction | Prefer `IInputComponentData` automatic tick mapping, or check `GetDataAtTick`. |
| Server code runs in `GhostInputSystemGroup` | That group is client-side; put server logic in `PredictedSimulationSystemGroup`. |
