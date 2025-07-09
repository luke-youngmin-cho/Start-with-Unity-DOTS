# System Group & Update Order 속성
### 시스템 실행 순서를 명확하게 제어하기

Unity ECS는 **System Group**과 다양한 **Update Order 어트리뷰트**를 통해 시스템 실행 순서를 결정합니다.  
적절히 사용하면 종속성 오류·프레임 지연 없이 예측 가능한 동작을 확보할 수 있습니다.

---

## 1. 기본 System Group 계층

| 그룹 | 역할 | 기본 포함 서브그룹 |
|------|------|-------------------|
| **InitializationSystemGroup** | 씬 로딩, 싱글톤 초기화 | `BeginInitializationECBSystem` |
| **SimulationSystemGroup** | 게임 로직, 물리, AI | `FixedStepSimulationSystemGroup`<br>`PhysicsSystemGroup`<br>`EndSimulationECBSystem` |
| **PresentationSystemGroup** | 렌더 데이터 준비 | `LateSimulationSystemGroup` |
| **FixedStepSimulationSystemGroup** | 일정 간격(기본 60Hz) 고정 업데이트 | `BeginFixedStepECBSystem` / `EndFixedStepECBSystem` |
| **TransformSystemGroup** *(Entities Graphics)* | LocalTransform 계산 | 내부 전용 |

> **World Bootstrap** 시 `DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups()` 가 위 그룹들을 자동 생성하고 각 시스템을 분류합니다.

---

## 2. Update Order 어트리뷰트

| 어트리뷰트 | 대상 | 기능 |
|------------|------|------|
| `[UpdateInGroup(typeof(G))]` | **System / Group** | 해당 그룹 `G` 안으로 시스템을 배치 |
| `[UpdateBefore(typeof(S))]` | System | 같은 그룹 내에서 **S** 이전 실행 |
| `[UpdateAfter(typeof(S))]` | System | 같은 그룹 내에서 **S** 이후 실행 |
| `[OrderFirst]` | System | 그룹 맨 앞(가장 먼저) |
| `[OrderLast]` | System | 그룹 맨 뒤(가장 마지막) |

### 우선순위 결정 규칙
1. `OrderFirst` > `UpdateBefore/After` > `OrderLast` > 등록 순서  
2. 충돌(사이클) 발생 시 Unity가 경고 로그를 출력하고 **등록 순서** 기준으로 정렬

---

## 3. 코드 예시

### 3‑1 그룹 정의 & 삽입
```csharp
// 커스텀 그룹 – 모든 레이캐스트 처리용
[UpdateInGroup(typeof(SimulationSystemGroup))]
[OrderFirst]                         // Simulation 그룹에서 가장 먼저
public partial class RaycastGroup : ComponentSystemGroup { }
```

### 3‑2 시스템 배치
```csharp
[UpdateInGroup(typeof(RaycastGroup))]
public partial struct CollectRaycastRequestsSystem : ISystem { /* ... */ }

[UpdateInGroup(typeof(RaycastGroup))]
[UpdateAfter(typeof(CollectRaycastRequestsSystem))]
public partial struct ExecuteRaycastsSystem : ISystem { /* ... */ }

[UpdateInGroup(typeof(RaycastGroup))]
[UpdateAfter(typeof(ExecuteRaycastsSystem))]
public partial struct ApplyRaycastResultsSystem : ISystem { /* ... */ }
```

### 3‑3 OrderFirst / OrderLast
```csharp
[UpdateInGroup(typeof(SimulationSystemGroup))]
[OrderLast]                         // 프레임 끝에서 버프 효과 제거
public partial struct ExpireBuffSystem : ISystem { /* ... */ }
```

---

## 4. FixedStepSimulation 사용법

```csharp
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
public partial struct PhysicsStepSystem : ISystem
{
    void OnCreate(ref SystemState s)
    {
        // 커스텀 Fixed Timestep (예: 0.02f → 50Hz)
        s.WorldUnmanaged.ResolveSystemStateRef<FixedStepSimulationSystemGroup>()
          .Timestep = 0.02f;
    }
}
```

* **FixedStepSimulationSystemGroup** 은 누적된 `TimeAccumulator` 가 Timestep 보다 클 때 여러 번 업데이트할 수 있으므로, **FixedDeltaTime** 기준 로직(물리/네트워크 틱)에 적합합니다.

---

## 5. Systems Window & Dependencies 탭

1. **Entities Debugger (Windows ▸ Entities ▸ Systems Window)** 열기  
2. **Group Tree** : 현재 월드의 그룹·시스템 구조 시각화  
3. **Dependencies 탭** : `UpdateBefore/After` 관계, Job 의존 그래프를 그래픽으로 표시  
4. 충돌·사이클이 빨간색으로 경고되면 어트리뷰트 재조정

> **팁** 실행 중 그룹 이름 옆의 **Frame Time** 열을 확인해 병목 시스템을 파악할 수 있습니다.

---

## 6. 베스트 프랙티스

| 상황 | 권장 전략 |
|------|-----------|
| 복잡한 하위 로직 묶기 | **커스텀 그룹** 정의 후 하위 시스템을 UpdateAfter 체인으로 연결 |
| 물리 → 게임 로직 순서 | `PhysicsSystemGroup` 이후 **GameplaySystemGroup** 을 UpdateAfter 로 배치 |
| 렌더 전처리 | Presentation 그룹 앞단(`OrderFirst`)에 렌더 준비 시스템 배치 |
| 순환 의존 문제 | 시스템을 분할하거나 `OrderFirst/Last` 로 우선순위 강제 |
| 멀티 월드(NetCode) | `ServerSimulationSystemGroup`, `ClientSimulationSystemGroup` 별도 관리 |

---

## 7. 체크리스트

- [ ] 모든 시스템이 **정확한 그룹**에 속해 있는가? (`UpdateInGroup`)  
- [ ] `UpdateBefore/After`로 **사이클**이 발생하지 않는가? (Systems Window 확인)  
- [ ] 프레임 의존성이 중요한 로직은 `FixedStepSimulationSystemGroup` 을 사용했는가?  
- [ ] 병목 시스템을 `BurstCompile` 또는 Job화 후, 순서를 재조정했는가?  

> **정확한 그룹화와 명시적 실행 순서**는 ECS 프로젝트의 **디버깅·성능·협업** 품질을 크게 향상시킵니다. 위 가이드를 토대로 자신의 시스템 그래프를 정리해 보세요.
