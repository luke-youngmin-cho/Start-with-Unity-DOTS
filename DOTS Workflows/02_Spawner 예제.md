# ECS 워크플로 빠른 시작 — **Spawner** 튜토리얼

> 이 튜토리얼은 **엔티티 생성·컴포넌트 읽기/쓰기·런타임 인스턴스화** 과정을 통해 ECS 워크플로를 익히는 단계별 예제입니다. 전체 흐름은 아래 5가지 작업을 순서대로 수행합니다.

| 순서 | 작업 | 목표 |
|------|------|------|
| 1 | SubScene 만들기 | 예제 컨텐츠를 담을 서브씬 생성 |
| 2 | 컴포넌트 생성 | 스포너 로직이 사용할 **Spawner** 컴포넌트 정의 |
| 3 | 엔티티 생성 | Authoring + Baker로 GameObject → Entity 변환 |
| 4 | 시스템 작성 | 싱글 스레드 `SpawnerSystem` 구현 |
| 5 | 시스템 최적화 | Burst‑Job 기반 **병렬** `OptimizedSpawnerSystem` 구현 |

---

## 1. SubScene 만들기
1. Hierarchy 우클릭 → **New Sub Scene ▸ Empty Scene** 선택 후 저장.  
2. 또는 기존 GameObject를 선택하고 **New Sub Scene ▸ From Selection** 으로 변환.  
3. 빈 GameObject에 **SubScene** 컴포넌트를 붙이고 *Scene Asset* 지정 시 기존 서브씬을 참조로 추가할 수 있습니다.

> **Tip** `Auto Load Scene` 체크를 켜 두면 Play 모드에서 자동 스트리밍됩니다.

---

## 2. Spawner 컴포넌트 작성
```csharp
using Unity.Entities;
using Unity.Mathematics;

public struct Spawner : IComponentData
{
    public Entity Prefab;
    public float3 SpawnPosition;
    public float  NextSpawnTime;
    public float  SpawnRate;
}
```
* **Prefab**  : 런타임에 복제할 템플릿 엔티티  
* **SpawnPosition** : 생성 위치  
* **SpawnRate**  : 생성 주기(초)  
* **NextSpawnTime** : 다음 생성 시각(런타임에서 갱신)
---

## 3. Spawner 엔티티 생성 (Authoring + Baker)

### 3‑1 Authoring 컴포넌트
```csharp
using UnityEngine;
using Unity.Entities;

class SpawnerAuthoring : MonoBehaviour
{
    public GameObject Prefab;
    public float SpawnRate;
}
```

### 3‑2 Baker
```csharp
class SpawnerBaker : Baker<SpawnerAuthoring>
{
    public override void Bake(SpawnerAuthoring authoring)
    {
        var entity = GetEntity(TransformUsageFlags.Dynamic);
        AddComponent(entity, new Spawner
        {
            Prefab        = GetEntity(authoring.Prefab, TransformUsageFlags.Dynamic),
            SpawnPosition = authoring.transform.position,
            NextSpawnTime = 0f,
            SpawnRate     = authoring.SpawnRate
        });
    }
}
``` 

### 3‑3 에디터 설정
1. 서브씬에 빈 GameObject **Spawner** 생성.  
2. **SpawnerAuthoring** 추가 후 *Prefab*·*Spawn Rate* 설정(예: 2초).  
3. **Entities Hierarchy** 창을 *Runtime* 또는 *Mixed* 모드로 두면 베이킹된 Spawner 엔티티와 컴포넌트 값을 바로 확인할 수 있습니다.

---

## 4. Spawner 시스템 작성 (싱글 스레드)

```csharp
using Unity.Entities;
using Unity.Transforms;
using Unity.Burst;

[BurstCompile]
public partial struct SpawnerSystem : ISystem
{
    public void OnCreate (ref SystemState state) { }
    public void OnDestroy(ref SystemState state) { }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (RefRW<Spawner> spawner in SystemAPI.Query<RefRW<Spawner>>())
        {
            ProcessSpawner(ref state, spawner);
        }
    }

    void ProcessSpawner(ref SystemState state, RefRW<Spawner> spawner)
    {
        if (spawner.ValueRO.NextSpawnTime < SystemAPI.Time.ElapsedTime)
        {
            // Prefab이 유효한지 확인
            if (spawner.ValueRO.Prefab == Entity.Null)
            {
                UnityEngine.Debug.LogError("Spawner의 Prefab이 설정되지 않았습니다!");
                return;
            }

            Entity e = state.EntityManager.Instantiate(spawner.ValueRO.Prefab);
            state.EntityManager.SetComponentData(
                e,
                LocalTransform.FromPosition(spawner.ValueRO.SpawnPosition));

            spawner.ValueRW.NextSpawnTime =
                (float)SystemAPI.Time.ElapsedTime + spawner.ValueRO.SpawnRate;
        }
    }
}
```

* `RefRW<Spawner>` : 컴포넌트 **읽기/쓰기** 목적.  
* 시스템이 **싱글 스레드**에서 실행되므로 소수 엔티티 처리에 적합.

---

## 5. Spawner 시스템 최적화 (Burst + 병렬 Job)

```csharp
using Unity.Collections;
using Unity.Entities;
using Unity.Transforms;
using Unity.Burst;

[BurstCompile]
public partial struct OptimizedSpawnerSystem : ISystem
{
    public void OnCreate (ref SystemState state) { }
    public void OnDestroy(ref SystemState state) { }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = GetEntityCommandBuffer(ref state);

        new ProcessSpawnerJob
        {
            ElapsedTime = SystemAPI.Time.ElapsedTime,
            Ecb         = ecb
        }.ScheduleParallel();
    }

    EntityCommandBuffer.ParallelWriter GetEntityCommandBuffer(ref SystemState state)
    {
        // BeginSimulationEntityCommandBufferSystem을 가져와서 ECB 생성
        var ecbSystem = state.World.GetExistingSystemManaged<BeginSimulationEntityCommandBufferSystem>();
        var ecb = ecbSystem.CreateCommandBuffer(state.WorldUnmanaged);
        return ecb.AsParallelWriter();
    }
}

[BurstCompile]
public partial struct ProcessSpawnerJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter Ecb;
    public double ElapsedTime;

    void Execute([ChunkIndexInQuery] int chunkIndex, ref Spawner spawner)
    {
        if (spawner.NextSpawnTime < ElapsedTime)
        {
            // Prefab이 유효한지 확인
            if (spawner.Prefab == Entity.Null)
                return;

            Entity e = Ecb.Instantiate(chunkIndex, spawner.Prefab);
            Ecb.SetComponent(
                chunkIndex, e,
                LocalTransform.FromPosition(spawner.SpawnPosition));

            spawner.NextSpawnTime =
                (float)ElapsedTime + spawner.SpawnRate;
        }
    }
}
```

* **`IJobEntity` + `ScheduleParallel()`** : 다수 엔티티를 다중 스레드에서 처리.  
* **EntityCommandBuffer** : 메인 스레드 외부에서 생성·설정을 기록하고, 이후 재생.  
* **BurstCompile** : 네이티브 코드로 컴파일되어 CPU 성능 상승.  

> **주의** 엔티티 수가 적으면 **스케줄 오버헤드**가 성능 이득보다 클 수 있으니 Profiler로 측정 후 적용하세요.

---

## 6. 플레이 모드 확인
* Play 모드 진입 시 **Prefab**이 설정한 주기로 생성되는지 Scene / Game 뷰에서 확인.  
* `Entities Hierarchy` 창에 새 엔티티가 실시간으로 추가되는 것을 볼 수 있습니다.
* 멀티스레딩 효과를 확인하려면 Spawner 오브젝트를 여러 개로 복제해 **한 번에 많은 스포너**를 실행해 보세요.

---

### 마무리
이로써 **Spawner 예제**를 통해 SubScene → Baker → System → Job 최적화까지 ECS 기본 워크플로를 실습했습니다. 다음 단계로는 **Entities Graphics**, **Netcode for Entities** 등을 결합해 보다 복잡한 런타임 로직을 구축해 보세요.
