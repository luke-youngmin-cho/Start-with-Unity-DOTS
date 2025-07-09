# ParallelWriter · Deterministic Playback
### 다중 스레드로 안전하게 Structural Change 기록하고, 예측 가능한 재생 순서 보장하기

`EntityCommandBuffer (ECB)` 는 멀티스레드 잡(Job)에서 구조 변경을 기록할 때 **ParallelWriter** 를 제공합니다.  
여러 스레드가 동시에 ECB에 명령을 추가해도 **데이터 경쟁 없이** 처리되며,  
재생(Playback) 시 **Deterministic Order**(결정적 순서) 를 유지하여 게임 로직의 재현성을 확보합니다.

---

## 1. ParallelWriter 기본

| 항목 | 설명 |
|------|------|
| **생성** | `ecb.AsParallelWriter()` – 스레드별 **ChunkIndexInQuery** 로 슬롯 지정 |
| **스레드 안전** | 내부에 per‑thread **NativeQueue** 사용 → Lock‑Free |
| **지원 명령** | `Instantiate`, `DestroyEntity`, `Add/Remove/SetComponent`, `AddBuffer`, ... |
| **Index 파라미터** | `chunkIndex` 또는 `sortKey` 로 결정적 입력값 필요 |

### 1‑1 예시
```csharp
var ecb = _endSimEcbSystem
            .CreateCommandBuffer(state.WorldUnmanaged)
            .AsParallelWriter();

[BurstCompile]
partial struct SpawnJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;
    public Entity Prefab;

    void Execute([ChunkIndexInQuery] int ciq, in SpawnRequest req)
    {
        Entity e = Ecb.Instantiate(ciq, Prefab);
        Ecb.SetComponent(ciq, e, new Translation { Value = req.Position });
    }
}

new SpawnJob { Ecb = ecb, Prefab = prefabEntity }
    .ScheduleParallel();
```

---

## 2. Deterministic Playback 원리

| 단계 | 내용 |
|------|------|
| **기록** | `sortKey`(보통 `chunkIndex` 또는 `entityInQueryIndex`) 를 기준으로 명령을 버킷에 저장 |
| **정렬** | Playback 직전, ECB 내부에서 **sortKey ASC** 로 명령 정렬 |
| **실행** | 동일 SortKey 간 명령은 FIFO, 모든 워커 스레드 간 순서 **고정** |

> **중요** : `chunkIndex` 변수는 **쿼리 결과 순서**에 종속 → 데이터 레이아웃이 바뀌면 SortKey 도 변함.  
> 재현성을 극대화하려면 **안정적 키** (`entityInQueryIndex` 또는 커스텀 ID) 사용을 고려하세요.

### 2‑1 커스텀 sortKey 사용
```csharp
int sortKey = unchecked((int) (entity.Index & 0xFFFF_FFFF)); // Stable ID
Ecb.DestroyEntity(sortKey, entity);
```

---

## 3. 실전 사용 패턴

| 패턴 | 권장 키 | 설명 |
|------|--------|------|
| **Prefab 일괄 Instantiate** | `chunkIndex` | Chunk 단위 배치 최적 |
| **엔티티별 파괴** | `entityInQueryIndex` | 고유 ID로 결정적 순서 |
| **Hierarchy(부모‑자식) 생성** | 동일 sortKey 사용 | 부모→자식 생성 순서 유지 |
| **Event Component 추가** | `NativeThreadIndex` 조합 | 스레드 구분 + 로컬 인덱스 |

---

## 4. 디버깅 & 프로파일링

1. **Entities Timeline**  
   * `Playback (EntityCommandBuffer)` 이벤트 확장 → 명령 수·소요 시간 확인  
2. **ECB Statistics (Editor 모드)**  
   * 디버그창: **Jobs ▸ Entity Debugger ▸ ECB Stats**  
3. **Safety 체크**  
   * `ENABLE_UNITY_COLLECTIONS_CHECKS` 빌드에서 **Write/Read 충돌** 오류 탐지

---

## 5. 주의사항 & 베스트 프랙티스

| 주제 | 가이드 |
|------|--------|
| **BurstCompile** | Job 구조체에 `[BurstCompile]` 추가 – ECB 호출 자체도 Burst 지원 |
| **Capacity 예측** | 대량 명령 예상 시 `CreateCommandBuffer` 의 `capacity` 인자 사용 |
| **Playback 시점** | Simulation 그룹s 끝(`EndSimulationECBSystem`) → 시스템 의존성 보장 |
| **Nested ECB** | 한 프레임에 여러 ECB Playback 가능 – 순서 고려 (`OrderBefore/After`) |
| **Cost** | 정렬·메모리 복사 비용 있으나, SyncPoint 1회로 대부분 상쇄 |

---

## 6. 체크리스트

- [ ] ParallelWriter 호출에 **안정적 sortKey** 를 사용했는가?  
- [ ] 동일 엔티티에 대해 **중복 명령**(예: Add & Remove 동일 컴포넌트) 작성 X?  
- [ ] Playback 그룹 위치가 의도대로 설정되었는가? (`UpdateInGroup`, `OrderAfter`)  
- [ ] Profiler 로 ECB **Sort + Playback** 시간이 병목인지 확인?  
- [ ] Capacity 설정으로 **리사이즈 스파이크**를 방지했는가?  

> **ParallelWriter + Deterministic Playback** 을 올바르게 사용하면  
> 다중 코어에서 구조 변경을 병렬로 기록하면서도, **순서의 일관성**과 **성능** 두 가지를 모두 충족할 수 있습니다.
