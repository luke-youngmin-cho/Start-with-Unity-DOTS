# Job System 소개 · 의존성 · 스케줄링 오버헤드
### Entities 1.4에서 고성능 멀티스레딩 달성하기

Unity의 **Job System**은 멀티코어 CPU를 활용하여 ECS 데이터를 병렬로 처리합니다.  
잡(Job) 간 **데이터 의존성**을 자동 추적하여 안전성을 확보하며, 스케줄링 오버헤드를 최소화하는 것이 핵심입니다.

---

## 1. Job System 개요

| 요소 | 설명 |
|------|------|
| **Job** | `IJob`, `IJobFor`, `IJobParallelFor`, `IJobEntity`, `IJobChunk` 등 인터페이스 구현체 |
| **Scheduler** | Unity JobWorker 스레드 풀이 잡 큐를 가져와 실행 |
| **Burst** | C++ 수준 최적화 네이티브 코드로 컴파일 |
| **Dependency** | 잡 간 Read/Write 충돌을 추적하는 **JobHandle** 그래프 |
| **Complete()** | JobHandle 완료 대기 (메인 스레드 블로킹) |

---

## 2. 잡 유형 비교

| 인터페이스 | 대상 | 특징 |
|------------|------|------|
| `IJob` | 단일 작업 | 간단/짧은 로직 |
| `IJobParallelFor` | 인덱스 배열 | 정적 WorkStealing 분할 |
| `IJobEntity` | ECS Entity | 소스 제너레이터가 `IJobChunk` 코드 생성 |
| `IJobChunk` | Chunk 단위 | 로우레벨·최고 성능 (수작업 루프) |
| `IJobForEach<T>` *(Obsolete)* | 엔티티 | `IJobEntity`로 교체 권장 |

---

## 3. 의존성 관리

### 3‑1 자동 추적
* `SystemAPI.Query<RefRO<A>, RefRW<B>>()` → 읽기/쓰기 플래그로 의존성 계산  
* 동일 컴포넌트를 **동시에 쓰기** 요청하면 잡이 **직렬화**된다.

### 3‑2 JobHandle 체인
```csharp
JobHandle h1 = job1.Schedule();
JobHandle h2 = job2.Schedule(h1);           // 의존성 연결
state.Dependency = h2;                      // 시스템 전역 Dependency
```

### 3‑3 `CombineDependencies`
```csharp
state.Dependency = JobHandle.CombineDependencies(handleA, handleB);
```
* 여러 작업을 동시에 기다려야 할 때 사용.

### 3‑4 완전 대기 (`Complete()`)
```csharp
state.Dependency.Complete();  // 다음 코드에서 결과 필요 시
```
* **주의** : 메인 스레드 블로킹 → 빈번 사용 시 프레임 드랍.

---

## 4. 스케줄링 오버헤드

| 상황 | 오버헤드 원인 | 해결책 |
|------|--------------|--------|
| **잡 당 엔티티 수가 적음** | 스케줄 + 컨텍스트 스위치 비용이 작업 시간보다 큼 | `Run()` 또는 싱글 스레드 `foreach` 전환 |
| **빈 잡** | 쿼리 결과 0 → 스케줄 불필요 | `[RequireMatchingQueriesForUpdate]` |
| **과도한 Complete** | 메인 스레드 대기 | 의존성 연결로 `Complete()` 최소화 |
| **잡 내 Burst off** | 관리 코드 호출 → JobWorker 스레드 작업 늘어짐 | 구조체 전용 데이터, Burst 호환 API 사용 |
| **Chunk 크기 분할 미세** | `ScheduleParallel` 분할(Stride) 미조정 | `state.GetProcessedChunkCountWithoutFilter()` 로 측정 후 Tuning |

### 4‑1 오버헤드 측정 지표
* **Profiler ▸ Timeline ▸ Worker Threads** – **JobFence** (대기) / **JobFlush** (Submit) 이벤트 확인  
* **System Inspector ▸ Frame Time** – `Schedule + WaitForJobGroupID` 구간

---

## 5. 베스트 프랙티스

1. **엔티티 ≥ 500** 또는 계산 무거움 → **`ScheduleParallel()`**  
2. **엔티티 < 200** → **`Run()`** 또는 `foreach`  
3. **잡 캡처 최소화** : `NativeArray`, `ComponentLookup`, `float`값 등 **struct field** 전달  
4. **BurstCompile** 와 `Unity.Mathematics` 사용 → CPU SIMD 활용  
5. **EntityCommandBuffer.ParallelWriter** 로 StructuralChange 기록  
6. **Iteration Stride** : `IJobChunk` – `batchCount` 튜닝(4~8 Chunks)  
7. **`state.CompleteDependency()`** 호출은 **프레임 경계에 1회** 이내로 유지

---

## 6. 실전 체크리스트

- [ ] `ScheduleParallel` 호출 전에 `Dependency` 체인을 정확히 설정했는가?  
- [ ] `Complete()` 호출을 꼭 필요한 곳(데이터 접근 직전)에만 배치했는가?  
- [ ] Profiler 에서 **MainThread WaitForJobGroup** 시간이 과도하지 않은가?  
- [ ] Burst Off 경로(관리 코드)가 Job Worker 스레드 안에 없는가?  
- [ ] 쿼리 결과 0개라도 매 프레임 잡을 스케줄하고 있지 않은가? -> `RequireMatchingQueries`  

> Job System의 이점을 최대한 누리려면 **스케줄링 이득 > 오버헤드** 조건을 충족해야 합니다.  
> 엔티티 수·작업 복잡도를 계측하여 **Run ↔ Schedule ↔ ScheduleParallel** 전략을 적절히 선택하세요.
