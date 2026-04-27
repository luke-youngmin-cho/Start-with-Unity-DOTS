---
title: Profiler · Bottleneck Analysis
updated: 2026-04-27
folder: Optimizations and Debugging
---

# Profiler · Bottleneck Analysis
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

The Unity Profiler (`Window → Analysis → Profiler`) is the primary tool for "why is this frame slow" questions in DOTS. Entities emits enough markers that you can attribute time to specific systems, jobs, and chunks — once you know where to look.

This page walks through reading the profile, the typical DOTS bottleneck patterns, and how to act on them.

---

## 2. Before you profile

1. **Turn on Deep Profile sparingly.** It is useful for tracking down a specific managed path but distorts timings — never trust Deep Profile for perf numbers.
2. **Enable "Record in Editor & Standalone"** so you capture builds, not just Editor frames.
3. **Set the Profiler to target the player**, not the Editor, when measuring released performance. Editor overhead is significant.
4. **Play for a few seconds before reading**; the first few frames include domain reload and JIT work.

---

## 3. Reading the CPU Usage module

Top-level markers under a DOTS frame:

```
PlayerLoop
├── Initialization.*
├── Update.ScriptRunBehaviourUpdate              (MonoBehaviour.Update)
├── SimulationSystemGroup                        ← DOTS simulation lives here
│   ├── SystemA.OnUpdate
│   ├── SystemB.OnUpdate
│   └── EndSimulationEntityCommandBufferSystem.Playback
├── FixedStepSimulationSystemGroup
├── PresentationSystemGroup
├── Render
└── WaitForTargetFPS
```

Drill into a `SystemA.OnUpdate` marker to see its subcalls — `ScheduleParallel`, job completion waits, `EntityManager` calls.

---

## 4. Where the time actually goes — common shapes

### 4.1 Main-thread `OnUpdate` dominant

If a single `SystemX.OnUpdate` accounts for 10+ ms, the work is running on the main thread. Check:

- Is it using `SystemAPI.Query` in a tight loop that should have been `IJobEntity.ScheduleParallel`?
- Is it a `SystemBase` forced to the main thread by a managed field?
- Is `EntityManager.AddComponent/RemoveComponent` being called per entity?

### 4.2 Job timings

Jobs show up in the **Timeline** view (or under "Workers" in the Hierarchy view). Look for:

- A job lane fully saturated while others are idle → work isn't spreading across chunks.
- Main-thread waiting on a job (`JobHandle.Complete()` marker) → `.Complete()` called too early; move it later in the frame.
- Huge Burst function names → normal, the symbol is the full generic instantiation.

### 4.3 `EntityManager` hotspots

Look for markers named:

- `EntityManager.SetArchetype` / `Move Entity` — structural change churn. See [`13_Structural Change & Safety.md`](../DOTS Workflows/13_Structural Change & Safety.md).
- `EntityCommandBufferSystem.OnUpdate (Playback)` — expensive ECB playback. Usually means too many commands recorded per frame.
- `TypeManager.Initialize` — only at startup; if it appears per-frame, something is reloading the world.

### 4.4 `GC.Alloc` markers

Any `GC.Alloc` in the simulation is a regression in DOTS. Sources:

- A `SystemBase` allocating in its `OnUpdate`.
- A `foreach` over a managed collection inside a system.
- Boxing — e.g. `object.ToString()` on a struct.

Burst systems should show zero allocations in the Profiler's Memory module.

---

## 5. Profiler modules to pin

- **CPU Usage** — timeline and hierarchy of markers.
- **Memory** — allocated heap; watch for climbing `GC.Used Memory`.
- **Entities** (if available) — a DOTS-specific module showing per-system timings over time.

The Entities Profiler module arrived with 1.8.0 / 6.x and aligns with the Systems window on a timeline. If you see multi-frame spikes, this is the fastest way to attribute them.

For networked projects, pair this with [`../DOTS Workflows/24_Netcode Profiler & Debugging.md`](../DOTS Workflows/24_Netcode Profiler & Debugging.md) to inspect Ghost snapshots, prediction ticks, and Netcode-specific bandwidth.

---

## 6. Common bottlenecks and fixes

| Pattern | Symptom in profiler | Fix |
|---------|--------------------|-----|
| Per-entity `Entities.ForEach` (legacy) | Main-thread-heavy `OnUpdate` with many small samples | Port to `IJobEntity.ScheduleParallel`. |
| Per-frame `AddComponent`/`RemoveComponent` | `Move Entity` marker dominates | Convert state to an enableable component. |
| ECB overload | `...ECBS.Playback` taking ms | Batch commands (one per chunk instead of per entity), or raise the batch size. |
| Shared-component value explosion | `SetSharedComponent` dominates | Bucket the value (quantize to fewer distinct keys). |
| `.Complete()` called mid-frame | Long `JobHandle.Complete` wait on main thread | Postpone the completion to the end of the frame, or restructure so the main thread doesn't need the result yet. |
| Burst fallback | Hot method shows unmangled C# name | A captured non-blittable type fell back to IL. Check the Burst Inspector. |
| Archetype fragmentation | Many small chunks; `EntityManager.*Query` markers high | Archetypes too granular — consolidate tags / use enableable components. |
| Bake-time regression | `Bake` markers during Play | Domain reload re-triggered baking. Usually Editor-only; confirm in a build. |

---

## 7. Frame budget worked example

Imagine a 16.6 ms budget (60 Hz). After ruling out Render:

- `SimulationSystemGroup`: 6 ms
  - `EnemyAISystem.OnUpdate`: 2.1 ms (Burst, parallel jobs on 4 workers)
  - `MovementSystem.OnUpdate`: 1.4 ms (Burst, parallel)
  - `EndSimulationECBS.Playback`: 1.8 ms ← suspicious
  - everything else: 0.7 ms

`1.8 ms` of ECB playback for what should be a small number of structural changes is usually the biggest lever. Check command counts:

1. Entities Journaling for 1 frame → counts of `Instantiate`/`AddComponent`/`DestroyEntity` per system.
2. If a system records hundreds of commands every frame, batch them (one command per chunk, or switch to a parallel recorder with chunk-index sort keys).

---

## 8. Standalone Profiling

For a real build:

1. **File → Build Profiles → Development Build + Autoconnect Profiler**.
2. Player runs, Profiler captures from the remote process.
3. Results are more honest about what your players will see. Editor measurements can be 2-5× slower for reasons unrelated to your code.

---

## 9. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Profile shows no DOTS markers | Profiler module is filtering them out — enable "Scripts" and "Jobs" in the module settings. |
| Frame is slow but no hot marker | Could be render-thread wait. Enable the GPU module and the timeline view. |
| Deep Profile shows different hot spots | Deep Profile distorts allocations and call counts. Trust the non-deep profile for relative timing. |
| Burst method shows managed call in stack | Fallback — Burst couldn't compile it. Check the Burst Inspector output. |
| `GC.Collect` marker spikes | Managed allocations somewhere in a system. Narrow down with Memory module → Snapshot. |
| System time oscillates wildly between frames | Work grows with entity count and chunks are entering/leaving the query each frame. Check structural churn. |
