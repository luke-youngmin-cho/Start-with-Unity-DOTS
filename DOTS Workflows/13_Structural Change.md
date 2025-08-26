# 구조 변경 & EntityCommandBuffer  
### EntityManager **직접 변경** vs **ECB 지연 변경** 패턴 비교

Structural Change(구조 변경)은 엔티티의 Archetype 을 바꾸므로 **Chunk 재배치 + Sync Point** 비용이 발생합니다.  
`EntityManager` 즉시 호출과 **EntityCommandBuffer(ECB)** 지연 변경의 차이를 이해하여, 필요한 비용만 지불하세요.

---

## 1. 즉시 변경: `EntityManager` 직접 호출

| 장점 | 단점 |
|------|------|
| 구현 간단, 코드량 적음 | 호출 시점에 **Sync Point** – 메인 스레드 대기 |
| 상태 디버깅 쉬움 | Worker Job 내부에서 호출 불가 |
| `Add/RemoveComponent`, `Instantiate`, `DestroyEntity` 모두 지원 | 프레임 중 연속 호출 시 성능 하락 |

### 1‑1 예시
```csharp
// 싱글 스레드, 즉시 파괴
if (hp.ValueRO.Value <= 0)
{
    state.EntityManager.DestroyEntity(entity);
}
```
*Profiler* 에서 **WaitForJobGroupID → StructuralChange** 이벤트가 길어질 수 있음.

---

## 2. 지연 변경: **EntityCommandBuffer (ECB)**

| 장점 | 단점 |
|------|------|
| Job(멀티스레드) 내부에서 안전하게 StructuralChange 기록 | Playback 시 **Sync Point** 발생 (전역 한 번) |
| 대량 변경을 한 번에 처리 → **캐시 Miss 최소화** | 코드 약간 복잡, ECB 시스템 추가 필요 |
| `AsParallelWriter()` 로 스레드 간 경쟁 없이 기록 | 실행(Playback) 순서 고려 필요 |

### 2‑1 기본 패턴
```csharp
// ① System에서 ECB 생성
var ecb = _endSimEcb.CreateCommandBuffer(state.WorldUnmanaged)
                    .AsParallelWriter();

// ② Job에서 StructuralChange 기록
[BurstCompile]
partial struct DestroyJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;

    void Execute([ChunkIndexInQuery] int chunkIndex,
                 Entity entity, in Health hp)
    {
        if (hp.Value <= 0)
            Ecb.DestroyEntity(chunkIndex, entity);
    }
}

new DestroyJob { Ecb = ecb }.ScheduleParallel();
```
* `Begin/EndSimulationEntityCommandBufferSystem` 을 이용해 **Group 끝** 에서 Playback.

### 2‑2 Playback 과정
1. 시스템 그룹 끝에서 `Playback()` 호출  
2. ECB 안 명령이 **EntityManager API** 로 실행  
3. Sync Point 1회 → 모든 쿼리/잡 재작성  

---

## 3. 두 방식 성능 비교

| 시나리오 | EntityManager 직접 | ECB 지연 |
|----------|------------------|----------|
| **≤ 100 StructuralChange / 프레임** | 간단·차이 미미 | 오버헤드 (ECB 생성·Playback) |
| **수천 개 파괴 / 생성** | SyncPoint 가 길어져 프레임 스파이크 발생 | 한 번의 SyncPoint 로 처리 → 안정적 |
| **멀티스레드 Job 내부** | 호출 불가 (Safety 오류) | `ParallelWriter` 로 안전 기록 |
| **디버깅** | Step 단위 확인 용이 | Playback 타이밍 고려 필요 |

---

## 4. 권장 베스트 프랙티스

1. **대량·병렬** StructuralChange → **ECB**  
2. **소규모·드문** 변경 & 메인 스레드 로직 → `EntityManager` 직접  
3. **Playback 타이밍**  
   * 상태 리스너 시스템(`ICleanupComponent`) → **EndSimulationECB** 앞/뒤 위치 조정  
4. **ParallelWriter 올바른 Index** (`chunkIndex` / `entityInQueryIndex`) 사용  
5. 동일 시스템에서 **직접 변경 + ECB 혼용** 시 SyncPoint 2회 발생 가능, 가급적 분리

---

## 5. 실전 체크리스트

- [ ] Job 내부에서 EntityManager 호출이 남아 있지 않은가? → ECB 로 이전  
- [ ] Playback 그룹 위치가 실제 의존성 순서와 맞는가? (`OrderBefore/After`)  
- [ ] Profiler **Entity Timeline → SyncPoint** 이벤트 1프레임당 1회 이하?  
- [ ] ECB 용량(NativeList) 확장으로 GC 또는 Realloc Spike 없는가?  
- [ ] **Burst Safety Checks** 활성 상태에서 오류 발생 여부?

> **요약**  
> *적은 양은 직방, 많은 양은 ECB.*  
> 패턴을 맞추면 **프레임 스파이크** 없이 안정적으로 구조 변경을 처리할 수 있습니다.
