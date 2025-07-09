# Entities 1.4 — 사용 중단 API & 대체 방법 가이드

## 1. 사용 중단 API 개요

| API | 상태 | 권장 대체 |
|-----|------|-----------|
| `Entities.ForEach` | **Obsolete** (1.4부터) | `IJobEntity` 또는 `SystemAPI.Query` |
| `Job.WithCode` | **Obsolete** | 전용 `IJob` 구조체 작성 |
| `IAspect` 및 관련 소스 제너레이터 | **Obsolete** (향후 제거 예정) | 개별 `Component` / `EntityQuery` API |
| `ComponentLookup.GetRefRWOptional`<br>`ComponentLookup.GetRefROOptional` | **Obsolete** | `TryGetRefRW` / `TryGetRefRO` |

> **참고**  
> 위 API들은 1.x 버전에서는 여전히 동작하지만 **다음 메이저 버전에서 제거**될 예정입니다. 지금 미리 교체해 두면 이후 업그레이드가 수월합니다.

---

## 2. 대표 마이그레이션 예시

### 2‑1 `Entities.ForEach` → `IJobEntity`

```csharp
// 이전 코드 : SystemBase + Entities.ForEach
protected override void OnUpdate()
{
    float dt = SystemAPI.Time.DeltaTime;
    Entities
        .ForEach((ref LocalTransform t, in RotationSpeed s) =>
        {
            t.Rotation = math.mul(
                math.normalize(t.Rotation),
                quaternion.AxisAngle(math.up(), s.RadiansPerSecond * dt));
        })
        .ScheduleParallel();
}

// 새 코드 : IJobEntity + Burst
[BurstCompile]
public partial struct RotateJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref LocalTransform t, in RotationSpeed s)
    {
        t.Rotation = math.mul(
            math.normalize(t.Rotation),
            quaternion.AxisAngle(math.up(), s.RadiansPerSecond * DeltaTime));
    }
}
protected override void OnUpdate()
{
    new RotateJob { DeltaTime = SystemAPI.Time.DeltaTime }
        .ScheduleParallel();
}
```

### 2‑2 `ComponentLookup.GetRef*Optional` → `TryGetRef*`

```csharp
// 이전
if (lookup.GetRefROOptional(entity, out var compRO))
{
    // ...
}

// 새로
if (lookup.TryGetRefRO(entity, out var compRO))
{
    // ...
}
```

`TryGetRef*` 는 **존재 여부 확인과 참조 반환**을 한 번에 수행하여 코드가 더 간결하고 안전합니다.

### 2‑3 `IAspect` 제거

* Aspect에 묶여 있던 필드는 **개별 컴포넌트** 혹은  
  `SystemAPI.Query<RefRW<T>, …>` 패턴으로 직접 다룹니다.  
* 공통 로직이 필요하면 별도 **static 헬퍼 메서드**로 분리하여 시스템을 깔끔하게 유지합니다.

---

## 3. 교체 시 유의사항

1. **성능** — `IJobEntity` / `SystemAPI.Query` 는 내부적으로 `IJobChunk` 코드로 변환되어 `Entities.ForEach` 보다 런타임·컴파일 속도가 우수합니다.  
2. **Burst 컴파일** — `IJobEntity` 는 기본적으로 Burst 비활성화이므로 반드시 `[BurstCompile]` 어트리뷰트를 추가하세요.  
3. **정적 분석** — 마이그레이션 후에도 Obsolete 경고가 남아 있다면 IDE 검색으로 잔여 호출을 찾아 일괄 수정하세요.  
4. **테스트** — 구조 변경 로직을 수정했다면 Profiler 로 **Sync Point** 발생 여부를 확인해 성능 회귀를 방지합니다.

---

## 4. 단계별 마이그레이션 체크리스트

1. 프로젝트 전역에서 사용 중단 API 호출 검색  
2. 위 예시 패턴을 따라 교체하고 컴파일  
3. `[BurstCompile]` 지정 후 **Profiler** 로 성능 검증  
4. `System Inspector → Dependencies` 탭으로 스케줄·의존 관계 확인  
5. 패키지 **Changelog / Upgrade Guide** 로 추가 변경 사항 반영
