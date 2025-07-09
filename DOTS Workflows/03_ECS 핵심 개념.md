# ECS 핵심 개념 – Entity / Component / System / World / Archetype

> **Entities 1.4 기준**으로 정리한 기본 용어와 구조입니다.  
> 각 개념을 정확히 이해하면 최적의 데이터 설계와 시스템 구조를 만들 수 있습니다.

---

## 1. Entity (엔티티)
| 특징 | 설명 |
|------|------|
| **ID + 태그** | 메모리상으로는 64‑bit *(index + version)* 정수. 자체 데이터는 없고, **컴포넌트의 컨테이너 역할**을 한다. |
| **경량 객체** | 생성·파괴 비용이 낮으며, **데이터와 로직을 분리**해 게임 오브젝트보다 부담이 적다. |
| **버전 관리** | 삭제 후 같은 index 재사용 시 version이 증가 → 참조 무결성 보장.|

```text
┌────────────┐
│  Entity 42 │  ← ID만 존재
└────────────┘
    ▲  ▲
    │  └──── Version (변경 시 증가)
    └─────── Index (Chunk 내 위치)
```

---

## 2. Component (컴포넌트)
| 구분 | 예시 | 메모리 특징 |
|------|------|-------------|
| **IComponentData**<br>(Unmanaged) | `LocalTransform`, `Velocity` | Struct ≤ 16 KB, Burst 친화적 |
| **IEnableableComponent** | `PhysicsCollider`(on/off) | BitMask로 Enable 상태 전환 (StructuralChange X) |
| **IBufferElementData** | `DynamicBuffer<Waypoint>` | Chunk 끝에 가변 길이 배열 |
| **ISharedComponentData** | `RenderMeshArray` 인덱스 | 동일 값 Entity끼리 Chunk 분리 (필터링 빠름) |
| **ManagedComponent** | `UnityEngine.Mesh` 참조 | GC Heap, Burst 불가 – 최소화 권장 |

> **불변 규칙**  
> *컴포넌트는 순수 데이터*만 보유하며 로직이 없어야 한다.

---

## 3. System (시스템)
| 타입 | 특징 | 사용 예 |
|------|------|---------|
| **ISystem** (struct) | Unmanaged, Burst 기본 지원, 상태 필드 적음 | 리스폰, 이동 계산 |
| **SystemBase** (class) | Managed 상태·API 풍부, OnCreate/Destroy 가능 | NetCode, 스트리밍 |
| **SystemGroup** | `UpdateInGroup`, `OrderFirst/Last` 어트리뷰트로 서브 그룹화 | `SimulationSystemGroup` |
| **BakingSystem** | 베이킹 파이프라인 전용, Editor에서만 실행 | 지오메트리 최적화 |

### 시스템 실행 순서 흐름도
```text
InitializationSystemGroup
    ├─ [..]
SimulationSystemGroup
    ├─ PhysicsSystemGroup
    │   ├─ PhysicsBuild
    │   └─ PhysicsStep
    ├─ MyGameplaySystem
    └─ EndSimulationEntityCommandBufferSystem
PresentationSystemGroup
```

---

## 4. World (월드)
| 속성 | 설명 |
|------|------|
| **엔티티·시스템 컨테이너** | 한 World 안에 독립적 EntityDB + SystemGroup 세트가 존재 |
| **다중 월드** | PlayMode 기본: `DefaultWorld` (게임), `ConversionWorld` (에디터 베이크) |
| **싱글톤 데이터** | `SystemAPI.GetSingleton<T>()` 로 전역 설정 전달 가능 |

> **Tip** 서로 다른 월드 간 엔티티 직접 이동은 불가. EntityScene 스트리밍이나 복제 필요.

---

## 5. Archetype & Chunk
### 5‑1 Archetype
* **컴포넌트 집합의 유형(signiture)**  
  동일한 컴포넌트 목록을 가진 엔티티는 같은 **Archetype** 으로 분류.
* **전용 메모리 풀** – 분류 기준이 바뀌면 Structural Change 발생.

### 5‑2 Chunk
| 항목 | 값 |
|------|----|
| **크기** | 기본 16 KB (플랫폼별 상이) |
| **정렬** | 동일 Archetype 엔티티가 **SoA** 방식으로 연속 배치 |
| **Access** | `ArchetypeChunk` 구조체로 Burst‑친화적 접근 |
| **Chunk Header** | Enable Mask, Version 등 메타데이터 포함 |

```text
┌───────────────────────────────┐  ◄ Chunk (16 KB)
│ Entity0 | Position0 | Velocity0│
│ Entity1 | Position1 | Velocity1│  ← 컴포넌트별 열(Column) 저장
│   ...   |    ...    |    ...   │
└───────────────────────────────┘
```

> **성능 원칙**  
> 1. **연속 메모리 접근** → 캐시 효율 극대화  
> 2. **Query** 는 Archetype 단위로 필터링 → 불필요한 검사 최소화

---

## 6. 핵심 정리
1. **Entity** 는 ID, **Component** 는 순수 데이터, **System** 은 로직.  
2. **World** 는 이 모든 것을 담는 컨테이너이자 시스템 실행 스케줄러.  
3. **Archetype + Chunk** 구조 덕분에 데이터가 유사한 엔티티를 묶어 CPU 캐시 히트율을 극대화한다.  
4. 성능을 위해 **불필요한 Managed Component** 를 지양하고, Query가 **Archetype 연속 메모리**를 활용하도록 설계한다.  
5. 실행 순서와 의존성을 `SystemGroup`, `UpdateBefore/After`로 명확히 정의해 **경합·Sync Point**를 최소화한다.
