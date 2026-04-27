---
title: Netcode Client-Server World & Bootstrap
updated: 2026-04-27
folder: DOTS Workflows
---

# Netcode Client-Server World & Bootstrap
### Unity 6000.5 · Entities 6.5.0 · Netcode for Entities 6.5.0

---

## 1. Overview

Netcode for Entities uses a **client-server model** and separates client and server logic into different ECS worlds.

| World | Role |
|-------|------|
| **Server World** | Authoritative simulation. |
| **Client World** | Local player simulation, prediction, interpolation, and presentation. |
| **Thin Client World** | Dummy client used for development and load testing. |

When a client device also hosts the server, the project is running a **client-hosted server** configuration.

---

## 2. World types and system filtering

### 2.1 `WorldSystemFilter`

Use `WorldSystemFilter` to declare which Netcode world a system can run in.

```csharp
using Unity.Entities;

[WorldSystemFilter(WorldSystemFilterFlags.ServerSimulation)]
public partial struct ServerOnlySystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs only in the server world.
    }
}
```

| Flag | Meaning |
|------|---------|
| `LocalSimulation` | Local world without Netcode systems. |
| `ServerSimulation` | Server simulation world. |
| `ClientSimulation` | Client simulation world. |
| `ThinClientSimulation` | Thin client simulation world. |

### 2.2 System-group filtering

Some Netcode groups only exist in specific worlds, so placing a system in the group also filters the world.

```csharp
using Unity.Entities;
using Unity.NetCode;

[UpdateInGroup(typeof(GhostInputSystemGroup))]
public partial struct MyInputSystem : ISystem
{
    // GhostInputSystemGroup exists in Client and Local worlds.
    // The system is excluded from Server worlds.
}
```

| System group | Worlds where it exists |
|--------------|------------------------|
| `GhostInputSystemGroup` | Client, Local |
| `PredictedSimulationSystemGroup` | Client, Server |
| `PresentationSystemGroup` | Client only; not Server or Thin Client |

---

## 3. Bootstrap process

`ClientServerBootstrap` creates Server and Client worlds at runtime.

### 3.1 Default behavior

The default bootstrap creates both a Server World and a Client World on startup. It does **not** automatically connect them, so the game still controls when to listen and connect.

### 3.2 Custom bootstrap

```csharp
using Unity.NetCode;

public class GameBootstrap : ClientServerBootstrap
{
    public override bool Initialize(string defaultWorldName)
    {
        // Create only the default local world.
        CreateLocalWorld(defaultWorldName);
        return true;
    }
}
```

You can then create worlds from a play button, lobby flow, test harness, or command line.

```csharp
using Unity.NetCode;

void OnPlayButtonClicked()
{
    var serverWorld = ClientServerBootstrap.CreateServerWorld("ServerWorld");
    var clientWorld = ClientServerBootstrap.CreateClientWorld("ClientWorld");

    AutomaticThinClientWorldsUtility.NumThinClientsRequested = 4;
    AutomaticThinClientWorldsUtility.BootstrapThinClientWorlds();
}
```

`AutomaticThinClientWorldsUtility` was added in the 1.5.0 line and is used to create or clean up Thin Client worlds at runtime.

---

## 4. Tick-rate configuration

Configure server simulation speed with the `ClientServerTickRate` singleton.

| Property | Default | Meaning |
|----------|---------|---------|
| `SimulationTickRate` | 60 | Server simulation ticks per second. |
| `NetworkTickRate` | Same as `SimulationTickRate` | Snapshot send rate in ticks per second. |
| `MaxSimulationStepsPerFrame` | 4 | Maximum simulation steps per frame. |
| `MaxSimulationStepBatchSize` | 4 | Number of ticks that can be batched with scaled `deltaTime`. |
| `TargetFrameRateMode` | `BusyWait` | `BusyWait`, `Sleep`, or `Auto`. |

```csharp
using Unity.Entities;
using Unity.NetCode;
using UnityEngine;

public class TickRateAuthoring : MonoBehaviour
{
    public int SimulationTickRate = 60;
    public int NetworkTickRate = 30;

    class Baker : Baker<TickRateAuthoring>
    {
        public override void Bake(TickRateAuthoring authoring)
        {
            var entity = GetEntity(TransformUsageFlags.None);
            AddComponent(entity, new ClientServerTickRate
            {
                SimulationTickRate = authoring.SimulationTickRate,
                NetworkTickRate = authoring.NetworkTickRate,
            });
        }
    }
}
```

Lowering `NetworkTickRate` below `SimulationTickRate` saves bandwidth. In that setup, `GhostSendSystem` load is spread across multiple simulation ticks with round-robin scheduling.

---

## 5. Server and client update flow

```text
Server World
└─ Fixed timestep, 60 Hz default
   └─ SimulationSystemGroup
      ├─ GhostSendSystem
      └─ CommandReceiveSystem

Client World
├─ Dynamic timestep
│  └─ SimulationSystemGroup
│     ├─ GhostReceiveSystem
│     └─ CommandSendSystem
├─ Fixed timestep, same as the server
│  └─ PredictedSimulationSystemGroup
│     └─ Prediction code and rollback loop
└─ PresentationSystemGroup
   └─ Rendering and interpolation
```

The server uses a fixed timestep for deterministic authoritative simulation. The client uses a dynamic timestep for normal simulation and presentation, while prediction code runs in the fixed predicted loop.

---

## 6. World migration

Use `DriverMigrationSystem` when a world transition must preserve connection state.

```csharp
using Unity.Entities;
using Unity.NetCode;

public World MigrateServerWorld(World sourceWorld)
{
    DriverMigrationSystem migrationSystem = null;
    foreach (var world in World.All)
    {
        if ((migrationSystem = world.GetExistingSystemManaged<DriverMigrationSystem>()) != null)
            break;
    }

    var ticket = migrationSystem.StoreWorld(sourceWorld);
    sourceWorld.Dispose();

    var newWorld = ClientServerBootstrap.CreateServerWorld("NewServerWorld");
    var newMigrationSystem = newWorld.GetExistingSystemManaged<DriverMigrationSystem>();
    newMigrationSystem.LoadWorld(ticket);
    return newWorld;
}
```

---

## 7. Related docs

- [`03_ECS Core Concepts.md`](03_ECS Core Concepts.md) — World, EntityManager, archetype, and chunk basics.
- [`09_System Group & Update Order.md`](09_System Group & Update Order.md) — base ECS system groups that Netcode extends.
- [`17_Netcode Network Connection & Approval.md`](17_Netcode Network Connection & Approval.md) — listen, connect, approve, and enter in-game state.

---

## 8. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| A system runs on the server when it should not | Add a `WorldSystemFilter` or move it into the correct Netcode system group. |
| Tick-rate settings do not apply | Check that `ClientServerTickRate` was baked into a loaded SubScene. |
| Thin Clients are not created | Confirm `AutomaticThinClientWorldsUtility.BootstrapThinClientWorlds()` is called after setting `NumThinClientsRequested`. |
| Client and server connect automatically | Check whether `AutoConnectPort` is set in the bootstrap. |
| Presentation code errors in Play mode | Rendering systems are running in a server world. Add a `ClientSimulation` filter or move them to `PresentationSystemGroup`. |
