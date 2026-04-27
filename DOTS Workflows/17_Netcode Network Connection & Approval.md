---
title: Netcode Network Connection & Approval
updated: 2026-04-27
folder: DOTS Workflows
---

# Netcode Network Connection & Approval
### Unity 6000.5 · Entities 6.5.0 · Netcode for Entities 6.5.0

---

## 1. Overview

Netcode for Entities represents every network connection as an **entity**. The connection entity carries `NetworkStreamConnection`; when the connection ends, the entity is destroyed.

The connection is not considered "in game" until both sides add `NetworkStreamInGame`. Without that component, the server does not send snapshots and the client does not send commands.

---

## 2. Core connection components

| Component | Meaning |
|-----------|---------|
| `NetworkStreamConnection` | Stores the transport handle. |
| `NetworkId` | Unique ID assigned by the server after approval. |
| `NetworkStreamInGame` | Required to exchange snapshots and command data. |
| `CommandTarget` | Entity that receives player commands; maintained by game code unless `AutoCommandTarget` is used. |
| `NetworkStreamRequestDisconnect` | Request to disconnect; do not call the driver disconnect directly. |
| `NetworkStreamIsReconnected` | Reconnection tracking component added in the 1.6.1 line. |

---

## 3. Three connection methods

### 3.1 Manual connection with `NetworkStreamDriver`

```csharp
using Unity.Entities;
using Unity.Networking.Transport;
using Unity.NetCode;

// Server.
var serverDriver = SystemAPI.GetSingletonRW<NetworkStreamDriver>();
serverDriver.ValueRW.Listen(NetworkEndpoint.AnyIpv4.WithPort(7979));

// Client.
var clientDriver = SystemAPI.GetSingletonRW<NetworkStreamDriver>();
clientDriver.ValueRW.Connect(
    state.EntityManager,
    NetworkEndpoint.Parse("127.0.0.1", 7979));
```

### 3.2 `AutoConnectPort` in bootstrap

```csharp
using Unity.NetCode;

public class AutoConnectBootstrap : ClientServerBootstrap
{
    public override bool Initialize(string defaultWorldName)
    {
        AutoConnectPort = 7979;
        CreateDefaultClientServerWorlds();
        return true;
    }
}
```

The server listens on a wildcard address, and the client connects to loopback.

### 3.3 Request components

```csharp
using Unity.Entities;
using Unity.NetCode;

var listenReq = serverWorld.EntityManager.CreateEntity(
    typeof(NetworkStreamRequestListen));
serverWorld.EntityManager.SetComponentData(listenReq,
    new NetworkStreamRequestListen { Endpoint = serverEndpoint });

var connectReq = clientWorld.EntityManager.CreateEntity(
    typeof(NetworkStreamRequestConnect));
clientWorld.EntityManager.SetComponentData(connectReq,
    new NetworkStreamRequestConnect { Endpoint = serverEndpoint });
```

---

## 4. Connection state machine

### 4.1 Client state

```text
Connecting → Handshake → [Approval] → Connected → Disconnected
```

| State | Meaning |
|-------|---------|
| **Connecting** | Raised once after `Connect`. |
| **Handshake** | Transport connection established; protocol handshake pending. |
| **Approval** | Exists only when connection approval is enabled. |
| **Connected** | Server assigned `NetworkId`. |
| **Disconnected** | Disconnect or timeout; `DisconnectReason` is set. |

### 4.2 Server state

```text
Handshake → [Approval] → Connected → Disconnected
```

| State | Meaning |
|-------|---------|
| **Handshake** | Client accepted; `NetworkProtocolVersion` exchange. |
| **Approval** | Exists only when connection approval is enabled. |
| **Connected** | Connection approved and `NetworkId` assigned. |
| **Disconnected** | Client disconnected. |

### 4.3 Listening for connection events

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.NetCode;
using UnityEngine;

[UpdateAfter(typeof(NetworkReceiveSystemGroup))]
[BurstCompile]
public partial struct ConnectionEventListener : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var events = SystemAPI.GetSingleton<NetworkStreamDriver>()
            .ConnectionEventsForTick;

        foreach (var evt in events)
        {
            Debug.Log($"[{state.WorldUnmanaged.Name}] {evt.ToFixedString()}!");
        }
    }
}
```

`ConnectionEventsForTick` is only valid inside `SimulationSystemGroup`; reading it elsewhere can miss or duplicate events.

---

## 5. Connection approval

Connection approval lets the server validate a whitelist, password, token, matchmaking ticket, or similar gate before assigning `NetworkId`.

### 5.1 Approval flow

```text
Client                         Server
  │                              │
  ├─ Connect ─────────────────► │
  │                              ├─ Handshake
  │◄──── Protocol Handshake ────┤
  │                              │
  ├─ IApprovalRpcCommand ─────► │  Approval RPC only
  │                              ├─ Validate
  │                              ├─ Add ConnectionApproved
  │◄──── NetworkId assigned ────┤
          Connected
```

### 5.2 Enable approval

```csharp
using Unity.Entities;
using Unity.NetCode;

using var drvQuery = serverWorld.EntityManager
    .CreateEntityQuery(ComponentType.ReadWrite<NetworkStreamDriver>());

drvQuery.GetSingletonRW<NetworkStreamDriver>().ValueRW
    .RequireConnectionApproval = true;
```

Set the same approval requirement on the client side.

### 5.3 Approval RPC

```csharp
using Unity.Collections;
using Unity.Entities;
using Unity.NetCode;

public struct ApprovalRpc : IApprovalRpcCommand
{
    public FixedString512Bytes Token;
}

[WorldSystemFilter(WorldSystemFilterFlags.ClientSimulation |
    WorldSystemFilterFlags.ThinClientSimulation)]
public partial struct ClientApprovalSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        foreach (var (connection, entity) in
            SystemAPI.Query<RefRW<NetworkStreamConnection>>()
                .WithNone<NetworkId>()
                .WithNone<ApprovalSent>()
                .WithEntityAccess())
        {
            var rpcEntity = ecb.CreateEntity();
            ecb.AddComponent(rpcEntity, new ApprovalRpc { Token = "MY_TOKEN" });
            ecb.AddComponent<SendRpcCommandRequest>(rpcEntity);
            ecb.AddComponent<ApprovalSent>(entity);
        }

        ecb.Playback(state.EntityManager);
    }
}

[WorldSystemFilter(WorldSystemFilterFlags.ServerSimulation)]
public partial struct ServerApprovalSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        foreach (var (req, rpc, entity) in
            SystemAPI.Query<RefRO<ReceiveRpcCommandRequest>, RefRO<ApprovalRpc>>()
                .WithEntityAccess())
        {
            var connEntity = req.ValueRO.SourceConnection;

            if (rpc.ValueRO.Token.Equals("MY_TOKEN"))
                ecb.AddComponent<ConnectionApproved>(connEntity);
            else
                ecb.AddComponent<NetworkStreamRequestDisconnect>(connEntity);

            ecb.DestroyEntity(entity);
        }

        ecb.Playback(state.EntityManager);
    }
}
```

Approval timeout is controlled by `ClientServerTickRate.HandshakeApprovalTimeoutMS`; the default is 5000 ms.

---

## 6. Entering in-game state

After approval, both client and server must add `NetworkStreamInGame` before gameplay data is exchanged.

```csharp
using Unity.Collections;
using Unity.Entities;
using Unity.NetCode;

[WorldSystemFilter(WorldSystemFilterFlags.ServerSimulation)]
public partial struct ServerGoInGameSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        foreach (var (id, entity) in
            SystemAPI.Query<RefRO<NetworkId>>()
                .WithNone<NetworkStreamInGame>()
                .WithEntityAccess())
        {
            ecb.AddComponent<NetworkStreamInGame>(entity);
            // Spawn the player entity and set CommandTarget as needed.
        }

        ecb.Playback(state.EntityManager);
    }
}

[WorldSystemFilter(WorldSystemFilterFlags.ClientSimulation |
    WorldSystemFilterFlags.ThinClientSimulation)]
public partial struct ClientGoInGameSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        foreach (var (id, entity) in
            SystemAPI.Query<RefRO<NetworkId>>()
                .WithNone<NetworkStreamInGame>()
                .WithEntityAccess())
        {
            ecb.AddComponent<NetworkStreamInGame>(entity);
        }

        ecb.Playback(state.EntityManager);
    }
}
```

---

## 7. Data-stream buffers

| Buffer | Direction | Data |
|--------|-----------|------|
| `IncomingRpcDataStreamBuffer` | Receive | RPC data. |
| `IncomingCommandDataStreamBuffer` | Receive | Command data. |
| `IncomingSnapshotDataStreamBuffer` | Receive on client | Snapshot data. |
| `OutgoingRpcDataStreamBuffer` | Send | RPC data. |
| `OutgoingCommandDataStreamBuffer` | Send on client | Command data. |

---

## 8. Host migration

Host Migration preserves client connection state when the server changes.

| Item | Meaning |
|------|---------|
| Activation | Enabled by default from the 1.7.0 line; older versions required `ENABLE_HOST_MIGRATION`. |
| Reconnection tracking | `NetworkStreamIsReconnected` is added to both connection entities after reconnect. |
| Prespawn Ghosts | Prespawn Ghost IDs are preserved across Host Migration. |

The 1.7.0 line substantially refactored Host Migration API names, so earlier samples need migration before use.

---

## 9. Related docs

- [`16_Netcode Client-Server World & Bootstrap.md`](16_Netcode Client-Server World & Bootstrap.md) — how Netcode worlds are created.
- [`18_Netcode Ghost Snapshot & Synchronization.md`](18_Netcode Ghost Snapshot & Synchronization.md) — why `NetworkStreamInGame` gates snapshot exchange.
- [`21_Netcode RPC.md`](21_Netcode RPC.md) — normal RPCs and approval RPCs.

---

## 10. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Connection succeeds but Ghosts do not appear | Add `NetworkStreamInGame` on both client and server. |
| Data still does not exchange after approval | Confirm `ConnectionApproved` was added to the connection entity. |
| Connection drops when the app loses focus | Set `Application.runInBackground = true`. |
| Protocol version mismatch | Use identical client/server builds and Ghost configurations. |
| Approval times out | Adjust `HandshakeApprovalTimeoutMS` or debug the approval RPC path. |
