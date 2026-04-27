---
title: Netcode for Entities 1.4 → 6.5 Key Changes
updated: 2026-04-27
folder: Changelog
---

# Netcode for Entities 1.4 → 6.5 Key Changes
### Unity 6000.5 · Entities 6.5.0 · Netcode for Entities 6.5.0

> A migration reader for Netcode for Entities changes accumulated from the 1.4 line to the 6.5 Core Package line. Netcode 6.5.0 moved into the Unity 6000.5 Editor as a built-in package, so later entries move toward Editor release notes rather than this package changelog.

---

## 1. Version highlights

### 6.5.0 (2026-02-05) — built-in Editor package

| Area | Change |
|------|--------|
| Distribution | Netcode moved to a built-in Editor package starting with Unity 6000.5. |
| Entities dependency | Changelog line lists `com.unity.entities` **1.4.2**. Keep this distinct from the DOTS manual's Unity 6000.5 / Entities Core Package target. |
| Profiler | Ghost Snapshot View TreeView labels gained context menus for quick prefab/component inspection. |
| Source Generator | IDE-side Source Generator execution was disabled for Rider and Visual Studio, improving IDE performance. |
| Breaking | Disconnect ECB changed from `BeginSimulationECBS` to `NetworkGroupCommandBufferSystem`. |

### 1.12.0 (2026-02-01)

| Area | Change |
|------|--------|
| Profiler | Tick-based navigation, search fields, and client Prediction / Interpolation tick display. |
| Entities dependency | Bumped from 1.4.2 to **1.4.4**. |
| Deprecated | Browser-based Network Debugger deprecated in favor of Netcode Profiler. |
| Removed | `NETCODE_PROFILER_ENABLED` is no longer required. |
| Breaking | `GhostAuthoringComponentBaker` `TransformUsageFlags` behavior changed. |

### 1.11.0 (2025-12-12)

| Area | Change |
|------|--------|
| Prefab List menu | Quick Ghost Prefab inspection. |
| GameObject Ghost | Work started for GameObject-layer Ghost support. |
| Static Ghost optimization | Static Ghost job timing reduced substantially for unchanged data. |
| Ghost Migration | `IncludeInMigration` supports non-Ghost data migration. |
| Bool serialization | Packed / unpacked bool serialization costs improved. |
| API breaking | `PrefabDebugName.Name` fully deprecated. |

### 1.10.0 (2025-11-09)

| Area | Change |
|------|--------|
| Zero-sized components | World state save supports zero-sized components. |
| Source Generator | IDE-side execution disabled; reflected in 6.5.0 behavior. |
| Fixes | `PredictedGhostSpawnSystem` assert, Sleep mode warning spam, `AlwaysRun` prediction-loop exception, and related fixes. |

### 1.9.x (2025-09 to 2025-10)

| Area | Change |
|------|--------|
| Ghost optimization docs | Dedicated Ghost optimization documentation added. |
| Stabilization | 1.8.0 Profiler Modules and Importance Visualizer stabilized. |

### 1.8.0 (2025-08-17)

| Area | Change |
|------|--------|
| Importance Visualizer | Added to PlayMode Tool. |
| Forced Input Latency | `ClientTickRate.ForcedInputLatencyTicks` added. |
| Profiler Modules | Experimental Server / Client Profiler modules for Unity 6+. |
| Breaking | `GhostDistanceImportance` scale functions no longer multiply `baseImportance` by 1000. |
| Transport | Updated to `com.unity.transport` **2.5.3**. |

### 1.7.0 (2025-07-29)

| Area | Change |
|------|--------|
| Host Migration | `ENABLE_HOST_MIGRATION` define removed; Host Migration enabled by default. |
| Host Migration API | Class and method names refactored; use the current package documentation/samples for exact type names. |
| Serialization | Ghost component serialization improved for Host Migration. |
| Transport | Updated to `com.unity.transport` **2.1.0**. |

### 1.6.1 to 1.6.2 (2025-05 to 2025-07)

| Area | Change |
|------|--------|
| Predicted ECB | Added begin/end ECB systems for `PredictedSimulationSystemGroup`. |
| Predicted spawning | Added `PredictedSpawningSystemGroup` after `EndPredictedSimulationCommandBufferSystem`. |
| Host Migration | Experimental feature hidden behind `ENABLE_HOST_MIGRATION`. |
| Reconnect tracking | `NetworkStreamIsReconnected` component added. |
| Snapshot history | `GhostSystemConstants.SnapshotHistorySize` configurable by compiler define. |
| Relevancy + Importance | Fast-path support for combining Ghost Relevancy and Importance Scaling. |

### 1.5.0 (2025-04-22)

| Area | Change |
|------|--------|
| Thin Clients | `AutomaticThinClientWorldsUtility` added for runtime Thin Client creation. |
| Serialization | FixedList and unsafe fixed-buffer replication support. |
| Baselines | `GhostAuthoringComponent.UseSingleBaseline` added for CPU savings. |
| Integer serialization | byte / short-specific templates. |
| Lag Compensation | Physics History Buffer increased from 16 to **32** ticks. |
| Relay | Relay package path shifted toward `com.unity.services.multiplayer` v1.1.0. |

### 1.4.0 (2024-11-14)

| Area | Change |
|------|--------|
| Send rate | `GhostAuthoringComponent.MaxSendRate` limits Ghost send frequency. |
| Chunk iteration | `GhostSendSystemData.MaxIterateChunks` limits chunks processed per tick. |
| `NetCodeConfig` | Added several Unity Transport `NetworkConfigParameters`. |
| Snapshot ACK | ACK history capacity became configurable. |
| Fixes | Snapshot ACK issues past 256 ticks, physics interpolation jitter, and related fixes. |

---

## 2. Cumulative breaking changes

| Version | Breaking change |
|---------|-----------------|
| **6.5.0** | Disconnect ECB changed from `BeginSimulationECBS` to `NetworkGroupCommandBufferSystem`. |
| **1.12.0** | `GhostAuthoringComponentBaker` `TransformUsageFlags` behavior changed. |
| **1.11.0** | `PrefabDebugName.Name` fully deprecated. |
| **1.8.0** | `GhostDistanceImportance` multiplier behavior changed. |
| **1.7.0** | Host Migration API class and method names refactored. |
| **1.5.0** | Command validation tightened and handshake timeout behavior changed. |
| **1.4.0** | `MaxSendChunks` scope changed. |

---

## 3. Trends to remember

1. **Profiler-first debugging** — the old browser Network Debugger is deprecated; use Netcode Profiler modules.
2. **Host Migration became default** — experimental in 1.6.1, enabled by default in 1.7.0+.
3. **IDE performance improved** — Netcode Source Generators no longer run inside IDE analysis.
4. **Serialization became more efficient** — bool packing, byte/short templates, and single-baseline options reduce CPU/bandwidth pressure.
5. **Core Package transition** — Netcode 6.5.0 ships with Unity 6000.5; future changes move toward Editor release notes.

---

## 4. Related docs

- [`../DOTS Workflows/16_Netcode Client-Server World & Bootstrap.md`](../DOTS Workflows/16_Netcode Client-Server World & Bootstrap.md)
- [`../DOTS Workflows/18_Netcode Ghost Snapshot & Synchronization.md`](../DOTS Workflows/18_Netcode Ghost Snapshot & Synchronization.md)
- [`../DOTS Workflows/22_Netcode Ghost Optimization · Importance · Relevancy.md`](../DOTS Workflows/22_Netcode Ghost Optimization · Importance · Relevancy.md)
- [`../DOTS Workflows/24_Netcode Profiler & Debugging.md`](../DOTS Workflows/24_Netcode Profiler & Debugging.md)

---

## 5. References

- [Netcode for Entities 6.5 Manual](https://docs.unity3d.com/Packages/com.unity.netcode@6.5/manual/index.html)
- [Netcode for Entities 6.5 Changelog](https://docs.unity3d.com/Packages/com.unity.netcode@6.5/changelog/CHANGELOG.html)
- [Netcode for Entities 1.12 Changelog](https://docs.unity3d.com/Packages/com.unity.netcode@1.12/changelog/CHANGELOG.html)
