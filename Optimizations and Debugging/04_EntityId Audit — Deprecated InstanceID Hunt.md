---
title: EntityId Audit — Deprecated InstanceID Hunt
updated: 2026-04-21
folder: Optimizations and Debugging
---

# EntityId Audit — Deprecated InstanceID Hunt
### Unity 6000.5 · Entities 6.5.0

---

## 1. Why this page exists

Unity 6000.3 introduced `EntityId` and began marking `InstanceID` APIs obsolete. Unity 6000.5 is where the deprecation hits code that previously compiled clean — implicit `int ↔ EntityId` conversion is obsolete, `FindObjectsByType(FindObjectsSortMode.InstanceID)` is deprecated, and `EntityId` is scheduled to grow to 8 bytes. Unity 6000.6α turns the remaining warnings into **compile errors**.

So the practical question on 6000.5 is not "does my code still run" — it's "where are the deprecation warnings, and which ones will break when I bump to 6000.6". This page is the checklist for finding those.

Background on the type itself is in [`DOTS Workflows/04_Identity Types — Entity · EntityId · UnityObjectRef.md`](../DOTS Workflows/04_Identity Types — Entity · EntityId · UnityObjectRef.md). The step-by-step migration is in [`Migration/03_InstanceID → EntityId.md`](../Migration/03_InstanceID → EntityId.md). Use this page as the "find all the broken places" companion.

---

## 2. What to grep for

Run these searches across your `Assets/` and `Packages/` folders:

| Pattern | Why it's suspicious |
|---------|---------------------|
| `\.GetInstanceID\(` | Deprecated on 6000.3+; every call site is a candidate for `GetEntityId()`. |
| `InstanceIDToObject` | Editor-only lookup; replaced by `EntityIdToObject`. **Compile error on 6000.6α**. |
| `(int)` near an InstanceID/EntityId | Explicit cast — obsolete on 6000.5, removed on 6000.6. |
| `int.Parse` near "id" / "instanceId" | String round-trip of an integer InstanceID. |
| `\.ToString\(\)` on an InstanceID | Serialization-as-string pattern. |
| `.OrderBy(` / `.Sort(` on InstanceID | Sorting to imply creation order — never safe with `EntityId`. |
| `FindObjectsSortMode.InstanceID` | Deprecated on 6000.5. |
| `FindObjectsByType<T>(.., FindObjectsSortMode.InstanceID)` | Same — switch to `FindObjectsSortMode.None` or your own sort. |
| `< 0` / `> 0` / `& 0x` near InstanceID | Sign or bit tricks — no contract on `EntityId` layout. |
| `Dictionary<int, ...>` where the int is an InstanceID | Keyed container — re-key to `EntityId`. |
| `objectInstanceId`, `parentId`, `instanceIDsCount` in Profiler scripts | Replaced with `EntityId`-typed Profiler APIs on 6000.5. |

Recommended: keep a running "InstanceID audit" scratch file. Paste each hit, note what replacement you chose. It makes the PR review straightforward.

---

## 3. Patterns and their safe replacements

### 3.1 Casting to/from `int`

```csharp
// BEFORE
int id = obj.GetInstanceID();
Dictionary<int, MyData> byId;
byId[id] = data;

// AFTER
EntityId id = obj.GetEntityId();
Dictionary<EntityId, MyData> byId;
byId[id] = data;
```

If the dictionary is shared with code that still uses `int`, box via a thin `EntityId → int` lookup table kept in one place; don't scatter adapters through the codebase.

### 3.2 `InstanceIDToObject`

```csharp
// BEFORE (Editor)
var obj = EditorUtility.InstanceIDToObject(id);

// AFTER (Editor, Unity 6000.3+)
var obj = EditorUtility.EntityIdToObject(id);
```

For runtime lookups (non-Editor), there is no equivalent — you must keep a strong reference via `UnityObjectRef<T>` or an asset-loading key.

### 3.3 String round-trip

```csharp
// BEFORE — save file
string line = $"prefab:{obj.GetInstanceID()}";

// AFTER
// InstanceID and EntityId aren't stable across sessions.
// Save by asset path / GUID instead:
#if UNITY_EDITOR
string guid = UnityEditor.AssetDatabase.AssetPathToGUID(path);
#endif
```

Anything you were storing to disk or sending over the wire as an InstanceID `int` was always fragile (not stable across sessions). `EntityId` just makes the invariant impossible to ignore: serialize by GUID, path, or an ID your game owns.

### 3.4 `GetHashCode` as identity

```csharp
// BEFORE
int id = obj.GetHashCode();  // used as a stand-in for InstanceID

// AFTER
EntityId id = obj.GetEntityId();
```

Never rely on `GetHashCode` for identity semantics — hash codes are not guaranteed unique. This pattern was always a bug; the migration is a good excuse to fix it.

### 3.5 Sign / bit tricks

```csharp
// BEFORE
if (obj.GetInstanceID() < 0) { /* asset */ }
if ((id & 0x80000000) != 0) { /* ??? */ }
```

There is no drop-in replacement because there is no bit structure to exploit. Replace with:

- **"Is it an asset?"** → `AssetDatabase.Contains(obj)` (Editor), or track it yourself at load time.
- **Packed state in high bits** → move that state to a separate field.

### 3.6 Sorting by InstanceID

```csharp
// BEFORE
items.Sort((a, b) => a.GetInstanceID().CompareTo(b.GetInstanceID()));

// AFTER — sort by something with semantic meaning
items.Sort((a, b) => a.CreationTime.CompareTo(b.CreationTime));
items.Sort((a, b) => string.Compare(a.name, b.name, StringComparison.Ordinal));
```

The prior sort was using creation-order as a proxy for InstanceID. Make the intent explicit.

---

## 4. Runtime audit — the defensive check

Add a temporary Editor-only assertion during migration:

```csharp
#if UNITY_EDITOR
public static class EntityIdAudit
{
    [UnityEditor.Callbacks.DidReloadScripts]
    static void WarnOnInstanceIDUse()
    {
        // Not a complete static check, but if your code keeps a registry
        // of int IDs, loop through it here and Debug.LogWarning anything
        // that looks like an InstanceID used as a map key.
    }
}
#endif
```

It catches only what you already track — the real work is the grep pass above. But it's a cheap guard while you're migrating.

---

## 5. Dependencies you can't change

Third-party assets (Asset Store packages, plugins) may use `InstanceID` internally. Strategies:

- **Wrap** — build a small adapter layer in your own code that converts to/from `EntityId`. Keep it in one file.
- **Wait** — the deprecation is not immediate removal. If the package is actively maintained it will be updated.
- **Fork** — as a last resort, fork the package and apply the same patterns from §3.

Document each dependency you couldn't update and revisit at each Unity upgrade.

---

## 6. Validation

Once you believe you're done:

1. **Grep again.** Zero hits on the patterns in §2 means the surface you searched is clean.
2. **Compile with warnings as errors.** Any remaining deprecated `InstanceID` API calls will surface as warnings you can treat as errors.
3. **Exercise the save/load path.** If you previously serialized InstanceIDs by accident, old save files will fail to load — catch it now, not in shipped builds.
4. **Run the game on a fresh project instance.** InstanceID values weren't stable across sessions anyway; confirm your code no longer assumes they were.

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Code compiles but warns `InstanceID is deprecated` | Replace the call site with `GetEntityId()` and re-key whatever stored it. |
| `InvalidCastException: cannot cast EntityId to Int32` | An old `(int)` cast slipped through. Remove it; change the downstream type. |
| Save files fail to load after migration | The save stored an InstanceID as an int. Migrate saves on load (best-effort: fall back to a lookup by name/path, accept some data loss). |
| Tests using fixture InstanceIDs break | The fixtures relied on cast-to-int or sign checks. Replace the test fixture with a deterministic key you own. |
| Third-party package crashes calling `InstanceIDToObject` | Package hasn't migrated. Wrap its boundary or pin an older Unity version. |
