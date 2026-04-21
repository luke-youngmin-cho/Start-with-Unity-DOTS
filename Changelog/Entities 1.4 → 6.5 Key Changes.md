---
title: Entities 1.4 → 6.5 Key Changes
updated: 2026-04-21
folder: Changelog
---

# Entities 1.4 → 6.5 Key Changes
### Unity 6000.5 · Entities 6.5.0

> A migration reader for moving from **Entities 1.4** (last Package Manager line) to **Entities 6.5** (Core Package, shipped with Unity 6000.5+). Two versioning tracks overlap here and are easy to confuse, so the next section pulls them apart first.

---

## 1. Two versioning tracks to keep separate

There are **two independent version numbers** at play. Read them as two axes, not one.

### Track A — `com.unity.entities` package

| Track | Distribution | Version example | Latest on docs |
|-------|--------------|-----------------|----------------|
| **1.x** | Package Manager (`com.unity.entities`) | 1.4.x | 1.4.5 (2026-02-16) |
| **6.x** | **Core Package** (ships with the Editor) | 6.4 / 6.5 | 6.5.0 (2025-10-22) |

The 6.x package numbers align with the Editor (`Entities 6.4` ↔ Unity 6000.4, `6.5` ↔ 6000.5).

### Track B — Unity Editor (engine-wide)

Unity Editor 6.x (6000.x) is the engine version. Its release notes track **engine-wide** changes — including the `InstanceID` → `EntityId` migration, which is **not** an Entities-package feature but a change to `UnityEngine.Object` identity used across the whole engine.

| Unity Editor | EntityId status |
|--------------|-----------------|
| 6000.3 | `EntityId` type introduced in `UnityEngine`; legacy `InstanceID` APIs start getting obsolete marks (OnOpenAsset int overload, `Selection.instanceIDs`, etc.) |
| 6000.4 | Obsolete range extended; more APIs get `EntityId` overloads |
| 6000.5 | **Major breaking changes**: implicit `int ↔ EntityId` conversion marked obsolete, `FindObjectsByType` + `FindObjectsSortMode` deprecated (EntityId can't imply creation order due to version reuse), Profiler fields (`objectInstanceId`, `parentId`, etc.) replaced, `EntityId` announced to grow from 4 → 8 bytes |
| 6000.6α | Implicit `int → EntityId` cast removed, `InstanceIDToObject(int)` becomes a compile error |

These two tracks intersect in practice — your project pins a specific Unity Editor and inherits whichever Entities package ships with it. Pre-1.4 package lines also live independently on older Editors. This manual targets **Unity Editor 6000.5 + Entities package 6.5**.

---

## 2. Entities package 6.4.0 — the Core Package transition (2025-10-16)

The single headline change from the package changelog:

> **"Package is now a core package embedded in Unity."**

It applies to Entities, Collections, Mathematics, and Entities Graphics. What it means in practice:

| Area | What changed |
|------|-------------|
| **Distribution** | Shipped with Unity 6000.4+ Editor. No `com.unity.entities` entry needed in `manifest.json`. |
| **Versioning** | Aligns with the Editor (Entities 6.4 ↔ Unity 6000.4). |
| **Release cadence** | Follows the Editor release schedule; future change logs migrate toward Editor release notes. |
| **Ecosystem** | Packages that depended on a pinned Entities UPM version transition with it (e.g. Netcode for Entities 6.5 is also a Core Package). |

This is a **distribution / packaging** change. It is independent of any API or behaviour change.

---

## 3. Entities package 6.5.0 — placeholder release (2025-10-22)

The package changelog marks 6.5.0 as simply `"Placeholder for 6.5.0"`. It is a version rename that aligns the package with Unity Editor 6000.5. **There are no substantive package-level API changes in this release.**

However, if you are moving a project onto Unity Editor 6000.5, the **engine-wide** changes in §4 below absolutely affect your code even though the package changelog is empty.

---

## 4. Unity Editor 6000.5 — engine-wide breaking changes

These sit in the Editor release notes, not the Entities package changelog:

| Change | Impact on ECS codebases |
|--------|------------------------|
| Implicit `int ↔ EntityId` conversion marked **obsolete** | Any code passing `InstanceID` as `int` to an `EntityId`-typed parameter (or vice versa) now emits a deprecation warning. Will become a compile error in 6000.6. |
| `FindObjectsByType(FindObjectsSortMode.InstanceID)` deprecated | `EntityId` doesn't preserve creation order (the underlying handle is versioned and reusable). If you were using `InstanceID` sort to get "newer first," switch to timestamps or your own counter. |
| Profiler API fields replaced | `objectInstanceId`, `parentId`, `instanceIDsCount` etc. have `EntityId`-typed successors. If you wrote custom profiler tooling, audit these. |
| `EntityId` size announced to grow **4 → 8 bytes** | Any native layout that assumed an `int`-sized instance id now needs to plan for a wider field. Affects serialisation, interop, and unsafe pointer math. |

### EntityId — assumptions that never held and are now actively checked

`EntityId` is an opaque handle; its internal representation is not a contract. Code relying on any of the following is either already deprecated or will stop compiling in Unity 6000.6+:

- Casting `InstanceID` **to / from `int`**.
- Using the **sign** of the value to check whether an object is asset-loaded.
- Using `Object.GetHashCode()` to derive an InstanceID.
- Serializing to string via `ToString` and reading back with `int.Parse`.
- **Sorting** to imply creation order (the value is versioned and reused).
- Storing state in the **high/first bit**.

Refactor guidance: [`Migration/03_InstanceID → EntityId.md`](../Migration/03_InstanceID → EntityId.md). Audit checklist: [`Optimizations and Debugging/04_EntityId Audit — Deprecated InstanceID Hunt.md`](../Optimizations and Debugging/04_EntityId Audit — Deprecated InstanceID Hunt.md).

> **Important:** `EntityId` is an engine-level replacement for `InstanceID` on `UnityEngine.Object`. It is **not** the same thing as an ECS `Entity`. See [`../DOTS Workflows/04_Identity Types — Entity · EntityId · UnityObjectRef.md`](../DOTS Workflows/04_Identity Types — Entity · EntityId · UnityObjectRef.md) for the three-way distinction (`Entity` / `EntityId` / `UnityObjectRef<T>`).

---

## 4. Cumulative 1.x changes worth knowing (1.4.0 → 1.4.5)

If you are coming from **pre-1.4** code, these are the notable entries between the 1.4.0 preview line and the latest bug-fix release 1.4.5.

### 1.4.5 (2026-02-16)
- `com.unity.burst` bumped to **1.8.27**.
- **Fixed:** IL post-processor no longer hangs on recursive references in managed components.

### 1.4.4 (2025-12-16)
- Dependency bumps: `mono-cecil` 1.11.6, `profiling.core` 1.0.3, `scriptablebuildpipeline` 1.23.1, `serialization` 3.1.3.
- **Removed:** expensive baking analytics during SubScene import.
- **Fixed:** race condition in `ArchetypeChunk.SetComponentEnabled()` and `SetComponentEnabledForAll()`.
- **Fixed:** `UnityObjectRef` garbage-collection issue.

### 1.4.3 (2025-10-17)
- **Added:** `UNITY_DOTS_IMHEX` script define to generate ImHex pattern files.
- **Changed:** SourceGenerator DLLs compiled with the `Deterministic` flag; Burst bumped to **1.8.25**.
- **Fixed:** `SystemAPI` debug breakpoints with leading whitespace; SubScene deserialization during async scene loading.

### 1.4.2 (2025-09-05)
- USS classes migrated from percentage to pixel sizing.
- **Fixed:** incorrect source generation for public partial `SystemBase`; `SGICE002` error with `SystemAPI` inside `Entities.ForEach` + `WithDeferredPlaybackSystem`; world serialization memory leak.

### 1.4.0 preview line (exp.2 / pre.3 / pre.4)
- **Added:** `ComponentLookup.TryGetRefRW<T>()` / `TryGetRefRO<T>()`.
- **Added:** `DisableBootstrapOverridesAttribute` for `ICustomBootstrap`.
- **Added:** `ArchetypeChunk.GetBufferAccessorRO<T>()` / `GetBufferAccessorRW<T>()` and `GetUntypedBufferAccessorReinterpret<T>()`.
- **Added:** `SystemTypeIndex` retrieval from `SystemHandle`; Disabled/Present/Absent/None columns in the System inspector; Dependency tab; prefab distinction in the Query window.
- **Changed:** `ChunkEntityEnumerator` performance improved.
- **Changed:** `Entities.ForEach` and `IAspect` marked **obsolete** — use `IJobEntity` and `SystemAPI.Query<T>` instead of `ForEach`, and direct component access instead of aspects.
- **Deprecated:** `ComponentLookup.GetRefRWOptional()` / `GetRefROOptional()` (use the `TryGet...` variants above).

---

## 5. Cumulative breaking changes — a cheat sheet

| Track | Area | Change | Where to migrate |
|-------|------|--------|-----------------|
| **Engine** (6000.3+) | Identity | `InstanceID` → `EntityId` — obsolete on 6000.5, compile errors on 6000.6 | [`Migration/03_InstanceID → EntityId.md`](../Migration/03_InstanceID → EntityId.md) |
| **Engine** (6000.5) | Object queries | `FindObjectsByType(FindObjectsSortMode.InstanceID)` deprecated | Same page |
| **Engine** (6000.5) | Profiler | `objectInstanceId`, `parentId`, `instanceIDsCount` replaced by `EntityId`-typed successors | Same page |
| **Entities package** (6.4) | Distribution | Package Manager → Core Package | [`Migration/02_Package Manager → Core Package.md`](../Migration/02_Package Manager → Core Package.md) |
| **Entities package** (1.4+) | Iteration | `Entities.ForEach` obsolete — removal planned for Entities 2.0 | [`Migration/04_foreach → IJobEntity.md`](../Migration/04_foreach → IJobEntity.md) |
| **Entities package** (1.4+) | Abstraction | `IAspect` obsolete — removal planned for Entities 2.0 | [`Migration/05_IAspect Removal.md`](../Migration/05_IAspect Removal.md) |
| **Entities package** (1.4+) | Optional refs | `ComponentLookup.GetRefRWOptional()` / `GetRefROOptional()` deprecated | Use `TryGetRefRW<T>()` / `TryGetRefRO<T>()` |

---

## 6. References

- [Entities 6.5 Changelog](https://docs.unity3d.com/Packages/com.unity.entities@6.5/changelog/CHANGELOG.html)
- [Entities 1.4 Changelog](https://docs.unity3d.com/Packages/com.unity.entities@1.4/changelog/CHANGELOG.html)
- [ECS Development Status — December 2025 (Unity Discussions)](https://discussions.unity.com/t/ecs-development-status-december-2025/1699284)
- [CoreCLR Scripting & ECS Status Update — March 2026 (Unity Discussions)](https://discussions.unity.com/t/coreclr-scripting-and-ecs-status-update-march-2026/1711852)
