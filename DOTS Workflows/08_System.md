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
| **멤버 필드** | 자유롭게 추가 (GC Heap) | 불가 |
| **저장 방식** | `this.field` | `SystemState`의 `Entity`에 **Component/Singleton** 저장 |
| **인스펙터 노출** | Entities Debugger에서 필드 확인 불가 | 동일 (필드가 없으므로) |

### 3‑1 ISystem에서 상태 저장 예시
```csharp
public struct TickCounter : IComponentData { public int Value; }

public void OnCreate(ref SystemState state)
{
    state.EntityManager.AddComponentData(state.SystemHandle,
        new TickCounter { Value = 0 });
}

public void OnUpdate(ref SystemState state)
{
    ref var counter = ref
      state.EntityManager.GetComponentDataRW<TickCounter>(state.SystemHandle).ValueRW;
    counter.Value++;
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

## 8. 마이그레이션 팁

| 단계 | 작업 |
|------|------|
| ① | `SystemBase` 코드 분석 → 외부 상태(필드) 의존 제거 |
| ② | 필드 → Singleton Component 로 이전 |
| ③ | `OnUpdate` 로직을 `ISystem` 형식으로 복사 & `SystemAPI` 활용 |
| ④ | `[BurstCompile]` 추가, Profiler 로 성능 비교 |
| ⑤ | 남은 Managed API 사용 여부 확인 후 대체 |

---

## 9. 최종 체크리스트

- [ ] Burst compile 에러 없는가? (`Jobs > Burst > Safety Checks`)  
- [ ] SystemGroup 순서가 의도대로 작동하는가? (`Dependencies` 탭)  
- [ ] GC Alloc이 감소했는가? (`Profiler > Memory`)  
- [ ] 테스트 월드에서 동일 기능을 유지하는가?  

> **결론** : **SystemBase**는 편리함, **ISystem**은 퍼포먼스.  
> 프로젝트 규모와 요구 성능에 따라 **하이브리드**로 운영하는 것이 가장 실용적인 접근입니다.
