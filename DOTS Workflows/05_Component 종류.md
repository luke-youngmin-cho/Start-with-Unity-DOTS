# 컴포넌트 타입 총정리  
### Unmanaged · Managed · Buffer · Chunk · Shared · Tag · Cleanup
---

## 1. 한눈에 보는 컴포넌트 분류표

| 분류 | 구현 인터페이스 | 메모리 위치 | Burst 지원 | 대표 용도 |
|------|----------------|-------------|-----------|-----------|
| **Unmanaged Component** | `IComponentData` *(struct)* | Chunk 내부 SoA | ✅ | 위치, 속도, 체력 등 빈번히 업데이트되는 데이터 |
| **Managed Component** | `ManagedComponent<T>` (class) | GC Heap | ❌ | UnityEngine 객체 참조, 문자열, 대용량 Asset 핸들 |
| **Dynamic Buffer** | `IBufferElementData` *(struct)* | Chunk 끝 가변 영역 | ✅ | 경로 노드, 인벤토리, 이펙트 리스트 등 가변 길이 |
| **Chunk Component** | `IComponentData` + `[ChunkSerializable]` | Chunk Header | ✅ | 한 Chunk 공통 설정값(LOD, Section ID) |
| **Shared Component** | `ISharedComponentData` *(struct/class)* | 외부 관리 테이블 | ❌ | 동일 렌더링 자료, AI 그룹 ID – 필터링 전용 |
| **Tag Component** | `IComponentData` (필드 없음) | Chunk 내부 | ✅ | 상태 플래그(“Dead”, “Selected”) |
| **Cleanup Component** | `ICleanupComponentData` *(struct)* | Chunk 내부 | ✅ | 엔티티 파괴 직후 시스템에 알림 |
| *(참고)* Enableable | `IEnableableComponent` | Chunk Enable Mask | ✅ | On/Off 토글 (Structural Change 없이) |

---

## 2. Unmanaged vs Managed

| 항목 | Unmanaged (`IComponentData`) | Managed (클래스 컴포넌트) |
|------|---------------------------|--------------------------|
| **저장 위치** | Chunk 내부 (Struct-of-Arrays) | GC Heap |
| **Burst** | 완전 지원 | 미지원 |
| **GC 비용** | 없음 | 할당·컬렉션 비용 발생 |
| **직렬화** | 빠른 바이너리 Copy | 리플렉션 기반 |
| **권장 크기** | ≤ 16 KB(권장) | 작게 유지; 다량 사용 지양 |
| **예시** | `float3 Position`, `int Health` | `Mesh mesh`, `string Name` |

> 프로젝트 규모가 커질수록 **Managed Component 최소화**가 성능과 GC 안정성에 결정적입니다.

---

## 3. Dynamic Buffer (`IBufferElementData`)

* **구조** : Chunk 뒤쪽에 가변 길이 배열이 연속 배치.  
* **Capacity 증가** 시 새 Chunk로 이동 → **Structural Change** 발생.  
* **ResizeUninitialized()** 로 용량 미리 확보하여 이동 비용 줄이기.  
* 내부 요소는 Burst 지원 **Unmanaged struct** 여야 함.

```csharp
public struct Waypoint : IBufferElementData
{
    public float3 Position;
}
// 사용
var buffer = entityManager.GetBuffer<Waypoint>(e);
buffer.Add(new Waypoint { Position = p });
```

---

## 4. Chunk Component (`IComponentData` + `[ChunkSerializable]`)

* Chunk에 **하나만 존재** – 같은 Archetype의 모든 엔티티가 공유.  
* 렌더 섹션 ID, LOD 마스크 등 **Chunk‑범위 설정**에 사용.  
* 쿼리에서 `.AddComponent<ChunkComponent>()` 로 필터 가능.

---

## 5. Shared Component (`ISharedComponentData`)

| 특징 | 세부 내용 |
|------|-----------|
| **Archetype 분할** | 값이 다른 엔티티는 별도 Chunk → 필터링 O(1) |
| **ECS Query** | `WithSharedComponentFilter()` 로 빠른 그룹 선택 |
| **값 변경 비용** | Chunk 이동(StructuralChange) + COW, 주의 |
| **Burst** | 값 자체는 Unmanaged 가능하지만 테이블 관리로 오버헤드 존재 |

```csharp
entityManager.SetSharedComponent(entity,
    new RenderMaterialID { Value = 3 });
```

---

## 6. Tag Component

* **필드 없는 struct** → 0 Byte, Enable Mask만 차지.  
* 존재 여부만으로 상태 표현.  
* `AddComponent<TagDead>(entity)` → “죽음” 상태, `RemoveComponent<TagDead>` 로 해제.

---

## 7. Cleanup Component (`ICleanupComponentData`)

* 엔티티 **Destroy** 시점에 제거되지 않고 **한 프레임 더 유지**.  
* `ICleanupComponentData` 를 쿼리하는 시스템에서 파괴 직후 정리 작업 수행 가능.  
* 자동으로 다음 프레임에 제거되어 메모리 누수 없음.

---

## 8. 설계 가이드라인

1. **빈번한 업데이트** → Unmanaged Components.  
2. **온·오프 토글** → `IEnableableComponent` 로 StructuralChange 회피.  
3. **대량 필터링 기준** → Shared Component, 단 값 변경 빈도는 최소화.  
4. **Chunk‑전역 값** → Chunk Component 로 데이터 중복 제거.  
5. **임시/파괴 후 처리** → Cleanup Component 사용.  
6. **GC 민감 로직** → Managed Component 삼가고 Asset ID(숫자)로 대체.

---

## 9. 빠른 체크리스트

- [ ] 빈번 업데이트 데이터가 GC Heap에 있지 않은가?  
- [ ] 버퍼 용량 증가로 StructuralChange 가 과도하게 발생하지 않는가?  
- [ ] SharedComponent 값 변경이 프레임마다 일어나지는 않는가?  
- [ ] `IEnableableComponent` 로 토글 가능한 필드를 굳이 Remove/Add 하고 있지 않은가?  

> 이 표를 기준으로 컴포넌트 타입을 올바르게 선택하면 **캐시 효율, GC 비용, StructuralChange 빈도**를 모두 최적화할 수 있습니다.
