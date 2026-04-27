---
title: Managed Object Reference Audit
updated: 2026-04-27
folder: Optimizations and Debugging
---

# Managed Object Reference Audit
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

Managed Unity object references are one of the easiest ways to accidentally pull an ECS system out of the fast path. This audit helps find places where component data, systems, or rendering glue hold `UnityEngine.Object` references in ways that block Burst, jobs, or clean SubScene baking.

Use this after the core ECS migration, or whenever a profiler trace shows managed work where you expected data-oriented code.

---

## 2. What to search for

| Pattern | Why it matters |
|---------|----------------|
| `class .*: IComponentData` | Managed component data; valid but usually not ideal for hot paths. |
| `UnityEngine.Object` | Broad marker for managed Unity object references. |
| `GameObject`, `Transform`, `Component` | Runtime scene-object references usually do not belong in simulation data. |
| `Mesh`, `Material`, `Texture`, `AudioClip` | Often should be `UnityObjectRef<T>` or presentation-side cached data. |
| `Resources.Load` | Runtime asset lookup; often better handled by baking/content systems. |
| `ScriptableObject` | Useful authoring source, but runtime jobs should read baked values or blobs. |

Keep the audit scoped to ECS-facing code. MonoBehaviour-only UI or editor tooling does not need to be forced into DOTS patterns.

---

## 3. Triage rules

| Finding | Likely action |
|---------|---------------|
| Managed component in a hot simulation query | Convert to unmanaged `IComponentData`, `BlobAssetReference<T>`, or `UnityObjectRef<T>`. |
| Managed component used only by presentation | Keep it if it is isolated from Burst/job simulation paths. |
| ScriptableObject read every frame | Bake the needed values into a component or blob asset. |
| Mesh/material reference in ECS data | Use `UnityObjectRef<Mesh>` / `UnityObjectRef<Material>` and dereference in presentation code. |
| GameObject/Transform stored in component data | Replace with ECS data or move the reference to a hybrid presentation bridge. |

---

## 4. Audit example

```csharp
// Suspicious: managed component data used as per-entity visual state.
public class EnemyVisual : IComponentData
{
    public Material Material;
}
```

Prefer:

```csharp
using Unity.Entities;
using UnityEngine;

public struct EnemyVisual : IComponentData
{
    public UnityObjectRef<Material> Material;
}
```

Then dereference in a main-thread presentation system or cache the resolved material in a renderer registry.

---

## 5. Profiler signs

| Profiler symptom | What to check |
|------------------|---------------|
| Simulation system allocates every frame | Managed object lookup or LINQ/collection allocation in `OnUpdate`. |
| Burst is disabled for a system that should be data-only | Managed component or managed API access in the query path. |
| Main thread has many tiny rendering lookups | Cache `UnityObjectRef<T>.Value` results outside the per-entity loop. |
| SubScene rebakes more often than expected | Authoring scripts or object references are creating unnecessary dependencies. |

---

## 6. Validation

1. Re-run the search patterns in §2.
2. Confirm hot simulation systems query only unmanaged components.
3. Check Burst Inspector for systems/jobs expected to compile with Burst.
4. Profile a representative scene and confirm managed allocations are gone from the hot path.
5. Enter Play Mode from a clean Editor launch to verify SubScene baking still resolves the references.

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Converted component still cannot be queried in Burst | A remaining field is managed. Check nested structs too. |
| `UnityObjectRef<T>.Value` returns null in Play Mode | The referenced asset was not baked or loaded. Check the Baker and SubScene/content workflow. |
| Visual system is still slow after conversion | You are dereferencing per entity every frame. Cache by shared asset or bake a compact lookup key. |
| ScriptableObject changes do not affect runtime data | The values were baked. Reimport/rebake the SubScene or add proper baking dependencies. |
| Editor-only references fail in player builds | Move editor APIs behind `#if UNITY_EDITOR` and bake runtime-safe data. |
