---
title: Netcode Profiler & Debugging
updated: 2026-04-27
folder: DOTS Workflows
---

# Netcode Profiler & Debugging
### Unity 6000.5 · Entities 6.5.0 · Netcode for Entities 6.5.0

---

## 1. Overview

Netcode for Entities provides a **Netcode Profiler** for analyzing connection state, snapshots, prediction, interpolation, and bandwidth. In the 1.12.0 line, the browser-based Network Debugger was deprecated in favor of the built-in Unity Profiler modules.

---

## 2. Netcode Profiler

### 2.1 Access

Unity Profiler includes two Netcode modules on Unity 6.0+.

| Module | What to inspect |
|--------|-----------------|
| **Server Profiler** | Server-side Ghost send, snapshot composition, and bandwidth. |
| **Client Profiler** | Client receive, prediction ticks, and interpolation state. |

### 2.2 Key features

| Feature | Meaning |
|---------|---------|
| **Ghost Snapshot View** | Tree view of per-Ghost snapshot size and composition. |
| **Tick navigation** | Move between frames where snapshots were sent or received. |
| **Search fields** | Search Ghost Snapshot TreeView and Prediction Error List. |
| **Client ticks** | Visualize Prediction Tick and Interpolation Tick. |
| **Context menu** | Inspect Ghost Prefabs and Components from TreeView labels in the 6.5.0 line. |
| **Average per-entity column** | Shows average data size per entity. |

### 2.3 `NETCODE_PROFILER_ENABLED`

The 1.12.0 line removed the need for the `NETCODE_PROFILER_ENABLED` scripting define. The Profiler is available without that define.

---

## 3. Network Debugger

The browser-based Network Debugger is deprecated in the 1.12.0 line. Use the Netcode Profiler instead.

The old flow was **Multiplayer → Open NetDbg**, which opened a browser-based view for snapshot composition, Ghost type distribution, and size analysis.

---

## 4. Multiplayer PlayMode Window

The Multiplayer PlayMode Window is the main editor workflow for testing client/server setups in Play mode.

| Feature | Meaning |
|---------|---------|
| **Play Mode Type** | Choose Client + Server, Client Only, or Server Only. |
| **Thin Client count** | Configure dummy clients. |
| **Simulator** | Simulate network latency and packet loss. |
| **Importance Visualizer** | Visualize Importance Scaling in the 1.8.0+ line. |

---

## 5. Debugging checklists

### 5.1 Connection

```text
□ Application.runInBackground = true
□ Protocol versions match between client and server
□ NetworkStreamInGame is present
□ Firewall and port are open
□ Unity Transport version is compatible
```

### 5.2 Ghost synchronization

```text
□ GhostAuthoringComponent is present
□ GhostField attributes are correct
□ Quantization is appropriate
□ Smoothing is configured
□ Ghost prefab is registered on both sides
```

### 5.3 Prediction

```text
□ Systems run in PredictedSimulationSystemGroup
□ Queries include Simulate
□ Simulation code is deterministic enough for prediction
□ Client and server run the same simulation code
□ Prediction smoothing is registered where needed
```

### 5.4 Input and commands

```text
□ Input is gathered in GhostInputSystemGroup
□ AutoCommandTarget requirements are met, or CommandTarget is set manually
□ Command payload is 1024 bytes or less
□ One-shot input uses InputEvent
```

---

## 6. Prefab List menu

The 1.11.0+ line includes a Prefab List menu for quickly listing and inspecting registered Ghost Prefabs in the project or scene hierarchy.

---

## 7. Useful debugging components

| Component | Use |
|-----------|-----|
| `NetworkStreamConnection` | Inspect connection state. |
| `NetworkId` | Inspect client ID. |
| `GhostInstance` | Inspect Ghost type and ID. |
| `PredictedGhost` | Inspect prediction state and applied tick. |
| `NetworkSnapshotAck` | Inspect snapshot ACK state and estimated RTT. |
| `GhostOwner` | Inspect Ghost owner ID. |

### 7.1 Entity Inspector

In Entities Hierarchy, select a Ghost entity to inspect:

- live `GhostField` values,
- prediction tick data,
- network connection state.

---

## 8. Packet dumps

Use packet dumps when low-level network packet analysis is required. In the 1.6.2 line, packet timestamp logs gained `UnityEngine.Time.frameCount`, which makes frame-level correlation easier.

---

## 9. Common bottlenecks

| Area | Where to look |
|------|---------------|
| Snapshot bandwidth | Profiler Ghost Snapshot View; inspect per-entity size. |
| Prediction re-simulation | Profiler Client Tick; inspect predicted tick count. |
| `GhostSendSystem` | Profiler Timeline; inspect system duration. |
| Serialization cost | Check GhostField write access and change frequency. |
| Physics re-simulation | Inspect `PredictedFixedStepSimulationSystemGroup` duration. |

---

## 10. Editor Analytics

The 1.12.0 line collects editor analytics for Netcode Profiler usage patterns.

---

## 11. Related docs

- [`Optimizations and Debugging/03_Profiler · Bottleneck Analysis.md`](../Optimizations and Debugging/03_Profiler · Bottleneck Analysis.md) — general DOTS profiling workflow.
- [`22_Netcode Ghost Optimization · Importance · Relevancy.md`](22_Netcode Ghost Optimization · Importance · Relevancy.md) — bandwidth controls.
- [`23_Netcode Physics Integration & Lag Compensation.md`](23_Netcode Physics Integration & Lag Compensation.md) — predicted physics costs.

---

## 12. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| No Netcode Profiler data | Confirm the test is running in a client/server Play Mode and using Unity 6.0+. |
| Ghost Snapshot View is large but gameplay feels fine | Look for low-priority Ghosts with excessive fields or send rate. |
| Prediction errors spike | Check `Simulate` queries, deterministic code paths, and input tick handling. |
| Network Debugger workflow is missing | It is deprecated; use Unity Profiler Netcode modules. |
| Packet-level logs are hard to correlate | Use packet dumps with frame-count timestamps in the 1.6.2+ line. |
