---
title: Entities 1.4 → 6.5 Key Changes
updated: 2026-04-27
folder: Changelog
---

# Entities 1.4 → 6.5 Key Changes
### Unity 6000.5 · Entities 6.5.0

> A scoped summary of the official `com.unity.entities` package changelog entries that matter when moving an ECS project from the 1.4 line to the 6.5 Core Package line. This page intentionally covers Entities package changes only.

---

## 1. Version tracks

Entities 6.x aligns with Unity Editor 6000.x and ships as a Core Package. Older Entities 1.x versions were Package Manager packages.

| Track | Distribution | Example |
|-------|--------------|---------|
| Entities 1.x | Package Manager (`com.unity.entities`) | 1.4.2 in the official 6.5 changelog history. |
| Entities 6.x | Core Package embedded in the Editor | 6.4.0 / 6.5.0. |

This distinction is about package distribution and versioning. It is not a license to pull unrelated UnityEngine or Editor-wide migrations into DOTS programming docs.

---

## 2. Entities 6.5.0

The official package changelog entry for Entities 6.5.0 is:

> "Placeholder for 6.5.0"

Treat 6.5.0 as the Unity 6000.5-aligned package version. The package changelog does not list new Entities API surface for this release.

---

## 3. Entities 6.4.0

The official package changelog entry for Entities 6.4.0 says:

> "Package is now a core package embedded in Unity."

Practical effects:

| Area | Change |
|------|--------|
| Installation | `com.unity.entities` is no longer added manually to `manifest.json` on Unity 6000.4+. |
| Version choice | The Entities version comes from the Editor install. |
| Lock file | `packages-lock.json` marks the package as built-in/core rather than a registry dependency. |
| Upgrade workflow | Remove old Package Manager entries and let the Editor regenerate package resolution. |

See [`../Migration/02_Package Manager → Core Package.md`](../Migration/02_Package Manager → Core Package.md).

---

## 4. Entities 1.4.2

The official changelog history included with `com.unity.entities@6.5` lists 1.4.2 before the Core Package transition.

### Changed

| Change | Why it matters |
|--------|----------------|
| Burst dependency updated to 1.8.24 | A dependency bump for projects still on the 1.4 line. |
| USS classes changed from percentage to pixel sizing | Editor UI styling detail. |
| Font-size field changed from percentage to pixel sizing | Editor UI styling detail. |

### Fixed

| Fix | Why it matters |
|-----|----------------|
| Incorrect source generation for `public partial SystemBase` | Relevant if older 1.4 code hit source-generation errors. |
| `SGICE002` with `SystemAPI` inside `Entities.ForEach` plus `WithDeferredPlaybackSystem` | Useful context while migrating away from `Entities.ForEach`. |
| World serialization memory leak | Runtime/editor stability. |
| Inspector initialization after exiting Play Mode with an entity selected | Editor tooling stability. |

---

## 5. 1.4.0 pre-release entries worth migrating for

The 1.4.0 preview/pre-release history is where the important ECS API direction appears.

| Area | Official change | Migration target |
|------|-----------------|------------------|
| Iteration | `Entities.ForEach` marked obsolete; future major removal planned. | Use `IJobEntity` or `SystemAPI.Query`. |
| Aspects | `IAspect` marked obsolete; future major removal planned. | Use direct component/query APIs. |
| Optional component refs | `ComponentLookup.GetRefRWOptional()` / `GetRefROOptional()` deprecated. | Use `TryGetRefRW<T>()` / `TryGetRefRO<T>()`. |
| Chunk buffers | `ArchetypeChunk.GetBufferAccessorRO<T>()` / `GetBufferAccessorRW<T>()` added. | Request the access mode you actually need. |
| Untyped buffers | `GetUntypedBufferAccessorReinterpret<T>()` added. | Use only when source/destination element sizes and buffer capacities are safely aliasable. |
| System handles | `SystemTypeIndex` can be retrieved from `SystemHandle`. | Better low-level system access. |
| Query tooling | Disabled, Present, Absent, None columns; Dependency tab; prefab distinction in Query window. | Better inspection/debugging of queries. |
| Object refs in ECS | Manual documentation for `UnityObjectRef` added. | Use `UnityObjectRef<T>` for managed Unity object references in unmanaged ECS data. |

---

## 6. Migration checklist

- [ ] Remove old Package Manager entries for Core Packages on Unity 6000.4+.
- [ ] Replace remaining `Entities.ForEach` with `IJobEntity` or `SystemAPI.Query`.
- [ ] Replace `IAspect` usage with direct component access.
- [ ] Replace `GetRefRWOptional` / `GetRefROOptional` with `TryGetRefRW<T>()` / `TryGetRefRO<T>()`.
- [ ] Review managed Unity object references and use `UnityObjectRef<T>` or baked data where appropriate.
- [ ] Recheck Systems, Query, and Entity Inspector windows after migration.

---

## 7. References

- [Entities 6.5 Manual](https://docs.unity3d.com/Packages/com.unity.entities@6.5/manual/index.html)
- [Entities 6.5 Changelog](https://docs.unity3d.com/Packages/com.unity.entities@6.5/changelog/CHANGELOG.html)
