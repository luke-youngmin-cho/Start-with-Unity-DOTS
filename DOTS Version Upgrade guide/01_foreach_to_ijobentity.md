# Entities.ForEach → IJobEntity 전환 가이드

`Entities.ForEach` API는 1.4에서 사용 중단(Obsolete)되었습니다.  
동일 기능을 **`IJobEntity` + `ScheduleParallel()`** 으로 교체하여  
성능을 유지하면서도 향후 버전 호환성을 확보하세요.

---

## 1. 기존 코드 예시 (SystemBase + Entities.ForEach)

```csharp
protected override void OnUpdate()
{
    float dt = SystemAPI.Time.DeltaTime;

    Entities
        .WithName("MoveSystem")
        .WithNone<Paused>()
        .ForEach((ref LocalTransform t, in Velocity v) =>
        {
            t.Position += v.Value * dt;
        })
        .ScheduleParallel();
}
```

---

## 2. 변경 후 코드 (IJobEntity)

```csharp
using Unity.Burst;
using Unity.Entities;

[BurstCompile]
public partial struct MoveJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref LocalTransform t, in Velocity v)
    {
        t.Position += v.Value * DeltaTime;
    }
}

protected override void OnUpdate()
{
    new MoveJob
    {
        DeltaTime = SystemAPI.Time.DeltaTime
    }.ScheduleParallel();
}
```

### 핵심 변경 포인트
| 포인트 | 설명 |
|--------|------|
| **람다 → Execute** | 람다 본문을 `Execute` 메서드로 이동 |
| **[BurstCompile]** | IJobEntity 자체를 Burst 컴파일 |
| **캡처 변수 전달** | `DeltaTime` 을 `public float` 필드로 전달 |
| **필터 체인** | `.WithNone<Paused>()` 는 `SystemAPI.Query` 체인으로 변환 |

---

## 3. 필터 체인 변환: .WithNone<Paused>()

기존 `Entities.ForEach`에서 사용하던 필터 체인은 `SystemAPI.Query`로 변환할 수 있습니다.

### 3-1 기존 필터 체인 (SystemBase)
```csharp
Entities
    .WithName("MoveSystem")
    .WithNone<Paused>()
    .WithAll<Active>()
    .ForEach((ref LocalTransform t, in Velocity v) =>
    {
        t.Position += v.Value * dt;
    })
    .ScheduleParallel();
```

### 3-2 SystemAPI.Query를 사용한 변환
```csharp
protected override void OnUpdate()
{
    var query = SystemAPI.QueryBuilder()
        .WithAll<LocalTransform, Velocity, Active>()
        .WithNone<Paused>()
        .Build();

    new MoveJob
    {
        DeltaTime = SystemAPI.Time.DeltaTime
    }.ScheduleParallel(query);
}
```

### 3-3 IJobEntity 내장 필터 (권장)
```csharp
[BurstCompile]
public partial struct MoveJob : IJobEntity
{
    public float DeltaTime;

    // IJobEntity는 Execute 메서드 파라미터로 필터링 자동 처리
    void Execute(ref LocalTransform t, in Velocity v, 
                 in Active active)  // WithAll<Active>와 동일
    {
        // WithNone<Paused>는 Execute 파라미터에서 제외하면 자동 필터링됨
        t.Position += v.Value * DeltaTime;
    }
}
```

| 필터 종류 | SystemBase | IJobEntity |
|----------|------------|------------|
| **WithAll<T>** | `.WithAll<T>()` | Execute 파라미터에 포함 |
| **WithNone<T>** | `.WithNone<T>()` | Execute 파라미터에서 제외 |
| **WithAny<T>** | `.WithAny<T>()` | `SystemAPI.QueryBuilder()` 사용 |

---

## 4. 체크리스트

- [ ] `[BurstCompile]` 어트리뷰트 추가했는가?  
- [ ] 캡처 값(시간, 상수 등)을 **job 필드**로 전달했는가?  
- [ ] `Entities.ForEach` 호출이 프로젝트에 더 남아 있지 않은가?  
- [ ] Profiler 로 성능이 유지 또는 향상됐는지 확인했는가?  
