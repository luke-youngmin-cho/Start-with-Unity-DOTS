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

## 3. 복잡한 Aspect 변환 예시

### 3-1 다중 컴포넌트와 헬퍼 메서드가 포함된 Aspect

#### 기존 코드 (IAspect)
```csharp
public readonly partial struct CombatAspect : IAspect
{
    public readonly RefRW<Health> Health;
    public readonly RefRW<LocalTransform> Transform;
    public readonly RefRO<Damage> Damage;
    public readonly RefRO<CombatStats> Stats;

    public bool IsDead => Health.ValueRO.Value <= 0;
    
    public void TakeDamage(float damage)
    {
        Health.ValueRW.Value = math.max(0, Health.ValueRO.Value - damage);
    }

    public void MoveTowards(float3 target, float speed)
    {
        var direction = math.normalize(target - Transform.ValueRO.Position);
        Transform.ValueRW.Position += direction * speed;
    }

    public float CalculateEffectiveDamage()
    {
        return Damage.ValueRO.Value * Stats.ValueRO.AttackMultiplier;
    }
}

// 사용부
partial struct CombatSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var combatAspect in SystemAPI.Query<CombatAspect>())
        {
            if (!combatAspect.IsDead)
            {
                combatAspect.TakeDamage(10f);
                combatAspect.MoveTowards(float3.zero, 2f);
                var effectiveDamage = combatAspect.CalculateEffectiveDamage();
            }
        }
    }
}
```

#### 변환 후 코드 (Static Helper + Direct Query)
```csharp
// 헬퍼 메서드를 static 클래스로 분리
public static class CombatHelper
{
    public static bool IsDead(in Health health) => health.Value <= 0;
    
    public static void TakeDamage(ref Health health, float damage)
    {
        health.Value = math.max(0, health.Value - damage);
    }

    public static void MoveTowards(ref LocalTransform transform, float3 target, float speed)
    {
        var direction = math.normalize(target - transform.Position);
        transform.Position += direction * speed;
    }

    public static float CalculateEffectiveDamage(in Damage damage, in CombatStats stats)
    {
        return damage.Value * stats.AttackMultiplier;
    }
}

// 변환된 시스템
partial struct CombatSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (health, transform, damage, stats) in 
                 SystemAPI.Query<RefRW<Health>, RefRW<LocalTransform>, 
                               RefRO<Damage>, RefRO<CombatStats>>())
        {
            if (!CombatHelper.IsDead(health.ValueRO))
            {
                CombatHelper.TakeDamage(ref health.ValueRW, 10f);
                CombatHelper.MoveTowards(ref transform.ValueRW, float3.zero, 2f);
                var effectiveDamage = CombatHelper.CalculateEffectiveDamage(
                    damage.ValueRO, stats.ValueRO);
            }
        }
    }
}
```

### 3-2 Entity 필터링과 조건부 컴포넌트 처리

#### 기존 코드 (IAspect + Conditional Components)
```csharp
public readonly partial struct PlayerAspect : IAspect
{
    public readonly Entity Entity;
    public readonly RefRW<LocalTransform> Transform;
    public readonly RefRO<PlayerInput> Input;
    public readonly RefRO<MovementSpeed> Speed;
    
    // Enableable 컴포넌트 처리
    public readonly EnabledRefRO<Sprinting> SprintEnabled;
    
    public float GetCurrentSpeed()
    {
        float baseSpeed = Speed.ValueRO.Value;
        if (SprintEnabled.ValueRO)
            return baseSpeed * 2f;
        return baseSpeed;
    }
}
```

#### 변환 후 코드
```csharp
partial struct PlayerMovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // 기본 이동 처리
        foreach (var (entity, transform, input, speed) in 
                 SystemAPI.Query<Entity, RefRW<LocalTransform>, 
                               RefRO<PlayerInput>, RefRO<MovementSpeed>>())
        {
            float currentSpeed = speed.ValueRO.Value;
            
            // Enableable 컴포넌트 조건부 처리
            if (SystemAPI.IsComponentEnabled<Sprinting>(entity))
            {
                currentSpeed *= 2f;
            }
            
            var movement = input.ValueRO.Direction * currentSpeed * SystemAPI.Time.DeltaTime;
            transform.ValueRW.Position += movement;
        }
    }
}
```

### 3-3 변환 가이드라인

| Aspect 기능 | 변환 방법 | 예시 |
|-------------|-----------|------|
| **프로퍼티 접근자** | `static` 헬퍼 메서드 | `IsDead => static bool IsDead(in Health)` |
| **Entity 참조** | 쿼리에 `Entity` 추가 | `Query<Entity, RefRW<T>>()` |
| **EnabledRef** | `SystemAPI.IsComponentEnabled<T>()` 사용 | 조건부 로직 분기 |
| **복잡한 로직** | 독립적인 `static` 유틸 클래스 | `CombatHelper.CalculateDamage()` |
| **테스트 코드** | Aspect → 개별 함수 단위 테스트 | `Assert.IsTrue(CombatHelper.IsDead(...))` |

---

## 4. 체크리스트

- [ ] `IAspect` 파일을 삭제 또는 주석 처리했는가?  
- [ ] 관련 쿼리를 모두 `SystemAPI.Query` 로 변환했는가?  
- [ ] Aspect 헬퍼 로직이 필요한 경우 `static` 메서드로 이동했는가?  
- [ ] Profiler 로 성능 회귀가 없는지 확인했는가?  
