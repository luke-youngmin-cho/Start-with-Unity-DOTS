---
title: Singleton Component
updated: 2026-04-21
folder: DOTS Workflows
---

# Singleton Component
### Unity 6000.5 · Entities 6.5.0

---

## 1. What a singleton is (and isn't)

A **singleton** in Entities is just a component that is expected to exist on **exactly one entity** in the world. There is no special `ISingleton` interface — any `IComponentData` becomes a singleton by convention, and `SystemAPI` provides ergonomic accessors that assert the "exactly one" rule.

Typical uses: game state, global config, time/tick sources, network state, central ECB registrations.

```csharp
public struct GameState : IComponentData
{
    public int   Score;
    public float TimeScale;
    public bool  Paused;
}
```

---

## 2. The accessors

Inside any `ISystem` / `SystemBase`:

```csharp
// Reads
GameState gs      = SystemAPI.GetSingleton<GameState>();
RefRW<GameState> gsRW  = SystemAPI.GetSingletonRW<GameState>();
bool has          = SystemAPI.HasSingleton<GameState>();
bool got          = SystemAPI.TryGetSingleton<GameState>(out var gs2);

// Writes
SystemAPI.SetSingleton(new GameState { Score = 10 });

// Entity handle, if you need it
Entity e          = SystemAPI.GetSingletonEntity<GameState>();

// Buffers work too
DynamicBuffer<Waypoint> buf = SystemAPI.GetSingletonBuffer<Waypoint>();
```

`Get*` variants throw if zero or more than one singleton exists. `TryGet*` returns `false` in those cases. `RW` variants mark the component as changed for change-tracking.

---

## 3. Creating the singleton

Two common patterns:

### 3.1 Bake it from a SubScene

```csharp
public class GameStateAuthoring : MonoBehaviour
{
    public float InitialTimeScale = 1f;

    class Baker : Baker<GameStateAuthoring>
    {
        public override void Bake(GameStateAuthoring authoring)
        {
            var e = GetEntity(TransformUsageFlags.None);
            AddComponent(e, new GameState
            {
                Score     = 0,
                TimeScale = authoring.InitialTimeScale,
                Paused    = false
            });
        }
    }
}
```

Attach `GameStateAuthoring` to a single GameObject in the SubScene; the Baker produces exactly one entity with the singleton.

### 3.2 Create at system startup

```csharp
public void OnCreate(ref SystemState state)
{
    var e = state.EntityManager.CreateEntity(typeof(GameState));
    state.EntityManager.SetComponentData(e, new GameState { TimeScale = 1f });
}
```

Use this for data that shouldn't live in authoring (server-injected config, procedurally generated, etc.).

---

## 4. Gating a system on the singleton

If a system only makes sense after the singleton exists:

```csharp
public void OnCreate(ref SystemState state)
{
    state.RequireForUpdate<GameState>();
}
```

`RequireForUpdate<T>()` tells the system to **skip `OnUpdate` entirely** when the component isn't present. Avoids defensive null checks inside `OnUpdate`.

---

## 5. Read-only vs read-write performance

- `GetSingleton<T>()` — reads a copy. No change tracking.
- `GetSingletonRW<T>()` — returns a `RefRW<T>` pointing into the chunk. Assigning `.ValueRW` marks the component changed, which is what you want if other systems use `WithChangeFilter<T>`.
- `SetSingleton` — full write, always bumps the version.

Prefer `GetSingleton` in read-heavy systems; switch to `GetSingletonRW` only when you actually mutate.

---

## 6. Multiple candidates — the "singleton with more than one" trap

`GetSingleton<T>` throws `InvalidOperationException: expected exactly one entity with component T, found N` when two entities end up with the same singleton component. Common causes:

- Two GameObjects in the SubScene both have the authoring script attached.
- A test fixture created the singleton at runtime **and** the SubScene also bakes it.
- A system creates the singleton in `OnCreate` without guarding with `HasSingleton`.

Defensive pattern:

```csharp
public void OnCreate(ref SystemState state)
{
    if (!SystemAPI.HasSingleton<GameState>())
    {
        var e = state.EntityManager.CreateEntity(typeof(GameState));
        state.EntityManager.SetComponentData(e, new GameState { TimeScale = 1f });
    }
}
```

---

## 7. Buffer singletons

A `DynamicBuffer<T>` can be a singleton too:

```csharp
public struct QueuedEvent : IBufferElementData
{
    public int Type;
    public int Payload;
}

// Access:
DynamicBuffer<QueuedEvent> events = SystemAPI.GetSingletonBuffer<QueuedEvent>();
events.Add(new QueuedEvent { Type = 1 });
```

Useful for global event queues and command streams.

---

## 8. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `InvalidOperationException: expected exactly one entity with component T, found 0` | Singleton entity was never created. Add a baker, or create in `OnCreate` with a `HasSingleton` guard. |
| `... found 2` | Two entities carry the component. Deduplicate in authoring or cleanup at startup. |
| `GetSingletonRW` doesn't appear to mark the component changed | You wrote to `.ValueRO` or ignored the `RefRW`. Only `.ValueRW` triggers change tracking. |
| System runs before the singleton is created | Use `RequireForUpdate<T>` or order your systems so the creator runs first (`[UpdateBefore]`). |
| Singleton is modified from two systems with no ordering | Add `[UpdateAfter]`/`[UpdateBefore]` or merge the logic. Race conditions on singletons are especially painful to debug. |
| Netcode: singleton value diverges between client and server | Ghost-sync the singleton, or rebuild it deterministically from synchronised inputs. |
