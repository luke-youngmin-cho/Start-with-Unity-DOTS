---
title: Core Packages Explained
updated: 2026-04-27
folder: Getting Started
---

# Core Packages Explained
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

With the **Entities package 6.4.0** release (shipping alongside Unity Editor 6000.4), the major DOTS packages transitioned from the Package Manager into the Editor itself as **Core Packages**. The exact wording from the package changelog is:

> *"Package is now a core package embedded in Unity."*

This affected `com.unity.entities`, `com.unity.collections`, `com.unity.mathematics`, `com.unity.entities.graphics`, and (on its matching 6.5 release) Netcode for Entities. It changes how you install, version, and track them — but it is a **distribution** change, not an API change.

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
| **Mathematics** | **Core Package** | Burst-optimized `float3`, `quaternion`, `int4`, etc. |
| **Entities Graphics** | **Core Package** | Renders entity-based graphics through the SRP |
| **Netcode for Entities** | **Core Package** (6.5) | Client-server networking for entities |
| **Unity Physics** | Package Manager | Stateless, deterministic physics for entities |
| **Character Controller** | Package Manager | ECS-based character controller |
| **Burst Compiler** | Built-in engine feature | Compiles jobs and systems to tight native code |
| **Job System** | Built-in engine feature | Multithreaded job scheduling |

> Note: Job System and Burst were never UPM packages — they are part of the Engine. The shift in 6000.4 is specifically about the *DOTS packages on top of them*.

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

No `com.unity.entities`, `com.unity.collections`, `com.unity.mathematics`, or `com.unity.entities.graphics` entries are needed. If you upgrade a 1.x project, you remove them — see [`Migration/02_Package Manager → Core Package.md`](../Migration/02_Package Manager → Core Package.md).

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
