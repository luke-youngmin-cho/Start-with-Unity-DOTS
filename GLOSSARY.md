# Glossary
### Unity 6000.5 · Entities 6.5.0 · Netcode for Entities 6.5.0

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

## 6. Entity reference types

These are the DOTS reference types used in ECS data and baking workflows.

| Type | What it is |
|---|---|
| `Entity` | ECS handle (`Unity.Entities`). Just an ID in the ECS world. |
| `EntityPrefabReference` | Reference to a baked entity prefab asset for content/streaming workflows (`Unity.Entities.Serialization`). |
| `UnityObjectRef<T>` | ECS-side reference to a `UnityEngine.Object` — lets entities point at managed assets (materials, meshes) without boxing. |

Details: [`DOTS Workflows/04_Entity References — Entity · EntityPrefabReference · UnityObjectRef.md`](DOTS Workflows/04_Entity References — Entity · EntityPrefabReference · UnityObjectRef.md).

---

## 7. Netcode for Entities

| Term | What it is | See |
|---|---|---|
| Netcode for Entities | Unity's DOTS multiplayer netcode layer for server-authoritative games with client prediction. | [`DOTS Workflows/16_Netcode Client-Server World & Bootstrap.md`](DOTS Workflows/16_Netcode Client-Server World & Bootstrap.md) |
| Server World | Authoritative ECS world that runs the server simulation and sends Ghost snapshots. | [`DOTS Workflows/16_Netcode Client-Server World & Bootstrap.md`](DOTS Workflows/16_Netcode Client-Server World & Bootstrap.md) |
| Client World | ECS world that receives snapshots, sends commands, predicts local gameplay, and presents interpolated state. | [`DOTS Workflows/16_Netcode Client-Server World & Bootstrap.md`](DOTS Workflows/16_Netcode Client-Server World & Bootstrap.md) |
| Thin Client World | Lightweight dummy client world used for testing load and connection behavior. | [`DOTS Workflows/16_Netcode Client-Server World & Bootstrap.md`](DOTS Workflows/16_Netcode Client-Server World & Bootstrap.md) |
| `NetworkStreamConnection` | Component on a connection entity that stores the transport connection handle. | [`DOTS Workflows/17_Netcode Network Connection & Approval.md`](DOTS Workflows/17_Netcode Network Connection & Approval.md) |
| `NetworkId` | Server-assigned connection ID after approval. | [`DOTS Workflows/17_Netcode Network Connection & Approval.md`](DOTS Workflows/17_Netcode Network Connection & Approval.md) |
| `NetworkStreamInGame` | Component that enables gameplay data exchange; without it, snapshots and commands do not flow. | [`DOTS Workflows/17_Netcode Network Connection & Approval.md`](DOTS Workflows/17_Netcode Network Connection & Approval.md) |
| Ghost | Server-authoritative networked entity replicated to clients through snapshots. | [`DOTS Workflows/18_Netcode Ghost Snapshot & Synchronization.md`](DOTS Workflows/18_Netcode Ghost Snapshot & Synchronization.md) |
| Snapshot | Serialized Ghost state sent from server to client, usually over unreliable transport. | [`DOTS Workflows/18_Netcode Ghost Snapshot & Synchronization.md`](DOTS Workflows/18_Netcode Ghost Snapshot & Synchronization.md) |
| `GhostField` | Attribute that marks a component field for Ghost serialization. | [`DOTS Workflows/18_Netcode Ghost Snapshot & Synchronization.md`](DOTS Workflows/18_Netcode Ghost Snapshot & Synchronization.md) |
| `PredictedGhost` | Component identifying Ghosts that participate in client prediction. | [`DOTS Workflows/19_Netcode Prediction & Rollback.md`](DOTS Workflows/19_Netcode Prediction & Rollback.md) |
| `Simulate` | Enableable tag used by Netcode to mark which predicted entities should run on the current prediction tick. | [`DOTS Workflows/19_Netcode Prediction & Rollback.md`](DOTS Workflows/19_Netcode Prediction & Rollback.md) |
| `IInputComponentData` | High-level input component type with generated command-buffer management. | [`DOTS Workflows/20_Netcode Command Stream & Input.md`](DOTS Workflows/20_Netcode Command Stream & Input.md) |
| `ICommandData` | Lower-level command stream type where tick mapping and buffer access are managed manually. | [`DOTS Workflows/20_Netcode Command Stream & Input.md`](DOTS Workflows/20_Netcode Command Stream & Input.md) |
| `InputEvent` | One-shot input marker for events such as jump/fire that must register exactly once on the target tick. | [`DOTS Workflows/20_Netcode Command Stream & Input.md`](DOTS Workflows/20_Netcode Command Stream & Input.md) |
| `IRpcCommand` | Reliable one-shot RPC message type. | [`DOTS Workflows/21_Netcode RPC.md`](DOTS Workflows/21_Netcode RPC.md) |
| `IApprovalRpcCommand` | Special RPC type allowed during connection approval. | [`DOTS Workflows/21_Netcode RPC.md`](DOTS Workflows/21_Netcode RPC.md) |
| Ghost Importance | Server-side priority score that decides which Ghosts fit into snapshot budget first. | [`DOTS Workflows/22_Netcode Ghost Optimization · Importance · Relevancy.md`](DOTS Workflows/22_Netcode Ghost Optimization · Importance · Relevancy.md) |
| Ghost Relevancy | Per-connection filtering that decides whether a Ghost should replicate to a specific client. | [`DOTS Workflows/22_Netcode Ghost Optimization · Importance · Relevancy.md`](DOTS Workflows/22_Netcode Ghost Optimization · Importance · Relevancy.md) |
| Lag Compensation | Server-side lookup of historical collision worlds so it can validate client actions at the client's perceived time. | [`DOTS Workflows/23_Netcode Physics Integration & Lag Compensation.md`](DOTS Workflows/23_Netcode Physics Integration & Lag Compensation.md) |

---

## 8. Legacy / obsolete (do NOT appear in new code)

| Term | Status on 6.5 | Migration target |
|---|---|---|
| `Entities.ForEach` | Obsolete since 1.4; still compiles with warnings; removal planned for a future major release. | `SystemAPI.Query<>()` or `IJobEntity` — [`Migration/04_foreach → IJobEntity.md`](Migration/04_foreach → IJobEntity.md) |
| `IAspect` | Obsolete since 1.4; still compiles with warnings; removal planned for a future major release. | Direct component parameters (`RefRW<T>` / `RefRO<T>`) — [`Migration/05_IAspect Removal.md`](Migration/05_IAspect Removal.md) |
| `ComponentLookup.GetRefRWOptional` | Obsolete. | `TryGetRefRW` — see [`Migration/01_Entities 1.x → 6.5 Overview.md`](Migration/01_Entities 1.x → 6.5 Overview.md) |

---

## 9. Supporting concepts

| Term | What it is |
|---|---|
| Burst | Unity's native compiler for jobs and `[BurstCompile]`-marked structs. Produces native SIMD code. |
| Change version | 32-bit counter bumped when a system writes to a component column in a chunk. Enables `SetChangedVersionFilter` to skip unchanged chunks. |
| SoA (Structure of Arrays) | Storage layout where each field becomes its own array — cache-friendly for scalar iteration. Chunks use SoA. |
| `RefRW<T>` / `RefRO<T>` | Wrapper types for accessing components in `SystemAPI.Query` with read-write / read-only intent. |
| `EnabledRefRW<T>` / `EnabledRefRO<T>` | Wrapper for reading/writing an enableable component's toggle bit without triggering a structural change. |
| TypeManager | The runtime registry that assigns each component a stable type index. Rebuilt at startup; large projects feel this. |
| `BlobAssetReference<T>` | Pointer to a large, read-only data blob stored outside the chunk. Avoids bloating chunk capacity. |
