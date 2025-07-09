# Deferred Entity · Placeholder 개념
### 베이크·런타임에서 **실제 엔티티가 아직 존재하지 않을 때** 안전하게 참조하기

Entities 워크플로에서는 **EntityCommandBuffer(ECB)** 또는 **Baker** 가  
`Instantiate()`, `CreateAdditionalEntity()`, `GetEntity(prefab)` 등을 호출할 때  
> “**엔티티를 지금 바로 만들지 않고, 나중에 한꺼번에 생성**”  
라는 **지연(Deferred) 방식**을 사용합니다.  
이때 반환되는 값이 바로 **Deferred Entity / Placeholder(자리표시자)** 입니다.

---

## 1. 왜 필요한가?

| 상황 | 문제 | Deferred Entity 해결 |
|------|------|---------------------|
| **Job Thread 안**에서 다량 Instantiate | 구조 변경을 즉시 할 수 없음 | Placeholder ID만 반환, 나중에 Sync Point 1회 |
| **Baker** 가 여러 추가 엔티티를 만들며 서로 참조 | 실제 Entity가 아직 없음 | 동일 ECB/Baker 내 Placeholder 로 선 연결 |
| **Prefab 인스턴스화** 후 컴포넌트 추가 | Instantiate 직후 EntityHandle 필요 | Placeholder 로 ID 확보 후 `AddComponent()` 기록 |

---

## 2. 동작 메커니즘

```text
1) Instantiate() 호출
   └─ Entity {index = -1, version = uniqueID}  ◄ Placeholder 반환
2) 같은 ECB 내 명령들은 Placeholder 사용
3) Playback 시 실 엔티티 할당
   └─ Placeholder → Real Entity 로 치환
```

* **Index = -1** 등 음수 값 → 아직 Chunk에 없다는 표시  
* **Version** 필드에 고유 키로 매칭  
* `EntityCommandBufferPlayback` 과정에서 모든 Placeholder를 실제 Entity로 리맵

---

## 3. 코드 예시

### 3‑1 Job 내부에서 부모‑자식 엔티티 생성
```csharp
[BurstCompile]
partial struct SpawnHierarchyJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;
    public Entity Prefab;

    void Execute([ChunkIndexInQuery] int ciq)
    {
        // ① 부모 프리팹 인스턴스화 → Placeholder p
        Entity parent = Ecb.Instantiate(ciq, Prefab);

        // ② 자식 추가
        Entity child  = Ecb.CreateEntity(ciq); // 새 Placeholder
        Ecb.AddComponent(ciq, child, new Parent { Value = parent });
        Ecb.AddComponent(ciq, child, new LocalTransform { /* ... */ });
    }
}
```
*Playback* 시 `Parent.Value` 가 실제 부모 Entity로 자동 교체됩니다.

### 3‑2 Baker 에서 추가 엔티티 연결
```csharp
public class SpawnerBaker : Baker<SpawnerAuthoring>
{
    public override void Bake(SpawnerAuthoring authoring)
    {
        var spawner = GetEntity(TransformUsageFlags.Dynamic);

        // 추가 엔티티
        var counter = CreateAdditionalEntity(TransformUsageFlags.None, "Counter");
        AddComponent(counter, new LinkedEntityGroup { Value = spawner });
        // Placeholder끼리의 참조 → Bake 완료 후 실 Entity 리맵
    }
}
```

---

## 4. 제약 사항

| 항목 | 내용 |
|------|------|
| **컴포넌트 조회 불가** | Placeholder 엔티티에 `GetComponentData<T>()` 호출하면 오류 |
| **Prefab 내부 링크** | 동일 ECB/Baker 내부에서만 Placeholder 해석, 외부 공유 X |
| **Runtime 접근** | Playback 완료 전까지 시스템에서 사용할 수 없음 (`SystemAPI.Exists()` false) |
| **Version 충돌** | Placeholder Version 충돌 시 런타임 Exception → 가능한 단일 ECB 사용 |

---

## 5. 베스트 프랙티스

1. **동일 작업 단위**(잡·베이커) 내에서 모든 관련 엔티티를 생성 & 링크.  
2. Placeholder 엔티티에는 **데이터 작성만** – 읽기는 Playback 후 다른 시스템에서.  
3. 여러 System이 같은 ECB를 공유할 필요가 있으면  
   **`Begin/EndSimulationECBSystem`** 위치를 명확히 조정(`UpdateBefore/After`).  
4. **Profiler ▸ Entity Timeline** 에서 “CreateDeferredEntity” 이벤트로 갯수·위치 확인.  
5. Placeholder 만으로는 **Hierarchy Transform** 계산이 안 되므로,  
   Playback 이후 Frame 에 로컬/월드 변환 업데이트 확인.

---

## 6. 실전 체크리스트

- [ ] Placeholder Entity 에 `GetComponentData` 접근 코드가 없는가?  
- [ ] 여러 ECB 가 같은 Placeholder 를 중복 참조하지 않는가?  
- [ ] Playback 전/후 시점 구분이 명확한가? (디버그 로그 타이밍)  
- [ ] `Entity.Null` 과 Placeholder 를 혼동하지 않았는가?  
- [ ] 프로파일러에서 SyncPoint 횟수 & 지속 시간을 모니터링했는가?  

> Deferred Entity / Placeholder 패턴은 **대량 인스턴스화**와 **복잡한 엔티티 관계 구축**을  
> 싱글 SyncPoint 로 처리하게 해 주는 핵심 기술입니다.  
> 올바르게 활용하면 구조 변경 비용을 최소화하면서, 안정적인 의존성·참조 설정을 보장할 수 있습니다.
