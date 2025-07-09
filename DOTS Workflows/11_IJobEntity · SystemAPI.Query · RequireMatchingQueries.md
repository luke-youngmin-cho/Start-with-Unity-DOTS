# IJobEntity · SystemAPI.Query · RequireMatchingQueries
### 고성능 엔티티 쿼리 & 잡(Job) 작성 핵심 가이드

Entities 1.4에서는 **`IJobEntity`** 와 **`SystemAPI.Query`** 가 `Entities.ForEach`를 대체하는 주력 API입니다.  
`RequireMatchingQueries` 속성은 **쿼리 누락/오류를 컴파일 타임에 방지**하여 안전성을 높여 줍니다.

---

## 1. IJobEntity 기본

| 항목 | 설명 |
|------|------|
| **정의 방식** | `public partial struct MyJob : IJobEntity { void Execute(…); }` |
| **자동 생성** | 소스 제너레이터가 `IJobChunk` 코드로 변환 → Burst 최적화 |
| **스케줄** | `.Schedule()`, `.ScheduleParallel()`, `.Run()` |
| **버스트** | `[BurstCompile]` 직접 선언 |
| **파라미터** | `in`/`ref` 컴포넌트, `[ChunkIndexInQuery]` 등 어트리뷰트 지원 |

### 1‑1 예시: 이동 시스템
```csharp
[BurstCompile]
public partial struct MoveJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref LocalTransform t, in Velocity v)
    {
        t.Position += v.Value * DeltaTime;
    }
}

// 시스템 내 호출
new MoveJob { DeltaTime = SystemAPI.Time.DeltaTime }
    .ScheduleParallel();
```

> **Tip** 잡에 캡처할 값은 **public 필드**로 전달 (Burst 안전).

---

## 2. SystemAPI.Query 패턴

| 사용 패턴 | 설명 |
|-----------|------|
| `foreach (var (rw, ro) in SystemAPI.Query<RefRW<A>, RefRO<B>>())` | `Ref*` 튜플 반환 |
| `.WithNone<C>()`, `.WithAll<D>()`, `.WithDisabled<E>()` | 쿼리 필터 체인 |
| `.ForEach((RefRW<A> a, in B b) => { … })` *(lambda)* | 소규모 엔티티 시 편리 |
| `SystemAPI.QueryBuilder().WithAspect<SomeAspect>()` | Aspect 필터 |

### 2‑1 예시: 체력 0 이하 엔티티 파괴
```csharp
foreach (var (e, health) in
        SystemAPI.Query<Entity, RefRO<Health>>()
                 .WithNone<DeadTag>())
{
    if (health.ValueRO.Value <= 0)
    {
        state.EntityManager.AddComponent<DeadTag>(e);
    }
}
```

---

## 3. RequireMatchingQueries 속성

```csharp
[RequireMatchingQueriesForUpdate]
public partial struct DamageSystem : ISystem
{
    void OnUpdate(ref SystemState state)
    {
        // Query 예시
        foreach (var hp in SystemAPI.Query<RefRW<Health>>()) { … }
    }
}
```

| 효과 | 설명 |
|------|------|
| **빈 쿼리 방지** | 모든 쿼리 결과가 0개면 **Update 호출 스킵** |
| **성능** | 불필요한 시스템 호출 제거, 프레임 시간 절약 |
| **주의** | 쿼리에 `.WithNone<>` 만 있고, 실제 매치가 없으면 시스템이 완전히 실행되지 않을 수 있음 |

---

## 4. IJobEntity vs SystemAPI.Query 비교

| 기준 | **IJobEntity** | **SystemAPI.Query (foreach)** |
|------|----------------|--------------------------------|
| **스레딩** | `ScheduleParallel()` 로 다중 코어 | 기본 싱글 스레드 |
| **오버헤드** | 스케줄 비용 존재, 대규모 엔티티에 효율 | 소규모/조건부 로직 간단 |
| **코드 길이** | 분리 구조체 정의 필요 | 메서드 내부 간결 |
| **Burst** | 구조체 전체 Burst | 루프 내부 Burst(가능) |
| **사용 시점** | 물리, 이동 등 반복 계산 | 이벤트, 드문 조건 검사 |

---

## 5. 베스트 프랙티스

1. **수천 엔티티 반복** → `IJobEntity.ScheduleParallel()` 적용.  
2. **엔티티 < 100** & 간단 로직 → `SystemAPI.Query` + `foreach` 사용.  
3. 시스템에 `[RequireMatchingQueriesForUpdate]` 추가하여 **빈 프레임 호출** 방지.  
4. `IJobEntity` 내부에서 `EntityCommandBuffer.ParallelWriter` 활용해 **StructuralChange 기록**.  
5. **프로파일링**: Job Schedule vs MainThread 루프 시간 비교, 오버헤드가 이득보다 크지 않은지 확인.

---

## 6. 체크리스트

- [ ] `Execute` 메서드에 Burst 불가 타입을 사용하지 않았는가?  
- [ ] `SystemAPI.Query` 에 `.WithAll<SimulationTag>()` 등 필터를 명확히 추가했는가?  
- [ ] `RequireMatchingQueriesForUpdate` 로 불필요한 전프레임 호출을 줄였는가?  
- [ ] 잡에서 `ComponentLookup` 사용 시 `isReadOnly` 플래그를 맞게 지정했는가?  

> 올바른 쿼리·잡 선택과 `RequireMatchingQueries` 활용은 **CPU 프레임 시간**과 **코드 안전성**을 동시에 향상시켜 줍니다. 자신의 시스템마다 엔티티 수·변경 빈도를 측정해 가장 적합한 방법을 적용해 보세요.
