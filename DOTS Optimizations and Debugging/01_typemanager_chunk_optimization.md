# TypeManager·Chunk 최적화 항목
### 런타임 메모리·캐시 효율을 극대화하고 디버거에서 병목을 찾아내기

---

## 1. TypeManager 최적화

| 항목 | 설명 | 최적화 팁 |
|------|------|-----------|
| **타입 등록(Initialize)** | 플레이어 실행 시 모든 `IComponentData`·`ISystem` 메타데이터를 스캔·테이블화 | • **Editor Only** 컴포넌트에 `#if UNITY_EDITOR` 사용해 빌드 시 제외<br>• AOT 플랫폼은 `StaticTypeRegistry` 활용해 리플렉션 제거 |
| **TypeIndex** | 각 컴포넌트 타입의 고유 인덱스 | • 타입 수가 많으면 Chunk 헤더 캐시 미스 증가 → _불필요한 Scriptable 컴포넌트 통합_ |
| **Field Offset / Alignment** | Unmanaged Component 내부 필드 정렬 | • `struct` 필드를 **16 byte 이하** 정렬로 압축<br>• 불필요한 `bool` 다수 대신 `byte flags` 비트필드 사용 |

> **스타트업 측정** : Profiler ▸ `TypeManager.Initialize()` 프레임 시간을 확인하고, 불필요한 컴포넌트·시스템을 제거한다.

---

## 2. Chunk 최적화

| 카테고리 | 증상 | 해결책 |
|----------|------|--------|
| **Chunk Utilization** | `Entities Debugger ▸ Chunks` 에서 **Util < 60 %** | • 동일 Archetype 엔티티 수를 늘려 **뭉치** 저장<br>• 크기가 큰 `IBufferElement` → 용량 미리 `ResizeUninitialized()` |
| **Fragmentation** | Frequent Add/Remove 로 Chunk 분할 | • `IEnableableComponent` 로 토글 ↔ StructuralChange 최소화 |
| **Component Layout** | 16 KB 초과하여 두 Chunk에 걸쳐 저장 | • 대형 버퍼/Blob 은 **BlobAssetReference** 로 외부화 |
| **Shared Component Split** | 값 변경 시 Chunk 이동 | • 자주 변하는 변수는 Unmanaged 필드로 이동 |
| **Memory Bandwidth** | Burst 루프에서 캐시 미스 | • 컴포넌트 사이즈 ≤ 64 byte 권장<br>• 불필요한 `float4x4` 대신 `float3x4` |

### 2‑1 Chunk Capacity 계산 예시
```text
ChunkSize (16 KB) / ComponentSize (Position+Velocity = 24 bytes) ≈ 682 엔티티
```
*컴포넌트가 40 byte 로 늘어나면 수용량이 409 → 40 % 감소.*

---

## 3. Editor·디버깅 도구

| 도구 | 위치 | 활용 포인트 |
|------|------|-------------|
| **Entities Hierarchy ▸ Statistics** | Window ▸ Entities ▸ Hierarchy | • World · Archetype 수 · Chunk 수 · 평균 Util 확인 |
| **Systems Window ▸ Frame Time 열** | Window ▸ Entities ▸ Systems | • 업데이트 시간 상위 시스템 식별 |
| **Chunks View** | Hierarchy 우측 ▸ Chunks 버튼 | • 각 Chunk 의 Component 크기·Util% · Version·EnableMask 확인 |
| **Query Window** | Window ▸ Entities ▸ Query | • 쿼리 결과에 포함된 Chunk / 엔티티 수 비교 |
| **Profiler ▸ Memory · Timeline** | Window ▸ Analysis ▸ Profiler | • `ArchetypeChunkData` 메모리 · SyncPoint 이벤트 탐색 |

---

## 4. 최적화 워크플로

1. **Statistics 창**으로 평균 Chunk Util < 60 % 인 Archetype 탐색  
2. `SharedComponent` 값 변경 → Chunk 분할 여부 확인 & 리팩터  
3. 버퍼 길이 급증 Chunk 는 Capacity 조절 (`ResizeUninitialized`)  
4. Profiler `Timeline` 의 **JobFence / WaitForJobGroup** 가 긴 곳은 StructuralChange 스파이크 → Enableable 또는 Deferred 방식 전환  
5. 최종적으로 **Build & Run** 후 **Player Build Report** 로 메모리 사용량 검증  

---

## 5. 체크리스트

- [ ] **TypeManager.Initialize** 시간이 10 ms 미만인가?  
- [ ] **Chunk Utilization** 이 70 % 이상인가? (대형 Prefab 제외)  
- [ ] `ISharedComponentData` 값 변경 빈도가 프레임당 1회 이하인가?  
- [ ] Empty / Fragmented Chunk 수가 과도하지 않은가?  
- [ ] Profiler 에서 Burst Job 의 캐시 미스가 많은 HotPath 찾았는가?  

> TypeManager·Chunk 를 주기적으로 점검하면 **CPU 캐시 효율** 과 **메모리 사용량** 을 동시에 개선할 수 있습니다.  
> **디버깅 도구**를 활용해 병목을 빠르게 찾고, 위 가이드라인으로 최적화하세요.
