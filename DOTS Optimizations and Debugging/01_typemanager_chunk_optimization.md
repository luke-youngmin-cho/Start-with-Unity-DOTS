# TypeManager & Chunk Optimization
### Maximize runtime memory and cache efficiency, and identify bottlenecks in the debugger

---

## 1. TypeManager Optimization

| Item | Description | Optimization Tips |
|------|------|-----------|
| **Type Registration (Initialize)** | Scans and tables all `IComponentData` and `ISystem` metadata at player startup | • Use `#if UNITY_EDITOR` on **Editor Only** components to exclude them from builds<br>• On AOT platforms, leverage `StaticTypeRegistry` to eliminate reflection |
| **TypeIndex** | Unique index for each component type | • A large number of types increases Chunk header cache misses → _consolidate unnecessary Scriptable components_ |
| **Field Offset / Alignment** | Internal field alignment of Unmanaged Components | • Pack `struct` fields with **16-byte or smaller** alignment<br>• Use `byte flags` bitfields instead of many unnecessary `bool` fields |

> **Startup Measurement**: Check the `TypeManager.Initialize()` frame time in the Profiler and remove unnecessary components and systems.

---

## 2. Chunk Optimization

| Category | Symptom | Solution |
|----------|------|--------|
| **Chunk Utilization** | **Util < 60 %** in `Entities Debugger ▸ Chunks` | • Increase the number of entities with the same Archetype to **cluster** storage<br>• For large `IBufferElement`, pre-allocate capacity with `ResizeUninitialized()` |
| **Fragmentation** | Chunk splits due to frequent Add/Remove operations | • Toggle with `IEnableableComponent` ↔ minimize StructuralChange |
| **Component Layout** | Exceeds 16 KB and spans two Chunks | • Externalize large buffers/Blobs via **BlobAssetReference** |
| **Shared Component Split** | Chunk moves when the value changes | • Move frequently changing variables to Unmanaged fields |
| **Memory Bandwidth** | Cache misses in Burst loops | • Recommend component size ≤ 64 bytes<br>• Use `float3x4` instead of unnecessary `float4x4` |

### 2‑1 Chunk Capacity Calculation Example
```text
ChunkSize (16 KB) / ComponentSize (Position+Velocity = 24 bytes) ≈ 682 entities
```
*If the component grows to 40 bytes, capacity drops to 409 — a 40% reduction.*

---

## 3. Editor & Debugging Tools

| Tool | Location | Usage Points |
|------|------|-------------|
| **Entities Hierarchy ▸ Statistics** | Window ▸ Entities ▸ Hierarchy | • Check World, Archetype count, Chunk count, and average Util |
| **Systems Window ▸ Frame Time Column** | Window ▸ Entities ▸ Systems | • Identify systems with the highest update times |
| **Chunks View** | Right side of Hierarchy ▸ Chunks button | • Check each Chunk's Component size, Util%, Version, and EnableMask |
| **Query Window** | Window ▸ Entities ▸ Query | • Compare the number of Chunks/entities included in query results |
| **Profiler ▸ Memory · Timeline** | Window ▸ Analysis ▸ Profiler | • Explore `ArchetypeChunkData` memory and SyncPoint events |

---

## 4. Optimization Workflow

1. Use the **Statistics window** to find Archetypes with average Chunk Util < 60%
2. Check whether `SharedComponent` value changes cause Chunk splits and refactor
3. For Chunks with rapidly growing buffer length, adjust capacity (`ResizeUninitialized`)
4. Where **JobFence / WaitForJobGroup** is long in the Profiler `Timeline`, this indicates a StructuralChange spike → switch to Enableable or Deferred approaches
5. Finally, **Build & Run** and validate memory usage with the **Player Build Report**

---

## 5. Checklist

- [ ] Is **TypeManager.Initialize** time under 10 ms?
- [ ] Is **Chunk Utilization** above 70%? (excluding large Prefabs)
- [ ] Is the frequency of `ISharedComponentData` value changes less than once per frame?
- [ ] Is the number of Empty / Fragmented Chunks not excessive?
- [ ] Have you found HotPaths with many cache misses in Burst Jobs in the Profiler?

> By periodically inspecting TypeManager and Chunks, you can improve both **CPU cache efficiency** and **memory usage** simultaneously.
> Use **debugging tools** to quickly find bottlenecks and optimize based on the guidelines above.
