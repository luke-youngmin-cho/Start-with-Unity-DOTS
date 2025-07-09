# Enableable Component 사용법
### `IEnableableComponent` 로 토글 비용 없이 상태 전환하기

> Enableable 컴포넌트는 **컴포넌트 데이터를 Chunk 밖으로 이동하지 않고** On / Off 를 전환할 수 있게 해 주는 1‑비트 마스크 기능입니다.  
> `AddComponent` / `RemoveComponent` 로 인한 **Structural Change** 를 피하면서, Tag Component 와 동일한 의미를 깔끔히 유지할 수 있습니다.

---

## 1. 개념 요약

| 항목 | 설명 |
|------|------|
| **인터페이스** | `public struct MyFlag : IEnableableComponent { public int Value; }` |
| **저장 위치** | Chunk 내부 데이터는 그대로, **Enable Mask**(비트 배열)로 상태 관리 |
| **토글 비용** | 메모리 1‑bit 수정 → StructuralChange 0 |
| **호환 API** | `SystemAPI.Query<EnabledRefRW<MyFlag>>()`, `ComponentLookup<MyFlag>().SetEnabled(entity, bool)` |
| **사용 목적** | 일시적 비활성화 / Debuff 상태 / Visibility Toggle 등 |

---

## 2. 선언 & 기본 사용

### 2‑1 컴포넌트 선언
```csharp
using Unity.Entities;

public struct Invisible : IEnableableComponent
{
    public float FadeTime;   // 비활성 상태일 때도 데이터 유지
}
```

### 2‑2 엔티티 생성 시 추가
```csharp
// 초기값과 함께 활성 상태로 추가
entityManager.AddComponentData(entity, new Invisible { FadeTime = 0 });
```

### 2‑3 활성 / 비활성 토글
```csharp
var lookup = SystemAPI.GetComponentLookup<Invisible>();
lookup.SetEnabled(entity, false);  // 비활성화
lookup.SetEnabled(entity, true);   // 재활성화
```

* **중요** : 토글 후에도 `Invisible` 데이터는 유지됩니다. StructuralChange 발생 X.

---

## 3. 쿼리 패턴

| 패턴 | 예시 코드 | 설명 |
|------|-----------|------|
| **활성 컴포넌트만** | `SystemAPI.Query<RefRW<Invisible>>()` | 기본 Query 는 **활성** 엔티티만 반환 |
| **비활성 포함 전체** | `SystemAPI.Query<EnabledRefRW<Invisible>>()` | `EnabledRefRW` / `EnabledRefRO` 사용 |
| **비활성 전용** | `.WithNone<Invisible>() .WithAllDisabled<Invisible>()` | `WithDisabled<>()` 어트리뷰트 |
| **토글‑in‑place** | `EnabledRefRW<T>.ValueRW = false;` | 잡 내부에서 바로 비트 수정 |

### 3‑1 예제: 일정 시간 후 깜빡임 토글
```csharp
partial struct BlinkJob : IJobEntity
{
    public double Time;
    void Execute(EnabledRefRW<Invisible> flag)
    {
        flag.ValueRW = (Time % 1.0) < 0.5; // 0.5초 간격 On/Off
    }
}
```

---

## 4. 잡(Job)에서의 활용

* `EnabledRefRW<T>` / `EnabledRefRO<T>` 는 Burst‑친화적이며 추가 메모리 복사 없이 **Enable Mask** 를 직접 수정·조회합니다.
* 병렬 잡에서 동시에 같은 엔티티를 토글하지 않도록 쿼리를 분할하거나 단일 쓰레드 경로로 처리하세요.

---

## 5. Tag Component vs Enableable

| 비교 | Tag Component(`IComponentData` 0‑byte) | Enableable(`IEnableableComponent`) |
|------|----------------------------------------|------------------------------------|
| **On/Off 비용** | Add/Remove → StructuralChange | 비트 수정(O(1)) |
| **데이터 필드** | 가질 수 없음 | 필드 보유 가능 |
| **메모리** | 0 byte + Chunk 헤더 | 필드 크기 + 1 bit |
| **GC 부담** | 없음 | 없음 |
| **Use case** | 드물게 상태 부여 / 제거 | 프레임마다 토글하거나 값 유지 필요 |

---

## 6. 주의사항 & 베스트 프랙티스

1. **빈번 토글** : 초당 수백 번 토글해도 StructuralChange 가 없으므로 안전하나, **동일 엔티티 다중 토글 경쟁** 상황은 피하세요.  
2. **Initial Disabled** : `AddComponent` 후 즉시 `SetEnabled(false)` 또는 `EnabledRefRW.ValueRW = false` 로 초기 비활성화 설정.  
3. **Query 성능** : `EnabledRef*` 쿼리는 약간의 추가 마스크 체크가 있지만, 일반 `Ref*` 대비 미미합니다.  
4. **Profiler 확인** : Entity Timeline → **Archetype 이동** 이벤트가 없으면 토글 성공.  
5. **Hybrid Renderer** : 렌더러 시스템은 Enable 상태를 고려하므로, Invisible 토글로 드로우콜을 즉시 줄일 수 있습니다.

---

## 7. 실습 체크리스트

- [ ] Enableable 컴포넌트 선언(`IEnableableComponent`).  
- [ ] `EnabledRefRW<T>` 패턴으로 잡에서 토글 구현.  
- [ ] Profiler 로 StructuralChange 없이 토글되는지 확인.  
- [ ] Tag Component 와 Enableable 차이를 팀원에게 설명할 수 있는가?  

> Enableable 컴포넌트를 활용하면 **게임플레이 상태 토글**이나 **임시 비활성화** 로직을 비용 없이 구현할 수 있으며, 전체 StructuralChange 횟수를 크게 줄여 퍼포먼스를 향상시킬 수 있습니다.
