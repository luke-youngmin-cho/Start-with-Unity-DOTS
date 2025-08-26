# ECS 워크플로 빠른 시작  
### SubScene 생성 → Baker 변환 흐름

---

## 1. 개요
* **SubScene** 은 GameObject 기반 “작성(Authoring) 데이터”를 ECS 데이터로 변환(bake)하기 위한 컨테이너입니다.  
* **Baker** 는 MonoBehaviour(Authoring Component)의 값을 읽어 **Entity**와 **ECS Component**를 생성·부착합니다.  
* 워크플로 핵심 단계:  
  1. SubScene 안에 GameObject 배치  
  2. SubScene 닫힘 → 자동 베이킹  
  3. Baker 코드가 Entities / Components 생성  
  4. (선택) Baking System이 후처리  
  5. 결과가 Entity Scene 파일에 저장 또는 라이브 월드에 반영  

---

## 2. SubScene 만들기

| 시나리오 | 절차 |
|----------|------|
| **새 SubScene 추가** | Hierarchy 창 우클릭 → **New Sub Scene ▸ Empty Scene** 선택 후 저장 |
| **기존 GameObject로 SubScene 생성** | Hierarchy에서 대상 GameObject 선택 → 우클릭 → **New Sub Scene ▸ From Selection** |
| **기존 SubScene 참조 추가** | 빈 GameObject 생성 → **SubScene** 컴포넌트 추가 → **Scene Asset** 지정 |

> **Tip** SubScene 컴포넌트의 **Auto Load Scene** 체크박스를 켜면 플레이라임에 자동 스트리밍됩니다.

### 열기 / 닫기
* **Open** 상태  
  * Hierarchy에 Authoring GameObject 트리 표시  
  * Scene View Mode(Entities / GameObjects)에 따라 런타임 데이터 확인  
  * 초기 베이킹 후 **변경 시마다 Incremental Bake**  
* **Closed** 상태  
  * 베이크된 Entity Scene이 스트리밍되어 Hierarchy에 접힘  
  * Play 모드 진입 시 몇 프레임 후 엔티티 이용 가능  
  * Open 상태와 달리 스트리밍 지연이 있음

---

## 3. 베이킹 프로세스 이해

```text
Authoring GameObject
        │  (SubScene 닫힘)
        ▼
┌─────────────┐
│  Entity 생성 │  (메타데이터만)
└─────────────┘
        ▼
┌─────────────┐
│   Baker 단계 │  (MonoBehaviour → ECS Component)
└─────────────┘
        ▼
┌─────────────┐
│ Baking Systems│ (선택적 후처리)
└─────────────┘
        ▼
Entity Scene 저장 / 라이브 월드 반영
```

* **Incremental Bake**  
  * 변경된 GameObject 및 그 의존 대상만 재베이크 → 실시간 미리보기  
  * 결과 일관성을 보장하려면 Baker/Baking System이 **상태를 캐싱 X**, 항상 입력만 통해 결과 계산
* **Full Bake**  
  * 씬 전체를 처음부터 변환 (디스크 라운드트립)

---

## 4. Authoring Component & Baker 작성

### 4‑1 기본 예제

```csharp
// MonoBehaviour Authoring
public class RotationSpeedAuthoring : MonoBehaviour
{
    public float DegreesPerSecond;
}

// ECS Component
public struct RotationSpeed : IComponentData
{
    public float RadiansPerSecond;
}

// Additional Component 정의
public struct AdditionalEntity : IComponentData
{
    public int SomeValue;
}

// Baker
public class SimpleBaker : Baker<RotationSpeedAuthoring>
{
    public override void Bake(RotationSpeedAuthoring authoring)
    {
        // Authoring GameObject → Entity
        var entity = GetEntity(TransformUsageFlags.Dynamic);

        // 1) 주 Entity에 컴포넌트 추가
        AddComponent(entity, new RotationSpeed {
            RadiansPerSecond = math.radians(authoring.DegreesPerSecond)
        });

        // 2) 추가 Entity 생성 & 컴포넌트 부착 (선택)
        var child = CreateAdditionalEntity(TransformUsageFlags.Dynamic, "Child");
        AddComponent(child, new AdditionalEntity { SomeValue = 123 });
    }
}
```

### 4‑2 핵심 규칙
| 항목 | 설명 |
|------|------|
| **상태 보존 금지** | Baker 인스턴스는 여러 Authoring Component를 반복 호출하므로 **멤버 변수에 캐시 X** |
| **자기 Entity만 수정** | 다른 Baker가 만든 Entity/Component 변경 금지 |
| **의존성 선언** | 외부 참조 데이터(다른 GameObject, Asset 등)는 `DependsOn()` 또는 Baker 전용 `GetComponent<T>()` 호출로 **추적** |
| **정적 / 순서 불보장** | Baker 실행 순서는 정의되지 않으므로 **상호 의존 로직을 작성하면 안 됨** |

#### 의존성 예시 (요약)
```csharp
public class DependentDataAuthoring : MonoBehaviour
{
    public GameObject Other;
    public Mesh Mesh;
}

public class GetComponentBaker : Baker<DependentDataAuthoring>
{
    public override void Bake(DependentDataAuthoring authoring)
    {
        DependsOn(authoring.Other);
        DependsOn(authoring.Mesh);

        if (authoring.Other == null || authoring.Mesh == null) return;

        var entity = GetEntity(TransformUsageFlags.Dynamic);
        // ...
    }
}
```

---

## 5. 실습 점검 목록

1. **SubScene 생성** 후 GameObject 배치  
2. SubScene 닫아 베이크 결과(Entity Scene) 확인  
3. Authoring Component + Baker 스크립트 작성 → 스크립트 적용  
4. **Entities Hierarchy** 창에서 Entity 및 Baking Preview 확인  
5. 값 수정 → **Incremental Bake** 동작 확인  
6. 플레이 모드에서 엔티티 존재 및 동작 검사  

---

## 6. 자주 발생하는 문제 & 해결

| 증상 | 원인 / 해결 |
|------|-------------|
| Baker가 다시 실행되지 않음 | 외부 데이터 접근 시 `DependsOn()` 누락 |
| 플레이 모드 전환 후 값 초기화 | SubScene Open 상태에서 변경 → 실시간 적용, Closed 일 때는 초기 몇 프레임 대기 |
| 컴포넌트가 중복 생성 | Baker가 여러 번 `AddComponent` 호출 – 조건문 확인 |
| 성능 저하 | SubScene 하나에 데이터 과다 → 여러 SubScene으로 분할 |

---

### 끝!  
이 가이드를 따라 **SubScene → Baker → Entity** 변환 흐름을 체험해 보세요. 작은 예제를 완성한 뒤, 컴포넌트·시스템 확장 및 Scene Streaming, Baking System까지 단계적으로 적용하면 실전 프로젝트에 바로 활용할 수 있습니다.
