---
title: Core Packages Explained
updated: 2026-04-28
folder: Getting Started
---

# Core Packages Explained
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

With the **Entities package 6.4.0** release (shipping alongside Unity Editor 6000.4), the major DOTS packages transitioned from the Package Manager into the Editor itself as **Core Packages**. The exact wording from the package changelog is:

> *"Package is now a core package embedded in Unity."*

This affected `com.unity.entities`, `com.unity.collections`, `com.unity.entities.graphics`, `com.unity.physics`, and (on its matching 6.5 release) Netcode for Entities. `com.unity.mathematics` remains part of the DOTS programming surface and is updated in the Unity 6000.5 package set, but its package changelog does not use the 6.x Core Package wording. These distribution changes affect how you install, version, and track packages, but they are not API changes by themselves.

This page explains:
- Which packages are now Core Packages (and which are not).
- Where their version numbers come from.
- What changes in your `manifest.json` and `packages-lock.json`.
- Why the shift happened.

---

## 2. What is a Core Package?

A **Core Package** is a package that ships **inside the Editor**. You do not add it via the Package Manager, you cannot change its version independently of the Editor, and its release cadence is the Editor's.

| Trait | Package Manager package | Core Package |
|-------|-------------------------|--------------|
| Install step | Add via UPM | None — present with the Editor |
| Version chosen by | You, in `manifest.json` | Unity, per Editor version |
| Release cadence | UPM independent | Editor releases |
| Appears in `manifest.json` | Yes | No |
| Appears in `packages-lock.json` | Yes | Yes, marked as `builtin` |
| Shown in Package Manager | Under "In Project" | Under "In Project" tagged **Built-in** |

---

## 3. The DOTS package map on Unity 6000.5+

| Package | On Unity 6000.5+ | Purpose |
|---------|-----------------|---------|
| **Entities** | **Core Package** (6.5) | ECS runtime: Entity, Component, System, World |
| **Collections** | **Core Package** | Native container types (`NativeList<T>`, `NativeHashMap<K,V>`, etc.) |
| **Mathematics** | DOTS dependency (`com.unity.mathematics` 1.4.x) | Burst-optimized `float3`, `quaternion`, `int4`, etc. |
| **Entities Graphics** | **Core Package** | Renders entity-based graphics through the SRP |
| **Netcode for Entities** | **Core Package** (6.5) | Client-server networking for entities |
| **Unity Physics** | **Core Package** (6.5) | Stateless, deterministic physics for entities |
| **Character Controller** | Package Manager | ECS-based character controller |
| **Burst Compiler** | Package/tooling dependency | Compiles jobs and systems to tight native code |
| **Job System** | Built-in engine feature | Multithreaded job scheduling |

> Note: The Job System is engine-level. Burst remains package/tooling surfaced through Unity's package system, even though DOTS packages depend on it and Unity 6000.x controls the compatible version set.

---

## 4. Version numbers after the transition

Core Package versions **align with the Editor**:

| Editor | Core Entities |
|--------|---------------|
| Unity 6000.4 | Entities 6.4 |
| Unity 6000.5 | Entities 6.5 |

The old 1.x line still exists for projects that cannot yet move to 6000.4+. The two tracks are separate; you upgrade to 6.x by upgrading the Editor.

See [`Changelog/Entities 1.4 → 6.5 Key Changes.md`](../Changelog/Entities 1.4 → 6.5 Key Changes.md) for Entities transition notes and [`Changelog/Netcode for Entities 1.4 → 6.5 Key Changes.md`](../Changelog/Netcode for Entities 1.4 → 6.5 Key Changes.md) for Netcode-specific changes.

---

## 5. Effect on your project files

### `manifest.json`

No `com.unity.entities`, `com.unity.collections`, `com.unity.entities.graphics`, `com.unity.physics`, or `com.unity.netcode` entries are needed on the matching Core Package Editor versions. Avoid direct `com.unity.mathematics` pins unless your project has a specific reason to override the Editor/package-set version. If you upgrade a 1.x project, see [`Migration/02_Package Manager → Core Package.md`](../Migration/02_Package Manager → Core Package.md).

### `packages-lock.json`

Core Packages appear with `"source": "builtin"` instead of a version URL.

### Assembly definitions (.asmdef)

Your `.asmdef` files still reference the assembly names (for example `Unity.Entities`, `Unity.Collections`). Nothing changes here — only the package-distribution layer changed.

---

## 6. Why the shift?

The stated goal is to let the ECS team ship features faster and more incrementally by coupling DOTS updates to the Editor release instead of an independent package schedule. It also lets the Engine rely on ECS internally, which was awkward while the packages were optional.

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Package Manager still lists `Entities` under "Unity Registry" with an old version | You are on a pre-6000.4 Editor. Either upgrade the Editor, or stay on the 1.x line. |
| `manifest.json` resolve error after removing `com.unity.entities` | Delete `packages-lock.json`, let the Editor regenerate it. |
| A sample or asset store package targets `com.unity.entities@1.x` | On 6000.5+ the 1.x API has mostly been superseded; check the Migration folder for equivalents. |
| Two Editor installs report different Entities versions | Expected. Version is per Editor install. |
