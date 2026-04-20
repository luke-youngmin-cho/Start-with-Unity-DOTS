# Systems / Entity / Component Inspector & Query Window Guide
### Visualize runtime data and diagnose issues quickly

Unity Entities includes several convenient **Editor tools** such as the **Systems Window**, **Entities Hierarchy/Inspector**, **Components Inspector**, and **Query Window**.
When used properly, they help you quickly find performance bottlenecks, incorrect data states, and dependency errors.

---

## 1. Systems Window

| Location | Window ▸ Entities ▸ **Systems** |
|------|--------------------------------|
| Main Tabs | **Group Tree**, **Frame Time**, **Dependencies**, **Queries** |

### 1‑1 Group Tree
* Displays the current World's **SystemGroup** hierarchy and execution order as a tree
* **Colors**
  * **Green** – Executed in the current frame
  * **Gray** – `[DisableAutoCreation]` or condition not met
* **Right-click ▸ Focus** to highlight only the selected system

### 1‑2 Frame Time Column
* Displays each system's **Update time** (ms) and **Self time**
* Sort to quickly identify **top bottleneck** systems

### 1‑3 Dependencies Tab
* Visualizes **UpdateBefore/After** and JobHandle dependency graphs between systems
* **Red lines**: circular dependency warnings, used to spot incorrect Order settings

---

## 2. Entities Hierarchy & Inspector

| Location | Window ▸ Entities ▸ **Hierarchy** |
|------|------------------------------------|

### 2‑1 Hierarchy View Modes
| Mode | Description |
|------|------|
| **GameObjects** | SubScene Open → displays Authoring tree |
| **Entities** | Tree of actual baked Entities |
| **Mixed** | Displays both |

* At runtime, you can navigate at the **Chunk / Archetype / Entity** level in **Entities mode**

### 2‑2 Inspector Panel
* View and edit the **field values** of the selected Entity/Component in real time
* Displays metadata such as **Version**, **EnableMask**, and **Chunk Index**
* **Live Conversion**: Modifications to Authoring values are immediately reflected in baked results

> **Tip** Editing component values at runtime immediately applies logic in that frame, making debugging easier.

---

## 3. Component Inspector (Components Window)

| Location | Window ▸ Entities ▸ **Components** |
|------|-------------------------------------|
| Features | Displays the list of **component types** across the entire World and the **number of entities** owning each |

* Double-click a specific component → automatically sets the filter in the **Query Window**
* **Enable-able** components show the **On/Off ratio** as a bar

---

## 4. Query Window

| Location | Window ▸ Entities ▸ **Query** |
|------|-------------------------------|

### 4‑1 Query Builder
* Set query conditions with **With All / With Any / With None / With Disabled** fields
* Immediately reflects the number of resulting entities and Chunks → verify filter effects

### 4‑2 Live Query
* Displays **EntityQuery** used by other systems in an auto-tracking list
* Double-click an entry → highlights the relevant system and focuses the Inspector
* **Prefab icon**: makes identifying prefab entities easy

### 4‑3 Compound Query Debugging
1. Identify the problematic query in the Systems Window ▸ **Queries** column
2. Open it in the Query Window and adjust conditions
3. From the resulting Chunk info, check whether **SharedComponent** causes splits and verify EnableMask state

---

## 5. Integrated Debugging Workflow

1. Detect **FPS drop** → select the bottleneck frame in the **Profiler**
2. Sort **Systems Window** Frame Time → check top systems
3. Double-click a system → analyze update cycles/waits in the **Dependencies tab**
4. Open that system's query in the **Query Window** to check entity count and Chunk Util
5. For entity details, use the **Hierarchy Inspector** to verify Component values
6. Re-measure with the **Profiler** after applying fixes

---

## 6. Best Practices

| Situation | Tool | Action |
|------|------|------|
| Chunk Util < 50 % | Hierarchy ▸ Chunks | Optimize SharedComponent values |
| System Update=0 ms, Self=0 ms | Systems Window | Actual query result is 0? → Add `[RequireMatchingQueries]` |
| Missing Prefab query | Query Window | Check the prefab filter (`IsPrefab`) |
| StructuralChange Spike | Profiler Timeline → SyncPoint | Check system location and switch to ECB pattern |

---

## 7. Checklist

- [ ] Have you identified the **Top 5** bottleneck systems via the Systems Window Frame Time column?
- [ ] Have you analyzed the problematic queries in detail via the Query Window?
- [ ] Have you fixed incorrect component values or Enable states in the Entities Inspector?
- [ ] Have you checked Fragmentation / Utilization with the Chunk View?

> By actively using **Editor debugging tools**, you can visually identify and resolve issues without modifying code.
> Build a habit of using these tools regularly to ensure both **performance optimization** and **data accuracy**.
