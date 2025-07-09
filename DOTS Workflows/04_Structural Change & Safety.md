# Structural Change & Safety

> **Structural Change**란 엔티티의 **Archetype(컴포넌트 구성)** 을 바꾸는 모든 작업을 의미합니다.  
> 컴포넌트 추가·제거, 엔티티 생성·파괴, `AddBuffer`, `SetSharedComponent` 등이 여기에 포함됩니다.  
> 이러한 변경은 **Chunk 재배치**를 일으키므로 신중한 사용이 필요합니다.

---

## 1. Structural Change가 발생하는 상황

| 작업 | 예시 메서드 | 설명 |
|------|------------|------|
| **엔티티 생성 / 파괴** | `EntityManager.Instantiate`, `EntityManager.DestroyEntity` | Chunk에 새 슬롯을 만들거나 제거 |
| **컴포넌트 추가 / 제거** | `AddComponent`, `RemoveComponent`, `SetSharedComponent` | Archetype이 달라지면 Chunk 이동 |
| **Buffer 동적 증가** | `DynamicBuffer<T>.Add()` ‑ 용량 초과 시 Chunk 재배치 |
| **SharedComponent 값 변경** | `SetSharedComponent` | 값이 달라지면 다른 Chunk로 이동 |
| **EntityCommandBuffer.Playback** | ECB에 기록된 Structural Change 모두 |

> **주의** Enable/Disable 전환(`IEnableableComponent`)은 **StructuralChange가 아닙니다**. 비트마스크만 수정하므로 비용이 매우 낮습니다.

---

## 2. Sync Point & 의존성

* Structural Change 는 모든 **읽기/쓰기 잡(Job)** 과 충돌할 수 있어 **Sync Point**(메인 스레드 일시 정지)를 유발합니다.  
* `EntityManager` API를 **잡 외부**(메인 스레드)에서 호출하면 즉시 Sync Point.  
* 잡 내부에서 Structural Change를 기록하려면 **EntityCommandBuffer (ECB)** 사용 후 **Playback** 시 Sync Point 발생.

---

## 3. 안전하게 Structural Change 다루기

### 3‑1 EntityCommandBuffer 사용 패턴
```csharp
[BurstCompile]
public partial struct SpawnJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;
    void Execute([ChunkIndexInQuery] int chunkIndex, ref Spawner spawner)
    {
        Entity e = Ecb.Instantiate(chunkIndex, spawner.Prefab);
        Ecb.AddComponent(chunkIndex, e, new Velocity { Value = 1 });
    }
}

// 시스템 본문
var ecb = _ecbSystem.CreateCommandBuffer(state.WorldUnmanaged)
                    .AsParallelWriter();
new SpawnJob { Ecb = ecb }.ScheduleParallel();
```
* **기록 단계**: 잡 내에서 `ParallelWriter` 로 StructuralChange 기록  
* **재생 단계**: 시스템 그룹 끝 (`EndSimulationEntityCommandBufferSystem`) 에서 `Playback`

### 3‑2 StructuralChange 차단 매크로
```csharp
// OnUpdate 첫 줄
state.CompleteDependency();  // 필요 시
state.EntityManager.CompleteEntityQueries(); // Editor 검증 용도
```
* 개발 중에는 `ENABLE_UNITY_COLLECTIONS_CHECKS` 조건부로 StructuralChange 위치를 검증할 수 있습니다.

### 3‑3 대량 변경 최적화
| 전략 | 설명 |
|------|------|
| **Batch Instantiate** | `EntityCommandBuffer.Instantiate` 의 `NativeArray<Entity>` 오버로드 사용 |
| **SharedComponent 최소화** | 값 변경 시 Chunk 분리 → 가능한 Unmanaged 필드로 대체 |
| **Enable/Disable 활용** | 일시적 비활성화는 `IEnableableComponent` 로 처리해 StructuralChange 회피 |
| **Prefab + LinkedEntityGroup** | 서브 엔티티 일괄 복제·파괴로 Change 횟수 축소 |

---

## 4. Safety(안전 체계)

| 안전 메커니즘 | 목적 |
|---------------|------|
| **Versioning** | Component write 버전을 기록 → 잘못된 읽기 감지 |
| **Dependency Tracking** | 잡 간 읽기/쓰기 충돌 자동 해결(완료·대기) |
| **Atomic Safety Handle** | 런타임에 컬렉션 오용(쓰기 후 읽기 등) 감지 |
| **Enable Unity Collections Checks** | Editor·Development 빌드에서 인덱스·Bounds 오류 검출 |
| **Burst Safety** | 컴파일 단계에서 잘못된 포인터·별칭 여부 분석 |

### 4‑1 대표 오류 메시지
| 메시지 | 원인 / 해결 |
|--------|-------------|
| `Entity has already been destroyed` | StructuralChange 후 오래된 Entity 값 사용 → **`entity.IsAlive`** 확인 |
| `InvalidOperationException: The previously scheduled job XYZ writes to ...` | 잡 의존성 미선언 → `ScheduleParallel` 대신 `Schedule` 또는 `Dependency` 연결 |
| `ArgumentException: A component with type XXX has not been added` | StructuralChange 중 AddComponent 누락, `GetComponent<T>` 호출 순서 오류 |

---

## 5. 실전 체크리스트

1. **메인 스레드에서 반복 호출**되는 `EntityManager` StructuralChange 코드를 **ECB** 로 이전.  
2. 잡 내부에선 **ParallelWriter** 로 기록하고, `ScheduleParallel()` 사용.  
3. **Enable/Disable**, **Buffer.ResizeUninitialized** 로 StructuralChange 최소화.  
4. Profiler 의 **Entity Timeline → Sync Point** 이벤트 확인하여 병목 제거.  
5. **Collections Checks** 를 켠 Development 빌드로 런타임 검증 → 성능 빌드에서는 끔.

---

## 6. 요약
* StructuralChange 는 **Archetype 이동** 때문에 비용이 크고 Sync Point 를 야기한다.  
* ECB 패턴, Enableable 컴포넌트, Prefab 복제 등으로 **발생 빈도와 범위**를 줄이는 것이 핵심.  
* Unity의 **안전 체계**(버전·의존성·Atomic Safety) 를 활용하면 데이터 경쟁·크래시 없이 멀티스레딩 이점을 극대화할 수 있다.
