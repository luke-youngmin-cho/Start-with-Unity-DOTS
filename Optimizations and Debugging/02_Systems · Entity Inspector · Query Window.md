# Systems · Entity Inspector · Query Window
### Unity 6000.5 · Entities 6.5.0

---

## 1. Overview

The three Entities tooling windows you'll live in:

| Window | Opens via | Answers |
|--------|----------|---------|
| **Entities Hierarchy** | `Window → Entities → Hierarchy` | "What entities exist? What components do they carry?" |
| **Systems** | `Window → Entities → Systems` | "What systems are running, in what order, and how long do they take?" |
| **Query** | `Window → Entities → Query` | "Given a component set, which entities match? Which systems query it?" |

A fourth, **Archetypes** (`Window → Entities → Archetypes`), is covered in [`01_Chunk Layout & TypeManager.md`](01_Chunk%20Layout%20%26%20TypeManager.md).

---

## 2. Entities Hierarchy

Acts as the live Hierarchy but for entities rather than GameObjects.

### World picker (top of window)

Switch between `DefaultGameObjectInjectionWorld` (single-player), `ClientWorld`/`ServerWorld` (Netcode), or a test fixture world. Changing the world re-runs the tree against that world's entities.

### Tree view

- Each row is an entity; child rows are entities linked via `Parent` / `LinkedEntityGroup`.
- Right-column counts: number of components, number of children.
- Click a row → its components appear in the **Inspector**.

### Search

- `c:Health` — filter to entities with a `Health` component.
- `n:Enemy` — name filter.
- `w:Server` — world filter.
- Combine: `c:Health c:Enemy n:boss`.

### Inspector

The Inspector for an entity shows:

- **Components** section — each `IComponentData` + its current values. Editable live in Play mode.
- **Relationships** — links to other entities (Parent, Child, LinkedEntityGroup).
- **Enabled state** — per `IEnableableComponent` toggle.

Live editing is **write-through**: changing a field immediately bumps that chunk's change version.

---

## 3. Systems

The Systems window lists every registered system, grouped by its `SystemGroup`, in the order the scheduler actually ticks them.

### Columns to read first

| Column | Meaning |
|--------|---------|
| **Name** | The system type. |
| **Namespace** | Where it lives (often identifies a package vs your code). |
| **Time (ms)** | Main-thread time spent in `OnUpdate`. Updates ~every frame. |
| **Queries** | Number of queries the system has touched. |
| **Entity Count** | Entities matched by the primary query. |

### What to look for

| Reading | Interpretation |
|---------|----------------|
| A system shows 0 ms but is green | Skipped by `RequireForUpdate<T>` or `[RequireMatchingQueriesForUpdate]`. Healthy. |
| A system shows consistent high ms | Actual work. Drill into the Profiler for a breakdown. |
| A system shows `--` for entity count | No matching query (or the query hasn't been built yet). |
| Two systems appear out of the expected order | `[UpdateBefore]` / `[UpdateAfter]` chain is wrong, or they're in different groups. |

### Enable / disable

Each system has a checkbox — untick to skip it in Play mode without recompiling. Great for A/B'ing.

### System detail pane

Selecting a system opens a side pane with:

- **Queries** — each query's component set (All/Any/None/Disabled/Present/Absent).
- **Dependencies** — systems this one `[UpdateBefore]`/`[UpdateAfter]`, resolved to the actual schedule.
- **Type Info** — whether `[BurstCompile]` is on the class and on each method.

The **Queries tab now shows Disabled / Present / Absent / None component states** in 1.4+ / 6.x — so enableable queries no longer look ambiguous.

---

## 4. Query window

Purpose-built for "who matches X?" questions.

Workflow:

1. Add `All` components via the plus button.
2. Add `Any` / `None` if needed.
3. The window lists **matching entities** in the left panel and **systems that run this query** in the right panel.

Use cases:

- "Which systems read `Health`?" → add `Health` to All; every system listed on the right touches it.
- "Why is this entity not matching my system's query?" → paste the system's query, compare component presence against the entity in the Hierarchy window.
- "Are any prefabs slipping into this query?" → prefabs are distinguished by icon in the result list (added in 1.4+).

The Query window also lets you save a query for reuse during the session.

---

## 5. Journaling — the last-resort diagnostic

`Window → Analysis → Entities Journaling` (enabled via Project Settings → Entities → Journaling) records every structural change, component write, and system update into a ring buffer. Playback views:

- **Operations** — each recorded change with timestamp, source system, and affected entity.
- **Entities** — per-entity timeline of who touched it and when.

Journaling has runtime cost — turn it on only while investigating a bug, off in normal development.

---

## 6. Field workflow — debugging a missing update

Common bug: "My system processed the entity but the component didn't change." Walkthrough:

1. **Entities Hierarchy** → find the entity, confirm it has the component.
2. **Inspector** → before Play, note the value. Press Play and watch it.
3. If the value doesn't change: **Systems window** → find the relevant system, check **Time (ms)**. If 0, it's skipped.
4. If time > 0 but value still doesn't change: **Query window** → paste the system's query + the entity's components. Verify the entity matches.
5. Still stuck: enable **Journaling** for 1-2 frames. Look for writes to that component from other systems that stomp yours.

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| Entities Hierarchy is empty in Play mode | World picker set to the wrong world (e.g. Client only, no Server). |
| A live field doesn't update | You're viewing a cached entity after a domain reload. Reselect it. |
| Systems window says 0 ms but my system definitely runs | Profiler may show it under a chunk job's marker if the actual work is in `ScheduleParallel`. Main-thread `OnUpdate` is genuinely 0 in that case — look at job timings in the Profiler. |
| Query window finds an entity my system doesn't | The system adds a `WithAll<T>()` or change filter that the Query window wasn't told about. Match them exactly. |
| Journaling window is empty | Journaling isn't enabled in Project Settings, or the buffer rolled over. |
| Entities and GameObjects look out of sync | SubScene is open (white icon) — close it (yellow icon) so the baker runs and entities appear. |
