# Glossary
### Unity 6000.5 · Entities 6.5.0

One-line definitions for every non-obvious term used in this manual. Cross-links point to the doc where each term is defined in depth.

---

## 1. Core nouns — the five things ECS is built from

| Term | Definition | See |
|---|---|---|
| **Entity** | An opaque 64-bit ID (`struct Entity`, `Unity.Entities` namespace). No data, no behaviour — just a key used to look up components. | [`DOTS Workflows/03_ECS Core Concepts.md`](DOTS Workflows/03_ECS Core Concepts.md) |
| **Component** | Data attached to an entity. Several kinds — see §2. | [`DOTS Workflows/05_Component Types.md`](DOTS Workflows/05_Component Types.md) |
| **Archetype** | The set of component types an entity has. Entities with the same archetype share storage. | [`DOTS Workflows/03_ECS Core Concepts.md`](DOTS Workflows/03_ECS Core Concepts.md) |
| **Chunk** | A 16 KB block of entities sharing an archetype. Components stored column-major (SoA) within the chunk. | [`Optimizations and Debugging/01_Chunk Layout & TypeManager.md`](Optimizations and Debugging/01_Chunk Layout & TypeManager.md) |
| **World** | Container of entities and systems. The runtime starts with a "Default World". | [`DOTS Workflows/03_ECS Core Concepts.md`](DOTS Workflows/03_ECS Core Concepts.md) |

---

## 2. Component kinds

| Kind | What it is |
|---|---|
| `IComponentData` | Unmanaged struct component — the default. |
| `IBufferElementData` | Element type for a `DynamicBuffer<T>` attached to an entity. |
| `ISharedComponentData` | Value-keyed component — entities with the same value share a chunk bucket. |
| `ICleanupComponentData` | Component that survives `DestroyEntity` so a system can clean up before the entity fully disappears. |
| Tag component | Empty `IComponentData` struct — zero bytes, used purely as a query filter. |
| Chunk component | Per-chunk data (one instance per chunk, not per entity). |
| `IEnableableComponent` | Component whose presence toggles via a per-chunk bitmask — **no structural change**. |

Details: [`DOTS Workflows/05_Component Types.md`](DOTS Workflows/05_Component Types.md) · [`DOTS Workflows/06_Enableable Component.md`](DOTS Workflows/06_Enableable Component.md).

---

## 3. Systems and jobs

| Term | What it is |
|---|---|
| `ISystem` | Modern struct-based system interface. Burst-compatible. **Preferred for new systems.** |
| `SystemBase` | Legacy class-based system interface. Use only when a system needs main-thread managed access. |
| SystemGroup | Ordered container of systems. Main groups: `InitializationSystemGroup`, `SimulationSystemGroup`, `PresentationSystemGroup`. |
| `IJobEntity` | Job that iterates entities matching a query. Source-generated, Burst-compiled. |
| `IJobChunk` | Job that iterates chunks (not individual entities) — use for per-chunk aggregation. |
| `SystemAPI` | Static helper surface inside systems: `Query<>()`, `GetComponentLookup<T>()`, `Time`, `GetSingleton<T>()`, etc. |

Details: [`DOTS Workflows/08_System — ISystem vs SystemBase.md`](DOTS Workflows/08_System — ISystem vs SystemBase.md) · [`DOTS Workflows/11_IJobEntity · SystemAPI.Query.md`](DOTS Workflows/11_IJobEntity · SystemAPI.Query.md) · [`DOTS Workflows/12_IJobEntity vs IJobChunk.md`](DOTS Workflows/12_IJobEntity vs IJobChunk.md).

---

## 4. Authoring → runtime

| Term | What it is |
|---|---|
| Baker | Converts an authoring GameObject + MonoBehaviour into ECS components at SubScene save/build time. |
| SubScene | Authoring container — GameObjects inside it are baked into entities and serialised to an EntityScene asset. |
| EntityScene | The baked asset produced from a SubScene — loaded into a World at runtime. |

Details: [`DOTS Workflows/01_Baker Pattern & SubScene.md`](DOTS Workflows/01_Baker Pattern & SubScene.md).

---

## 5. Structural change safety

| Term | What it is |
|---|---|
| Structural change | Adds/removes a component or creates/destroys an entity. Moves entities between chunks → expensive; not safe mid-iteration. |
| `EntityCommandBuffer` (ECB) | Records structural changes during iteration, plays them back at a safe boundary. |
| `EntityCommandBuffer.ParallelWriter` | Thread-safe ECB variant for parallel jobs; each call takes a `sortKey` for deterministic playback. |
| Deferred entity | An entity created via `ecb.CreateEntity()` / `Instantiate()` that doesn't exist yet — placeholder index resolves at ECB playback. |
| Sort key | Integer passed to each parallel ECB command; `[ChunkIndexInQuery]` is the default. |

Details: [`DOTS Workflows/13_Structural Change & Safety.md`](DOTS Workflows/13_Structural Change & Safety.md) · [`DOTS Workflows/14_EntityCommandBuffer · Deferred Entity.md`](DOTS Workflows/14_EntityCommandBuffer · Deferred Entity.md) · [`DOTS Workflows/15_ParallelWriter · Deterministic Playback.md`](DOTS Workflows/15_ParallelWriter · Deterministic Playback.md).

---

## 6. Identity types — the three-way distinction

These are **three different things** that are easy to confuse.

| Type | What it is |
|---|---|
| `Entity` | ECS handle (`Unity.Entities`). Just an ID in the ECS world. |
| `EntityId` | Engine-level replacement for `UnityEngine.Object.GetInstanceID()`, introduced in Unity 6000.3. **Not** the ECS `Entity`. |
| `UnityObjectRef<T>` | ECS-side reference to a `UnityEngine.Object` — lets entities point at managed assets (materials, meshes) without boxing. |

Details: [`DOTS Workflows/04_Identity Types — Entity · EntityId · UnityObjectRef.md`](DOTS Workflows/04_Identity Types — Entity · EntityId · UnityObjectRef.md).

---

## 7. Legacy / obsolete (do NOT appear in new code)

| Term | Status on 6.5 | Migration target |
|---|---|---|
| `Entities.ForEach` | Obsolete since 1.4; still compiles with warnings; removal planned for Entities 2.0. | `SystemAPI.Query<>()` or `IJobEntity` — [`Migration/04_foreach → IJobEntity.md`](Migration/04_foreach → IJobEntity.md) |
| `IAspect` | Obsolete since 1.4; still compiles with warnings; removal planned for Entities 2.0. | Direct component parameters (`RefRW<T>` / `RefRO<T>`) — [`Migration/05_IAspect Removal.md`](Migration/05_IAspect Removal.md) |
| `InstanceID` (engine-level id) | Obsolete on Unity 6000.5; compile errors expected on 6000.6. | `EntityId` — [`Migration/03_InstanceID → EntityId.md`](Migration/03_InstanceID → EntityId.md) |
| `ComponentLookup.GetRefRWOptional` | Obsolete. | `TryGetRefRW` — see [`Migration/01_Entities 1.x → 6.5 Overview.md`](Migration/01_Entities 1.x → 6.5 Overview.md) |

---

## 8. Supporting concepts

| Term | What it is |
|---|---|
| Burst | Unity's native compiler for jobs and `[BurstCompile]`-marked structs. Produces native SIMD code. |
| Change version | 32-bit counter bumped when a system writes to a component column in a chunk. Enables `SetChangedVersionFilter` to skip unchanged chunks. |
| SoA (Structure of Arrays) | Storage layout where each field becomes its own array — cache-friendly for scalar iteration. Chunks use SoA. |
| `RefRW<T>` / `RefRO<T>` | Wrapper types for accessing components in `SystemAPI.Query` with read-write / read-only intent. |
| `EnabledRefRW<T>` / `EnabledRefRO<T>` | Wrapper for reading/writing an enableable component's toggle bit without triggering a structural change. |
| TypeManager | The runtime registry that assigns each component a stable type index. Rebuilt at startup; large projects feel this. |
| `BlobAssetReference<T>` | Pointer to a large, read-only data blob stored outside the chunk. Avoids bloating chunk capacity. |
