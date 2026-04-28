---
title: Environment Setup — Unity 6000.5 + Entities 6.5.0
updated: 2026-04-28
folder: Getting Started
---

# Environment Setup — Unity 6000.5 + Entities 6.5.0
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

From **Unity 6000.4+**, Entities is a **Core Package** — it ships inside the Editor and does not need to be added through the Package Manager. This guide walks through installing the Editor, creating a DOTS-ready project, and confirming that Entities is active.

> **Project Environment reference:** Unity 6000.5.0b2 · Entities 6.5.0 · Collections / Entities Graphics / Unity Physics (Core Packages) · Mathematics 1.4.x.

---

## 2. Install Unity 6000.5+

1. Open **Unity Hub**. Install `Unity Hub` first if you don't have it (<https://unity.com/download>).
2. In Hub, go to **Installs → Install Editor**.
3. Pick any **6000.5.x** version (b-series beta or newer). Entities 6.5 is only available on 6000.5+.
4. Modules: leave the defaults. No extra module is required for ECS itself.

> You do **not** need to select any "DOTS" module — Entities is built into the Editor starting from 6000.4.

---

## 3. Create a new project

From **Hub → Projects → New project**:

| Field | Value |
|-------|-------|
| Editor Version | 6000.5.x |
| Template | **Universal 3D** (or **3D (Built-in)**, **HDRP** — any works) |
| Project Name | anything |

Click **Create project**. The Editor opens on an empty scene.

> The old dedicated "DOTS template" is no longer required on 6000.4+. Any template can host an Entities workflow.

---

## 4. Verify Core Packages are active

Open **Window → Package Manager** and switch the scope to **In Project**. You should see packages marked **"Built-in"** for:

- **Entities**
- **Collections**
- **Entities Graphics**
- **Unity Physics** (if your Editor/package set includes the 6.5 DOTS physics package)

`Unity.Mathematics` is still used throughout DOTS code examples, but its 6000.5 package-set entry is `com.unity.mathematics` 1.4.x rather than a 6.x Core Package changelog entry.

If you prefer to verify via files, check `Packages/manifest.json`: there is **no** `com.unity.entities` entry because it ships with the Editor.

Next, open the ECS windows:

- **Window → Entities → Hierarchy** — shows entities at runtime.
- **Window → Entities → Systems** — shows registered systems.
- **Window → Entities → Archetypes** — shows chunk layout.

If all three menus exist, Entities is wired up correctly.

---

## 5. Smoke test — create a SubScene

1. In the Hierarchy, right-click → **New Sub Scene → Empty Scene…**
2. Name it `MainSubScene.unity` and save.
3. Inside the new SubScene, add any primitive (**3D Object → Cube**).
4. Save the scene. The Cube is **baked** into an entity as soon as the SubScene is closed (yellow icon) or at build time.
5. Press **Play**. Open **Window → Entities → Hierarchy** — you should see a `Cube` entity. The smoke test passes.

> A SubScene is the authoring entry point: its GameObjects are baked to Entities on save and at build. The remaining tutorial in [`03_Hello DOTS — First Entity.md`](03_Hello DOTS — First Entity.md) builds on this.

---

## 6. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `Window → Entities` menu is missing | Editor is older than 6000.4 — Entities is not yet a Core Package. Upgrade the Editor. |
| `Baker<T>` / `IComponentData` not found | Assembly-definition file excludes `Unity.Entities`. Add it as a reference, or delete the asmdef for a quick test. |
| SubScene stays as a GameObject and never bakes | The SubScene must be **closed** (the yellow icon) for baked entities to appear. Open/close via the checkbox next to its name in the Hierarchy. |
| Entities Hierarchy window shows nothing in Play mode | Check that the SubScene is included in the current scene and that **Window → Entities → Hierarchy**'s "World" dropdown is set to `DefaultGameObjectInjectionWorld`. |
| Build error: `com.unity.entities not found` | Your `manifest.json` still has a leftover `com.unity.entities` entry from a 1.x project — delete it. See [`Migration/02_Package Manager → Core Package.md`](../Migration/02_Package Manager → Core Package.md). |
