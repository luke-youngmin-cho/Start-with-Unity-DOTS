# InstanceID → EntityId Migration
### Unity 6000.5 · Entities 6.5.0

---

## 1. What changes, and when

`EntityId` is a Unity Editor–level replacement for `UnityEngine.Object.GetInstanceID()`. It is **not** the ECS `Entity` struct — see [`../DOTS Workflows/04_Identity Types — Entity · EntityId · UnityObjectRef.md`](../DOTS%20Workflows/04_Identity%20Types%20%E2%80%94%20Entity%20%C2%B7%20EntityId%20%C2%B7%20UnityObjectRef.md) for the three-way distinction (`Entity` / `EntityId` / `UnityObjectRef<T>`).

Timeline on the Editor track:

| Unity Editor | Status |
|--------------|--------|
| 6000.3 | `EntityId` type introduced; legacy `InstanceID` APIs get obsolete marks |
| 6000.4 | Obsolete range extended |
| 6000.5 | Implicit `int ↔ EntityId` conversion obsolete; `FindObjectsByType(FindObjectsSortMode.InstanceID)` deprecated; Profiler fields replaced; `EntityId` announced to grow 4 → 8 bytes |
| 6000.6α | Implicit cast **removed**; `InstanceIDToObject(int)` is a **compile error** |

On Unity 6000.5 (the target of this manual) you will see deprecation warnings. Code continues to compile. The migration is about clearing those warnings before bumping to 6000.6, where several of them become hard compile errors.

Key things you can no longer rely on:

- `EntityId` cast to or from `int` (obsolete on 6000.5; removed on 6000.6).
- Sign-based `InstanceID` tricks (no contract on `EntityId` layout; also widening to 8 bytes).
- `EntityId.ToString()` round-trip via `int.Parse`.
- Sorting to imply creation order — `EntityId` is versioned and reusable.
- Bit packing.

This page is the step-by-step refactor. The "find everything to fix" companion is [`../Optimizations and Debugging/04_EntityId Audit — Deprecated InstanceID Hunt.md`](../Optimizations%20and%20Debugging/04_EntityId%20Audit%20%E2%80%94%20Deprecated%20InstanceID%20Hunt.md).

---

## 2. API mapping

| Before | After |
|--------|-------|
| `obj.GetInstanceID()` (returns `int`) | `obj.GetEntityId()` (returns `EntityId`) |
| `UnityEditor.EditorUtility.InstanceIDToObject(int)` | `UnityEditor.EditorUtility.EntityIdToObject(EntityId)` |
| `Dictionary<int, T>` keyed on InstanceID | `Dictionary<EntityId, T>` |
| `int idField` stored for later lookup | `EntityId idField` stored for later lookup |
| `(int)instanceId` cast | Not available — redesign |
| `instanceId.ToString()` → `int.Parse(...)` | Not available — use GUID or an ID you own |

---

## 3. Step-by-step refactor

### 3.1 Replace the call site

```csharp
// BEFORE
int id = target.GetInstanceID();

// AFTER
EntityId id = target.GetEntityId();
```

If the receiving variable was `int`, change its type.

### 3.2 Re-key any dictionaries

```csharp
// BEFORE
private readonly Dictionary<int, AudioSource> _sources = new();

public void Register(AudioSource s) => _sources[s.GetInstanceID()] = s;

// AFTER
private readonly Dictionary<EntityId, AudioSource> _sources = new();

public void Register(AudioSource s) => _sources[s.GetEntityId()] = s;
```

### 3.3 Editor-side InstanceIDToObject

```csharp
#if UNITY_EDITOR
// BEFORE
var obj = UnityEditor.EditorUtility.InstanceIDToObject(id);

// AFTER
var obj = UnityEditor.EditorUtility.EntityIdToObject(id);
#endif
```

Behaviour is the same — the parameter type changed.

### 3.4 Serialized / persisted IDs — the hard case

If you previously wrote InstanceID values into:

- Save files
- Editor custom data (asset metadata)
- Analytics events
- Network packets

…then you already had a bug: InstanceIDs were never stable across sessions. `EntityId` just makes the invariant visible. Replacement strategy:

```csharp
// Save by stable identity
#if UNITY_EDITOR
string guid = UnityEditor.AssetDatabase.AssetPathToGUID(
    UnityEditor.AssetDatabase.GetAssetPath(obj));
#endif
```

For runtime objects that aren't assets, introduce your own ID scheme:

```csharp
public class StableId : MonoBehaviour
{
    [SerializeField] private SerializableGuid _id;
    public SerializableGuid Id => _id;

    void Reset() => _id = SerializableGuid.NewGuid();
}
```

### 3.5 Sign / bit tricks

```csharp
// BEFORE — abused the sign bit to flag "is asset"
bool isAsset = obj.GetInstanceID() < 0;

// AFTER — ask the question directly
#if UNITY_EDITOR
bool isAsset = UnityEditor.AssetDatabase.Contains(obj);
#else
bool isAsset = /* track this at load time in a set */;
#endif
```

### 3.6 Sorting

```csharp
// BEFORE — sort by creation order via InstanceID
items.Sort((a, b) => a.GetInstanceID().CompareTo(b.GetInstanceID()));

// AFTER — sort by a field with semantic meaning
items.Sort((a, b) => a.SpawnTick.CompareTo(b.SpawnTick));
```

Sorting by `EntityId` is allowed as long as you only need *some* total order; just don't assume it matches creation order.

---

## 4. Code inside ECS components

Components cannot hold managed references, so you don't normally see `Object` references inside `IComponentData`. If you were storing an `int` InstanceID as a component field to remember a `UnityEngine.Object`, switch to `UnityObjectRef<T>`:

```csharp
// BEFORE
public struct VFXAuthor : IComponentData
{
    public int ParticleSystemInstanceID;
}

// AFTER
public struct VFXRef : IComponentData
{
    public UnityObjectRef<ParticleSystem> ParticleSystem;
}
```

And bake it:

```csharp
AddComponent(entity, new VFXRef
{
    ParticleSystem = new UnityObjectRef<ParticleSystem>(authoring.ParticleSystem)
});
```

See [`../DOTS Workflows/04_Identity Types — Entity · EntityId · UnityObjectRef.md`](../DOTS%20Workflows/04_Identity%20Types%20%E2%80%94%20Entity%20%C2%B7%20EntityId%20%C2%B7%20UnityObjectRef.md) for the full `UnityObjectRef<T>` surface.

---

## 5. Verifying the refactor

1. **Grep for leftover calls** — the patterns in [`../Optimizations and Debugging/04_EntityId Audit — Deprecated InstanceID Hunt.md`](../Optimizations%20and%20Debugging/04_EntityId%20Audit%20%E2%80%94%20Deprecated%20InstanceID%20Hunt.md).
2. **Treat warnings as errors** — any remaining `GetInstanceID()` calls show up as deprecation warnings.
3. **Exercise save/load** — if you migrated persisted IDs, verify old saves either still work or fail loudly.
4. **Run unit tests** that previously keyed on `int` IDs. If they were relying on a stable numeric value, rewrite them around `EntityId` or around your new stable-id scheme.

---

## 6. Third-party code

Not all packages are migrated yet. Two strategies:

- **Wrap at the boundary.** Keep a small adapter file that converts between `int` (what the old package wants) and `EntityId` (what your code uses). The adapter will `GetInstanceID()` for the conversion, which still compiles.
- **Fork or replace the package.** For packages you depend on deeply, forking may be faster than waiting.

Track wrapped boundaries in a TODO — they're the pressure point to revisit each Unity upgrade.

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `error CS0029: Cannot implicitly convert type 'EntityId' to 'int'` | Old `int id = obj.GetInstanceID();` became `EntityId id = obj.GetEntityId();`. Change the variable type. |
| `Dictionary<int, ...>` doesn't compile when passing `EntityId` | Re-key to `Dictionary<EntityId, ...>`. |
| Save file loads produce the wrong objects | Old save stored non-stable InstanceIDs. Migrate on load with a best-effort fallback (name/path lookup) or wipe and regenerate. |
| Tests break because they pinned specific InstanceID values | The test was always fragile; regenerate expected values from the new stable-id scheme. |
| `EntityIdToObject` missing | You're editing a build script outside `#if UNITY_EDITOR`. The API is Editor-only. |
| A third-party editor tool fails after migration | It still calls `InstanceIDToObject(int)` internally. Wait for an update, or wrap the call. |
