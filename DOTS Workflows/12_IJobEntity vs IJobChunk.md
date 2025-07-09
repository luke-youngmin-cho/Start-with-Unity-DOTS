# IJobEntity vs IJobChunk 패턴 비교
### 엔티티 레이어 · Chunk 레이어 선택 기준

Entities 1.4에서는 **`IJobEntity`**(엔티티 단위) 와 **`IJobChunk`**(Chunk 단위) 두 가지 저수준 Job API를 제공합니다.  
목적·엔티티 수·캐시 패턴에 따라 올바른 패턴을 선택하여 **최고 성능**을 달성하세요.

---

## 1. 핵심 차이점

| 항목 | **IJobEntity** | **IJobChunk** |
|------|---------------|---------------|
| **루프 단위** | *Entity* (컴포넌트 튜플) | *Chunk* (Archetype‑별 16 KB 배치) |
| **코드 생성** | 소스 제너레이터가 자동 `IJobChunk`로 변환 | 직접 Chunk 루프 작성 |
| **버스트 최적화** | 자동 | 수동 (더 세밀한 제어) |
| **개발 난이도** | 간단 (Execute = per‑entity) | 복잡 (Chunk iteration, 인덱스 계산) |
| **캐시 친화성** | 좋음 | 최상 (Chunk‑내 선형 접근) |
| **플레그??** | Query 필터 자동 처리 | 수동 필터/EnableMask 적용 필요 |
| **권장 상황** | 일반 게임 로직, 90% 케이스 | 극한 성능, 커스텀 메모리 패턴, PhysX Step 등 |

---

## 2. 코드 비교

### 2‑1 IJobEntity 예시
```csharp
[BurstCompile]
public partial struct DamageJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref Health hp, in DamagePerSecond dps)
    {
        hp.Value -= dps.Value * DeltaTime;
    }
}

// 시스템
new DamageJob{DeltaTime = SystemAPI.Time.DeltaTime}
    .ScheduleParallel();
```

### 2‑2 IJobChunk 예시
```csharp
[BurstCompile]
public struct DamageChunkJob : IJobChunk
{
    public float DeltaTime;
    public ComponentTypeHandle<Health> HealthHandle;
    [ReadOnly] public ComponentTypeHandle<DamagePerSecond> DpsHandle;

    public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
    {
        var hpArray  = chunk.GetNativeArray(HealthHandle);
        var dpsArray = chunk.GetNativeArray(DpsHandle);

        for (int i = 0; i < chunk.Count; i++)
        {
            hpArray[i] = new Health
            {
                Value = hpArray[i].Value - dpsArray[i].Value * DeltaTime
            };
        }
    }
}

// 시스템 Prepare & Schedule
var job = new DamageChunkJob
{
    DeltaTime     = SystemAPI.Time.DeltaTime,
    HealthHandle  = state.GetComponentTypeHandle<Health>(false),
    DpsHandle     = state.GetComponentTypeHandle<DamagePerSecond>(true)
};
state.Dependency = job.ScheduleParallel(
    state.GetEntityQuery(typeof(Health), typeof(DamagePerSecond)),
    state.Dependency);
```

> **포인트**  
> *IJobChunk*는 **TypeHandle 캐싱** 과 **NativeArray 직접 인덱스**로 최저 오버헤드 루프를 구현합니다.

---

## 3. 퍼포먼스 비교

| 테스트 조건 | IJobEntity | IJobChunk |
|-------------|-----------|-----------|
| **100K 엔티티** | 1.0× | ~1.05× (약간 빠름) |
| **1M 엔티티** | 1.0× | ~1.20× (캐시 선형 접근 효과) |
| **빈번 StructuralChange** | 동일 | 동일 (외부 ECB) |
| **Burst OFF 경로** | 비슷 | 수동 코드 → 실수 위험 |

> **Tip** : 10만 미만 엔티티에서는 차이가 미미. **대규모** 또는 **특수 루프 패턴** 에서 IJobChunk 이득이 크다.

---

## 4. 장단점 요약

| | 장점 | 단점 |
|---|-----|-----|
| **IJobEntity** | 구현 간단·안전, Query 필터 자동, 독립 변수 전달 용이 | 소스 제너레이터 추가 빌드 시간, 미세 튜닝 제한 |
| **IJobChunk** | 최대 성능, 캐시 패턴 완전 제어, TypeHandle 재사용 최적화 | 코드 복잡, 버그 위험, 필터/EnableMask 직접 관리 필요 |

---

## 5. 선택 가이드

1. **엔티티 ≤ 200k** : `IJobEntity` + `ScheduleParallel()` 로 충분.  
2. **엔티티 ≥ 500k** 또는 **물리・랜더링 핵심 루프** : `IJobChunk` 고려.  
3. 프로파일러에서 **Worker Thread Time > Schedule/Sync Overhead** 확인 후 전환.  
4. **TypeHandle 캐싱** : 시스템 필드(`ComponentTypeHandle<T>`)로 저장→OnUpdate마다 `Update(ref state)` 호출.  
5. IJobChunk 사용 시 **BurstSafetyChecks** 로 인덱스/경계 오류를 조기 발견.

---

## 6. 체크리스트

- [ ] IJobChunk 코드에 **EnableMask** 고려 루프가 포함됐는가?  
- [ ] `ArchetypeChunk` NativeArray 접근 시 **ReadOnly** 정확히 지정?  
- [ ] 한 시스템에서 **IJobEntity ↔ IJobChunk** 혼용 시 Dependency 연결 명확?  
- [ ] 성능 차이가 측정으로 검증됐는가? (Profiler ▸ Timeline)  

> IJobEntity로 시작해, 프로파일링 후 병목 구간만 IJobChunk로 리팩터링하는 **하이브리드 전략**이 실무에서 가장 효율적입니다.
