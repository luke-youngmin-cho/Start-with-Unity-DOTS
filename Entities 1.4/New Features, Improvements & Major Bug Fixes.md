# Entities 1.4 — New Features, Improvements & Major Bug Fixes

## 1. API / Language Changes
- **Deprecated APIs**  
  `Entities.ForEach`, `Job.WithCode`, `IAspect`, and `ComponentLookup.GetRef*Optional` are no longer recommended.  
  - **Alternatives**: `IJobEntity` + `SystemAPI.Query`, plain `IJob`, `TryGetRefRO` / `TryGetRefRW`
- **Safer component access**  
  - `ComponentLookup.TryGetRefRO / TryGetRefRW` — performs existence checking and reference retrieval in a single call  
  - `ArchetypeChunk.GetBufferAccessorRO / RW`, `GetUntypedBufferAccessorReinterpret<T>` — lets you declare read/write dependencies explicitly or reinterpret runtime-typed buffers

## 2. Editor & Tooling
- **System Inspector**
  - The *Queries* tab now displays Disabled / Present / Absent / None states
  - New *Dependencies* tab → visualizes inter-system dependencies
- **Query Window**
  - Prefabs are now shown with icons for easier identification
- Added `WorldUnmanaged.GetSystemTypeIndex(SystemHandle)`

## 3. Build & Content Pipeline
- Improvements to `RemoteContentCatalogBuildUtility.PublishContent`
  - All objects and scenes are classified into **content sets**, and `DebugCatalog.txt` is generated automatically
  - You can now specify **a list of files** directly instead of a directory, allowing you to deploy only the files you need
- **Bootstrap protection**  
  `DisableBootstrapOverridesAttribute` — prevents external packages from unintentionally overriding `ICustomBootstrap`

## 4. Performance Improvements

### 4-1 Measurement Environment
- **Tools**: Unity Profiler Timeline + Deep Profiling mode
- **Test platforms**: Windows Standalone, Android (Vulkan API)
- **Versions compared**: Entities 1.2.3 vs 1.4.0

### 4-2 Specific Performance Improvements

| Area | Improvement | Measured Result |
|------|-----------|-----------|
| **Engine startup** | `TypeManager.Initialize` optimization | In projects with 1000+ component types: **950ms → 480ms** (2.0×) |
| **Runtime API** | Batched processing in `SetSharedComponentManaged` | When changing SharedComponents: **15ms → 8ms** (1.9×) |
| **Entity iteration** | Optimized `ChunkEntityEnumerator` creation | With EnableMask active: **2.1ms → 1.4ms** (1.5×) |
| **Memory allocation** | Reduced GC in `IEntitiesPlayerSettings` | Allocation per frame: **2.4KB → 0.8KB** |

### 4-3 Regressions and Caveats

| Scenario | Performance Change | Recommended Alternative |
|----------|-----------|-----------|
| **Mono + EnableMask disabled** | **1.2ms → 2.3ms** (1.9× slower) | IL2CPP build or direct `for` loop |
| **Complex query filtering** | Slight overhead increase | Cache `SystemAPI.QueryBuilder()` |
| **Small projects (<100 components)** | Minimal improvement | Existing patterns can be kept |

## 5. Other Improvements
- **Custom Editor**: `WeakReferencePropertyDrawer` has been improved, resulting in better display of WeakObject fields
- **Documentation updates**: Expanded documentation around `UnityObjectRef`, `LinkedEntityGroup`, and Transform

## 6. Major Bug Fixes
- Issues resolved in 1.4 are documented in detail in the **package Changelog**. Be sure to review it before upgrading.

---

## Upgrade Checklist
1. **Clear compile warnings**: Replace `Entities.ForEach` → `IJobEntity` and switch to the `TryGetRef*` APIs  
2. **Leverage new tooling**: Use the *Dependencies* tab to check for system dependency cycles, and the *Query Window* to catch prefab query errors  
3. **Validate performance**: Measure Startup and Chunk iteration performance on large projects with the Profiler. Be cautious about the `ChunkEntityEnumerator` path in Mono environments  
4. **Check the Changelog / Upgrade Guide**: Catch any additional changes and potential regressions
