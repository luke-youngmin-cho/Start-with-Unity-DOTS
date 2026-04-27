---
title: Netcode RPC
updated: 2026-04-27
folder: DOTS Workflows
---

# Netcode RPC
### Unity 6000.5 · Entities 6.5.0 · Netcode for Entities 6.5.0

---

## 1. Overview

An RPC is a **reliable one-shot message** between client and server. RPC serialization and deserialization code is generated automatically, and RPC handling still fits into the ECS / Job System frame.

RPCs are not a replacement for Ghost replication or command streams. Use them for discrete game-flow events, not high-frequency state.

---

## 2. RPC vs Ghost vs Command

| | RPC | Ghost | Command |
|---|---|---|---|
| Transport behavior | Reliable, ordered. | Unreliable with replication optimization. | Unreliable with redundant sends. |
| Direction | Bidirectional. | Server → Client. | Client → Server. |
| Frequency | One-shot. | Every network tick as needed. | Every simulation tick. |
| Use | Game flow, lobby, chat. | Replicated entity state. | Player input. |
| In-flight limit | Yes, reliable pipeline limit. | No. | No. |

Because RPCs are ordered and reliable, frequent RPC traffic can stall behind in-flight packets.

---

## 3. Define RPCs

### 3.1 Empty RPC

```csharp
using Unity.NetCode;

public struct PingRpc : IRpcCommand
{
}
```

### 3.2 RPC with data

```csharp
using Unity.Collections;
using Unity.NetCode;

public struct ChatMessageRpc : IRpcCommand
{
    public FixedString128Bytes Message;
    public int SenderId;
}
```

---

## 4. Send RPCs

### 4.1 Client to server

```csharp
using Unity.Collections;
using Unity.Entities;
using Unity.NetCode;
using UnityEngine;

[WorldSystemFilter(WorldSystemFilterFlags.ClientSimulation)]
public partial struct SendChatSystem : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<NetworkId>();
    }

    public void OnUpdate(ref SystemState state)
    {
        if (!Input.GetKeyDown(KeyCode.Space))
            return;

        var ecb = new EntityCommandBuffer(Allocator.Temp);
        var entity = ecb.CreateEntity();

        ecb.AddComponent(entity, new ChatMessageRpc
        {
            Message = "Hello!",
            SenderId = SystemAPI.GetSingleton<NetworkId>().Value,
        });

        ecb.AddComponent<SendRpcCommandRequest>(entity);
        ecb.Playback(state.EntityManager);
    }
}
```

`ISystem` is a struct. Do not store frame-to-frame state in system instance fields; store state in components or singletons instead.

### 4.2 Server to one client

```csharp
using Unity.Entities;
using Unity.NetCode;

Entity targetConnectionEntity = default;

var entity = ecb.CreateEntity();
ecb.AddComponent(entity, new NotifyRpc());
ecb.AddComponent(entity, new SendRpcCommandRequest
{
    TargetConnection = targetConnectionEntity,
});
```

### 4.3 Server broadcast

```csharp
using Unity.Entities;
using Unity.NetCode;

var entity = ecb.CreateEntity();
ecb.AddComponent(entity, new GameStartRpc());
ecb.AddComponent(entity, new SendRpcCommandRequest
{
    TargetConnection = Entity.Null,
});
```

`Entity.Null` broadcasts the RPC.

---

## 5. Receive RPCs

```csharp
using Unity.Collections;
using Unity.Entities;
using Unity.NetCode;
using UnityEngine;

[WorldSystemFilter(WorldSystemFilterFlags.ServerSimulation)]
public partial struct ReceiveChatSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        foreach (var (req, chat, entity) in
            SystemAPI.Query<RefRO<ReceiveRpcCommandRequest>, RefRO<ChatMessageRpc>>()
                .WithEntityAccess())
        {
            var sourceConnection = req.ValueRO.SourceConnection;
            Debug.Log($"Player {chat.ValueRO.SenderId}: {chat.ValueRO.Message}");

            ecb.DestroyEntity(entity);
        }

        ecb.Playback(state.EntityManager);
    }
}
```

Always destroy the received RPC entity after processing it. Otherwise the same RPC is processed again every frame.

---

## 6. `RpcQueue`

Use `RpcQueue` only when manual RPC scheduling is required.

```csharp
using Unity.Entities;
using Unity.NetCode;

var rpcQueue = SystemAPI.GetSingleton<RpcCollection>()
    .GetRpcQueue<MyRpc, MyRpc>();
var bufferLookup = SystemAPI.GetBufferLookup<OutgoingRpcDataStreamBuffer>();

if (bufferLookup.HasBuffer(connectionEntity))
{
    var buffer = bufferLookup[connectionEntity];
    rpcQueue.Schedule(buffer, new MyRpc());
}
```

---

## 7. Manual serialization

Automatic generation is the default. If you disable it, serialization and deserialization must be symmetric.

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.NetCode;
using Unity.Networking.Transport;

[BurstCompile]
public struct MyCustomRpc : IComponentData, IRpcCommandSerializer<MyCustomRpc>
{
    public int Data;

    public void Serialize(ref DataStreamWriter writer, in RpcSerializerState state,
        in MyCustomRpc data)
    {
        writer.WriteInt(data.Data);
    }

    public void Deserialize(ref DataStreamReader reader, in RpcDeserializerState state,
        ref MyCustomRpc data)
    {
        data.Data = reader.ReadInt();
    }

    // Execute implementation omitted.
}
```

---

## 8. `IApprovalRpcCommand`

`IApprovalRpcCommand` is a special RPC used only during connection approval. Normal `IRpcCommand` messages are blocked during Handshake / Approval state.

```csharp
using Unity.Collections;
using Unity.NetCode;

public struct LoginApproval : IApprovalRpcCommand
{
    public FixedString512Bytes AuthToken;
}
```

See [`17_Netcode Network Connection & Approval.md`](17_Netcode Network Connection & Approval.md) for the full approval flow.

---

## 9. Common patterns

| Pattern | Example RPCs |
|---------|--------------|
| Lobby / matchmaking | `JoinLobbyRpc`, `ReadyRpc`. |
| Game flow | `GameStartRpc`, `RoundEndRpc`. |
| Chat | `ChatMessageRpc`. |
| Item trading | `TradeRequestRpc`, `TradeAcceptRpc`. |
| Connection approval | `IApprovalRpcCommand` implementations. |

---

## 10. Related docs

- [`18_Netcode Ghost Snapshot & Synchronization.md`](18_Netcode Ghost Snapshot & Synchronization.md) — Ghost replication for state.
- [`20_Netcode Command Stream & Input.md`](20_Netcode Command Stream & Input.md) — command streams for input.
- [`17_Netcode Network Connection & Approval.md`](17_Netcode Network Connection & Approval.md) — approval RPCs.

---

## 11. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| RPC is not received | Confirm `SendRpcCommandRequest` was added and the system runs in the intended world. |
| RPC is processed every frame | Destroy the received RPC entity after handling it. |
| RPC order appears delayed | Reliable ordered transport can stall behind in-flight packets. Use Ghosts or commands for frequent data. |
| RPC is blocked during approval | Use `IApprovalRpcCommand`, not `IRpcCommand`. |
| Manual serialization fails | Ensure `Serialize` and `Deserialize` read/write fields in the same order. |
