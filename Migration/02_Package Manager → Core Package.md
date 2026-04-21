# Package Manager → Core Package
### Unity 6000.5 · Entities 6.5.0

---

## 1. What changes

On Unity 6000.4+, Entities (and its companion DOTS packages) stop being installed via the Package Manager. They ship inside the Editor as **Core Packages**. Your project files change in three places:

- **`Packages/manifest.json`** — the `com.unity.entities*` entries are removed.
- **`Packages/packages-lock.json`** — the resolved graph shrinks; affected packages appear with `"source": "builtin"`.
- **Package Manager window** — the packages now appear under "In Project" labelled **Built-in**.

Your C# code does **not** change. Assembly names (`Unity.Entities`, `Unity.Collections`, etc.) are the same; `.asmdef` references are unchanged.

---

## 2. Which packages move

Moved to **Core Package** status on 6000.4+:

| UPM id (before) | After |
|-----------------|-------|
| `com.unity.entities` | Core Package, ships with Editor |
| `com.unity.collections` | Core Package |
| `com.unity.mathematics` | Core Package |
| `com.unity.entities.graphics` | Core Package |
| `com.unity.netcode` (Netcode for Entities) | Core Package in 6.5 |

Stay **on Package Manager**:

- `com.unity.physics` (Unity Physics)
- `com.unity.charactercontroller` (Character Controller)

Built into the engine (unchanged):

- Jobs system
- Burst (`com.unity.burst` is still exposed as a UPM entry in some setups, but on 6000.4+ it's tied to the Editor version; safest to leave manifest as-is).

---

## 3. Editing `manifest.json`

Open `Packages/manifest.json`. It looks something like:

```json
{
  "dependencies": {
    "com.unity.entities": "1.4.5",
    "com.unity.collections": "2.4.5",
    "com.unity.mathematics": "1.3.2",
    "com.unity.entities.graphics": "1.4.5",
    "com.unity.netcode": "1.12.0",
    "com.unity.physics": "1.3.14",
    "com.unity.render-pipelines.universal": "17.0.4",
    ...
  }
}
```

Remove the five lines that are now Core Packages:

```diff
 {
   "dependencies": {
-    "com.unity.entities": "1.4.5",
-    "com.unity.collections": "2.4.5",
-    "com.unity.mathematics": "1.3.2",
-    "com.unity.entities.graphics": "1.4.5",
-    "com.unity.netcode": "1.12.0",
     "com.unity.physics": "1.3.14",
     "com.unity.render-pipelines.universal": "17.0.4",
     ...
   }
 }
```

Leave `com.unity.physics` and anything else that is still UPM.

Save the file.

---

## 4. Rebuilding `packages-lock.json`

Two options:

### 4.1 Let the Editor regenerate it

Close the Editor, delete `Packages/packages-lock.json`, reopen. Unity regenerates it. This is the cleanest option.

### 4.2 Edit in place

Open `packages-lock.json` and remove entries matching the five packages above. The Editor reconciles on next open. Slightly faster if you have a complex lock file, but more error-prone.

Both result in the final lock file containing entries like:

```json
"com.unity.entities": {
  "version": "6.5.0",
  "depth": 0,
  "source": "builtin",
  "dependencies": {}
}
```

Note the `"source": "builtin"` — that's the tell that the package is now a Core Package.

---

## 5. Verifying the result

1. **Package Manager window** (`Window → Package Manager` → "In Project") — Entities, Collections, Mathematics, Entities Graphics (and Netcode for Entities if you use it) should appear tagged **Built-in**.
2. **Editor menus** — `Window → Entities → Hierarchy/Systems/Archetypes` exist and work.
3. **Compile** — press Play; the project should enter Play mode with no compile errors referencing missing assemblies.

---

## 6. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `Unable to find package 'com.unity.entities@1.x.y'` after editing manifest | `packages-lock.json` still references the old version. Delete it and let the Editor regenerate. |
| `The type or namespace Unity.Entities could not be found` | You're on an Editor older than 6000.4 — Core Package isn't available. Upgrade Editor, or add the UPM package back. |
| Package Manager shows both Built-in and UPM Entities | Manifest still has `com.unity.entities` entry. Remove it. |
| Third-party asset won't compile because it pins `com.unity.entities@1.x` | The asset needs updating. Either wait, fork, or stay on the 1.x line for this project. |
| Unity Physics breaks because it wanted a specific Entities version | Upgrade Unity Physics to a version compatible with Entities 6.5. Check the package's changelog on docs.unity3d.com. |
| `packages-lock.json` regeneration fails | Network issue or a pinned private registry. Check Editor logs for the specific resolution failure. |
| Netcode for Entities still appears as UPM | On 6000.5+, Netcode 6.5 is Built-in. If `com.unity.netcode` is still in manifest, remove it and regenerate. |
