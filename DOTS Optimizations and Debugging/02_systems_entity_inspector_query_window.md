# Systems / Entity / Component Inspector & Query Window 활용 가이드
### 런타임 데이터를 시각화하고 문제를 빠르게 진단하기

Unity Entities에는 **Systems Window**, **Entities Hierarchy/Inspector**, **Components Inspector**, **Query Window** 등  
여러 편리한 **Editor 도구**가 내장되어 있습니다. 올바르게 사용하면 성능 병목, 잘못된 데이터 상태, 의존성 오류를 빠르게 찾을 수 있습니다.

---

## 1. Systems Window

| 위치 | Window ▸ Entities ▸ **Systems** |
|------|--------------------------------|
| 주요 탭 | **Group Tree**, **Frame Time**, **Dependencies**, **Queries** |

### 1‑1 Group Tree
* 현재 World 의 **SystemGroup** 계층과 실행 순서를 트리 형태로 표시  
* **색상**  
  * **녹색** – 현재 프레임에 실행  
  * **회색** – `[DisableAutoCreation]` 또는 조건 불일치  
* **우클릭 ▸ Focus** 로 선택 시스템만 강조

### 1‑2 Frame Time 열
* 각 시스템의 **Update 시간**(ms) 과 **Self 시간** 표시  
* 정렬하여 **상위 병목** 시스템을 빠르게 식별

### 1‑3 Dependencies 탭
* 시스템 간 **UpdateBefore/After**·JobHandle 의존 그래프 시각화  
* **빨간선** : 순환 의존 경고, 잘못된 Order 설정 확인

---

## 2. Entities Hierarchy & Inspector

| 위치 | Window ▸ Entities ▸ **Hierarchy** |
|------|------------------------------------|

### 2‑1 Hierarchy 뷰 모드
| 모드 | 설명 |
|------|------|
| **GameObjects** | SubScene Open → Authoring 트리 표시 |
| **Entities** | 베이크된 실 Entity 트리 |
| **Mixed** | 양쪽 모두 표시 |

* 런타임에는 **Entities 모드**에서 **Chunk·Archetype·Entity** 레벨로 탐색 가능

### 2‑2 Inspector 패널
* 선택한 Entity·Component 의 **필드 값** 실시간 확인·수정  
* **Version**, **EnableMask**, **Chunk Index** 등 메타데이터 표시  
* **Live Conversion**: Authoring 값을 수정하면 즉시 베이크 결과 반영

> **Tip** Runtime 중 컴포넌트 값을 수정하면, 그 프레임에 바로 로직이 반영되어 디버깅이 쉬워집니다.

---

## 3. Component Inspector (Components 윈도)

| 위치 | Window ▸ Entities ▸ **Components** |
|------|-------------------------------------|
| 기능 | 전체 World 의 **컴포넌트 타입** 리스트 및 **엔티티 보유 수** 표시 |

* 특정 컴포넌트 더블클릭 → 자동으로 **Query Window** 에 필터 설정  
* **Enable‑able** 컴포넌트는 **On/Off 비율**을 막대로 표시

---

## 4. Query Window

| 위치 | Window ▸ Entities ▸ **Query** |
|------|-------------------------------|

### 4‑1 Query Builder
* **With All / With Any / With None / With Disabled** 필드로 쿼리 조건 설정  
* 즉시 결과 엔티티 수・Chunk 수 반영 → 필터 효과 확인

### 4‑2 Live Query
* 타 시스템에서 사용하는 **EntityQuery** 를 자동 추적 리스트에 표시  
* 항목 더블클릭 → 해당 시스템 하이라이트 및 Inspector 로 포커스  
* **Prefab 아이콘** : 프리팹 엔티티 식별 용이

### 4‑3 복합 쿼리 디버깅
1. Systems Window ▸ **Queries** 열에서 문제 쿼리 확인  
2. Query Window 로 열어 조건 조정  
3. 결과 Chunk 정보에서 **SharedComponent** 분할 여부, EnableMask 상태 확인

---

## 5. 통합 디버깅 워크플로

1. **FPS 하락** 감지 → **Profiler** 로 병목 프레임 선택  
2. **Systems Window** Frame Time 정렬 → 상위 시스템 확인  
3. 시스템 더블클릭 → **Dependencies 탭** 으로 Update 순환/대기 분석  
4. 해당 시스템의 쿼리를 **Query Window** 로 열어 엔티티 수・Chunk Util 확인  
5. 엔티티 상세는 **Hierarchy Inspector** 로 Component 값 검증  
6. 수정 사항 적용 후 **Profiler** 재측정

---

## 6. 베스트 프랙티스

| 상황 | 도구 | 액션 |
|------|------|------|
| Chunk Util < 50 % | Hierarchy ▸ Chunks | SharedComponent 값 최적화 |
| 시스템 Update=0 ms, Self=0 ms | Systems Window | 실제 쿼리 결과 0? → `[RequireMatchingQueries]` 추가 |
| Prefab 쿼리 누락 | Query Window | 프리팹 필터(`IsPrefab`) 확인 |
| StructuralChange Spike | Profiler Timeline → SyncPoint | 시스템 위치 확인 후 ECB 패턴 전환 |

---

## 7. 체크리스트

- [ ] Systems Window Frame Time 열로 **Top 5** 병목 시스템을 파악했는가?  
- [ ] Query Window 로 문제 쿼리를 상세 분석했는가?  
- [ ] Entities Inspector 에서 잘못된 컴포넌트 값·Enable 상태를 수정했는가?  
- [ ] Chunk View 로 Fragmentation / Utilization 을 확인했는가?  

> **Editor 디버깅 도구**를 적극 활용하면 코드 수정 없이도 문제를 시각적으로 파악하고 해결할 수 있습니다.  
> 주기적인 도구 사용 습관을 통해 **성능 최적화**와 **데이터 정확성**을 모두 확보하세요.
