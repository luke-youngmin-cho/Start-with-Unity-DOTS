# System 작성 가이드 – **SystemBase(Managed) vs ISystem(Unmanaged)**

Unity Entities 1.4부터는 **SystemBase**(클래스 기반) 과 **ISystem**(구조체 기반) 두 가지 스타일을 모두 지원합니다.  
아래 비교표와 예시를 참고하여 **성능**·**API 편의성**·**GC 비용** 등을 종합적으로 고려해 선택하세요.

---

## 1. 구조 비교 한눈에 보기

| 항목 | **SystemBase** (클래스) | **ISystem** (struct) |
|------|------------------------|----------------------|
| **런타임 타입** | Managed 객체 | Unmanaged 값 타입 |
| **상태 필드** | `class` 멤버 필드 (GC Heap) | `ref SystemState state` 통해 접근하는 **`SystemState`** |
| **OnCreate / OnDestroy** | 가상 메서드 (`protected override`) | 선택 메서드 (`void OnCreate(ref SystemState)` 등) |
| **Burst 기본** | ❌ (직접 Burst 불가, 내부 잡만 Burst) | ✅ (구조체 코드 자체도 Burst 대상) |
| **GC 부담** | 필드 당 GC Heap 사용 | 없음 |
| **Reflection API** | UnityEngine / Managed API 사용 자유 | Unmanaged 제약 (가능하면 Burst‑Friendly API 사용) |
| **Entities.ForEach** | 지원 (Obsolete) | 사용 불가 |
| **IJobEntity / Query** | 완전 지원 | 완전 지원 |
| **컴파일 속도** | 느림 (소스 제너레이터 + 클래스) | 빠름 |
| **용도** | 복잡한 게임플레이, UnityEngine 의존 시스템 | 핫루프, 대량 연산, 서버 사이드, NetCode |

---

## 2. 기본 골격

### 2‑1 SystemBase 예시
```csharp
using Unity.Entities;
using Unity.Burst;

public partial class MoveSystem : SystemBase
{
    protected override void OnCreate()
    {
        // Managed 필드 가능
        random = new Unity.Mathematics.Random(1);
    }

    protected override void OnUpdate()
    {
        float dt = Time.DeltaTime;

        Entities
            .WithNone<Paused>()
            .ForEach((ref LocalTransform t, in Velocity v) =>
            {
                t.Position += v.Value * dt;
            })
            .ScheduleParallel();
    }

    private Unity.Mathematics.Random random; // 예시
}
```

### 2‑2 ISystem 예시
```csharp
using Unity.Entities;
using Unity.Burst;

[BurstCompile]
public partial struct MoveSystemIS : ISystem
{
    public void OnCreate(ref SystemState state) { }

    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (t, v) in
                 SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>()
                          .WithNone<Paused>())
        {
            t.ValueRW.Position += v.ValueRO.Value * dt;
        }
    }

    public void OnDestroy(ref SystemState state) { }
}
```

> **주의** `ISystem` 안에서는 **`SystemAPI`** 를 통해 쿼리·시간·싱글톤에 접근합니다.

---

## 3. 상태(State) 관리 차이

| 주제 | SystemBase | ISystem |
|------|------------|---------|
| **멤버 필드** | 자유롭게 추가 (GC Heap) | Unmanaged 타입만 가능 (struct 필드) |
| **저장 방식** | `this.field` | 구조체 필드 또는 **Singleton Component** 저장 |
| **인스펙터 노출** | Entities Debugger에서 필드 확인 불가 | 동일 (필드가 없으므로) |

### 3‑1 ISystem에서 상태 저장 예시
```csharp
// ISystem에서는 구조체 내부에 필드를 직접 저장할 수 있습니다
[BurstCompile]
public partial struct MoveSystemWithState : ISystem
{
    // 상태를 구조체 필드로 저장 (Unmanaged 타입만 가능)
    private Unity.Mathematics.Random random;
    private float elapsedTime;
    
    public void OnCreate(ref SystemState state)
    {
        // 구조체 필드 초기화
        random = new Unity.Mathematics.Random(1);
        elapsedTime = 0f;
    }

    public void OnUpdate(ref SystemState state)
    {
        // 구조체 필드 사용
        elapsedTime += SystemAPI.Time.DeltaTime;
        
        // 복잡한 상태가 필요한 경우 Singleton 컴포넌트 사용
        // 예: SystemAPI.GetSingletonRW<GameState>()
    }
}

// 또는 Singleton Component를 활용한 상태 관리
public struct SystemState : IComponentData 
{ 
    public int TickCount;
    public float AccumulatedTime;
}
```

---

## 4. Burst & 성능

* `ISystem` 전체가 **BurstCompile** 대상 → 조건 분기·루프가 많아도 네이티브 최적화.  
* `SystemBase`는 C# 클래스 코드가 **Burst 밖**에 있으므로, 핫루프는 반드시 Burst 잡으로 분리해야 함.  
* 빈번한 호출·프레임 임계(>90 fps) 로직은 ISystem을 우선 고려.

---

## 5. API 사용 제약

| 범주 | SystemBase | ISystem |
|------|-----------|---------|
| **UnityEngine API** | 사용 가능 (`Time`, `Random.Range`, `Debug.Log`) | **불가** 또는 Burst Off 필요 |
| **Job.WithCode / Entities.ForEach** | Legacy 지원 | 미지원 |
| **Managed 객체 캡처** | 람다 안 캡처 가능 (GC) | 불가 또는 Burst Off |
| **NetCode Ghost** | Server/Client Worlds에서 `ISystem` 선호 | 둘 다 가능 |

---

## 6. 선택 기준 가이드

1. **핫루프·연산 집약** → **ISystem**  
2. **복잡한 상태·클래스 참조** 필요 → **SystemBase**  
3. **편의성 우선(빠른 프로토타이핑)** → SystemBase + 추후 최적화 시 ISystem으로 리팩터  
4. **패키지 샘플 코드 준수** → NetCode/Physics 등은 최신 샘플이 대부분 ISystem 기반  
5. **Editor 전용** ‑ BakingSystem, Conversion 단계 → ISystem (BakingSystem) 사용

---

## 7. 혼합 전략

* **상위 순서 제어** : SystemGroup 내에서 SystemBase ↔ ISystem을 함께 배치 가능 (`[UpdateInGroup]`, `[OrderFirst/Last]`).  
* **다중 모드** : GameLogicSystem (SystemBase)이 **이벤트** 생성 → Physics/Animation 계산은 ISystem 잡으로 분리.  
* **점진적 마이그레이션** : 성능 병목을 개별 System 단위로 ISystem 변환, 나머지는 유지.

---

## 8. SystemBase → ISystem 단계별 전환 가이드

### 8-1 전환 준비 단계

#### 1단계: 기존 SystemBase 코드 분석
```csharp
// 기존 SystemBase 코드 예시
public partial class WeaponSystem : SystemBase
{
    private float lastShotTime;          // 상태 필드 1
    private Dictionary<Entity, float> cooldowns; // 상태 필드 2
    private Random random;               // 상태 필드 3

    protected override void OnCreate()
    {
        random = new Random(42);
        cooldowns = new Dictionary<Entity, float>();
    }

    protected override void OnUpdate()
    {
        float currentTime = Time.ElapsedTime;
        
        // UnityEngine API 사용
        if (Input.GetKeyDown(KeyCode.Space))
        {
            FireWeapon(currentTime);
        }
    }

    private void FireWeapon(float time)
    {
        // 복잡한 상태 의존 로직...
    }
}
```

#### 2단계: 상태 분석 및 Singleton Component 생성
```csharp
// 상태를 Singleton Component로 분리
public struct WeaponSystemState : IComponentData
{
    public float LastShotTime;
    public Unity.Mathematics.Random Random;
}

public struct WeaponCooldown : IComponentData
{
    public float CooldownTime;
    public float LastFireTime;
}
```

#### 3단계: ISystem으로 기본 변환
```csharp
[BurstCompile]
public partial struct WeaponSystemIS : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        // Singleton 엔티티 생성
        var systemStateEntity = state.EntityManager.CreateEntity();
        state.EntityManager.AddComponentData(systemStateEntity, new WeaponSystemState
        {
            LastShotTime = 0f,
            Random = new Unity.Mathematics.Random(42)
        });
    }

    public void OnUpdate(ref SystemState state)
    {
        // Singleton 상태 접근
        var weaponState = SystemAPI.GetSingletonRW<WeaponSystemState>();
        float currentTime = (float)SystemAPI.Time.ElapsedTime;

        // TODO: Input 처리는 별도 시스템으로 분리 필요
        
        // 쿼리로 쿨다운 처리
        foreach (var (cooldown, entity) in 
                 SystemAPI.Query<RefRW<WeaponCooldown>>().WithEntityAccess())
        {
            if (currentTime - cooldown.ValueRO.LastFireTime >= cooldown.ValueRO.CooldownTime)
            {
                // 발사 로직
                FireWeapon(ref weaponState.ValueRW, cooldown.ValueRW, currentTime);
            }
        }
    }

    private void FireWeapon(ref WeaponSystemState state, RefRW<WeaponCooldown> cooldown, float time)
    {
        state.LastShotTime = time;
        cooldown.ValueRW.LastFireTime = time;
        // 추가 발사 로직...
    }
}
```

### 8-2 고급 전환 패턴

#### 복잡한 상태 관리: Entity Storage Pattern
```csharp
// 복잡한 상태는 별도 Entity에 저장
[BurstCompile]
public partial struct ComplexSystemIS : ISystem
{
    private Entity systemDataEntity;

    public void OnCreate(ref SystemState state)
    {
        systemDataEntity = state.EntityManager.CreateEntity();
        
        // 복잡한 데이터 구조도 Component로 저장 가능
        state.EntityManager.AddBuffer<StateBuffer>(systemDataEntity);
        state.EntityManager.AddComponentData(systemDataEntity, new SystemConfig
        {
            MaxEntities = 1000,
            UpdateRate = 60f
        });
    }

    public void OnUpdate(ref SystemState state)
    {
        // SystemState를 통해 저장된 Entity 참조
        if (!state.EntityManager.Exists(systemDataEntity))
            return;

        var config = state.EntityManager.GetComponentData<SystemConfig>(systemDataEntity);
        var stateBuffer = state.EntityManager.GetBuffer<StateBuffer>(systemDataEntity);
        
        // 복잡한 로직 수행...
    }
}
```

#### Managed API 의존성 분리
```csharp
// 입력 처리를 별도 SystemBase로 분리
public partial class InputSystem : SystemBase
{
    protected override void OnUpdate()
    {
        // UnityEngine Input API 사용
        bool firePressed = Input.GetKeyDown(KeyCode.Space);
        
        if (firePressed)
        {
            // 입력 이벤트를 Component로 전달
            var inputEntity = GetSingletonEntity<PlayerInputSingleton>();
            EntityManager.AddComponent<FireInputEvent>(inputEntity);
        }
    }
}

// 실제 게임플레이 로직은 ISystem
[BurstCompile]
public partial struct WeaponFireSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // 입력 이벤트 체크
        foreach (var (fireEvent, entity) in 
                 SystemAPI.Query<RefRO<FireInputEvent>>().WithEntityAccess())
        {
            // 발사 로직 수행
            ProcessWeaponFire(ref state);
            
            // 이벤트 제거
            state.EntityManager.RemoveComponent<FireInputEvent>(entity);
        }
    }

    private void ProcessWeaponFire(ref SystemState state)
    {
        // Burst-compiled 발사 로직...
    }
}
```

### 8-3 전환 검증 체크리스트

#### 컴파일 및 실행 확인
- [ ] `[BurstCompile]` 어트리뷰트 추가 후 컴파일 에러 없음
- [ ] `SystemAPI` 사용으로 모든 쿼리·시간·싱글톤 접근 변환
- [ ] UnityEngine API 사용 부분을 별도 SystemBase로 분리
- [ ] 모든 상태 필드를 Singleton Component 또는 구조체 필드로 이전

#### 성능 측정 및 비교
```csharp
// 성능 측정 유틸리티
[BurstCompile]
public partial struct PerformanceMeasureSystem : ISystem
{
    private long lastMeasureTime;

    public void OnUpdate(ref SystemState state)
    {
        long startTime = System.Diagnostics.Stopwatch.GetTimestamp();
        
        // 측정하고자 하는 시스템 로직...
        
        long endTime = System.Diagnostics.Stopwatch.GetTimestamp();
        float elapsedMs = (endTime - startTime) / (float)System.Diagnostics.Stopwatch.Frequency * 1000f;
        
        // Unity Profiler에서 확인 가능
        Unity.Profiling.ProfilerUnsafeUtility.BeginSample("SystemPerformance");
        // 결과 기록...
        Unity.Profiling.ProfilerUnsafeUtility.EndSample();
    }
}
```

### 8-4 일반적인 전환 문제와 해결책

| 문제 | 원인 | 해결책 |
|------|------|--------|
| **Burst 컴파일 실패** | Managed 타입 참조 | `[BurstDiscard]` 또는 별도 SystemBase로 분리 |
| **상태 손실** | 구조체 필드 초기화 누락 | `OnCreate`에서 명시적 초기화 |
| **싱글톤 접근 실패** | 싱글톤 Entity 생성 누락 | `OnCreate`에서 싱글톤 생성 확인 |
| **성능 저하** | 과도한 구조체 복사 | `ref` 파라미터 사용 |

---

## 9. 최종 체크리스트

- [ ] Burst compile 에러 없는가? (`Jobs > Burst > Safety Checks`)  
- [ ] SystemGroup 순서가 의도대로 작동하는가? (`Dependencies` 탭)  
- [ ] GC Alloc이 감소했는가? (`Profiler > Memory`)  
- [ ] 테스트 월드에서 동일 기능을 유지하는가?  

> **결론** : **SystemBase**는 편리함, **ISystem**은 퍼포먼스.  
> 프로젝트 규모와 요구 성능에 따라 **하이브리드**로 운영하는 것이 가장 실용적인 접근입니다.
