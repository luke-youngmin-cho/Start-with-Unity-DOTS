# IAspect 제거 대응 가이드 – 직접 EntityQuery 코드로 전환

`IAspect` API는 1.4에서 사용 중단 예정입니다.  
아래 방법으로 **직접 `SystemAPI.Query` + `RefRW / RefRO`** 패턴으로 변환하세요.

---

## 1. 기존 Aspect 코드

```csharp
public readonly partial struct MovementAspect : IAspect
{
    public readonly RefRW<LocalTransform> Transform;
    public readonly RefRO<Velocity> Velocity;
}

partial struct MoveSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var aspect in SystemAPI.Query<MovementAspect>())
        {
            aspect.Transform.ValueRW.Position +=
                aspect.Velocity.ValueRO.Value * SystemAPI.Time.DeltaTime;
        }
    }
}
```

---

## 2. 변환 후 코드

```csharp
public partial struct MoveSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (transform, velocity) in
                 SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * dt;
        }
    }
}
```

### 변경 포인트
| 항목 | 대체 방법 |
|------|-----------|
| **Aspect 필드** | `RefRW<T>`, `RefRO<T>` 튜플로 직접 사용 |
| **Aspect Helper** | `static` 유틸 메서드로 이동 |
| **쿼리** | `SystemAPI.Query<…>` 로 작성 |
| **테스트 코드** | Aspect 단위 테스트 → 함수 단위 테스트로 리팩터 |

---

## 3. 체크리스트

- [ ] `IAspect` 파일을 삭제 또는 주석 처리했는가?  
- [ ] 관련 쿼리를 모두 `SystemAPI.Query` 로 변환했는가?  
- [ ] Aspect 헬퍼 로직이 필요한 경우 `static` 메서드로 이동했는가?  
- [ ] Profiler 로 성능 회귀가 없는지 확인했는가?  
