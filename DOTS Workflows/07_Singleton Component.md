# Singleton Component 패턴
### 전역 게임 상태를 ECS 방식으로 안전하게 다루기

> **Singleton Component**는 특정 **월드(World)** 안에 **정확히 한 개만 존재**하는 컴포넌트를 말합니다.  
> 전통적으로 `static` 변수를 사용하던 설정·점수·난수 시드 등을 **Data‑Oriented 방식**으로 통합·직렬화·테스트하기 좋게 만들 수 있습니다.

---

## 1. 기본 개념

| 항목 | 설명 |
|------|------|
| **유일성** | `EntityQueryOptions.IncludeDisabledEntities` + **assert** 로 한 개만 유지 |
| **저장 위치** | 일반 Unmanaged Component → Chunk 내부 |
| **접근 방법** | `SystemAPI.GetSingleton<T>()`, `SystemAPI.SetSingleton<T>()` |
| **생성 시점** | `SystemBase.OnCreate()` 또는 베이킹(Baker)에서 Entity를 만들어 추가 |
| **장점** | 간단한 전역 상태, DOTS 직렬화, 잡(Job) 안전 접근 |
| **주의** | 싱글톤 **엔티티 파괴 시 런타임 오류** 발생 위험 → 게임 도중 Remove 금지 |

---

## 2. 선언 & 초기화

### 2‑1 데이터 구조
```csharp
public struct GameSettings : IComponentData
{
    public int   MaxLives;
    public float GlobalGravity;
}
```

### 2‑2 엔티티 생성 (System)
```csharp
public partial class BootstrapSettingsSystem : SystemBase
{
    protected override void OnCreate()
    {
        if (!SystemAPI.HasSingleton<GameSettings>())
        {
            Entity settings = EntityManager.CreateEntity();
            EntityManager.AddComponentData(settings, new GameSettings
            {
                MaxLives       = 3,
                GlobalGravity  = -9.81f
            });
        }
    }
    protected override void OnUpdate() { }
}
```

> **Tip** 여러 월드가 존재할 경우(예: 서버/클라이언트) 각 월드마다 별도 싱글톤을 생성합니다.

---

## 3. 읽기 / 쓰기 패턴

### 3‑1 읽기 전용
```csharp
public partial struct ApplyGravityJob : IJobEntity
{
    [ReadOnly] public GameSettings Settings;

    void Execute(ref Velocity vel)
    {
        vel.Value.y += Settings.GlobalGravity * SystemAPI.Time.DeltaTime;
    }
}

// 시스템
var job = new ApplyGravityJob
{
    Settings = SystemAPI.GetSingleton<GameSettings>()
}.ScheduleParallel();
```

### 3‑2 쓰기
```csharp
public partial struct DebugKeyInputSystem : ISystem
{
    void OnUpdate(ref SystemState state)
    {
        if (Input.GetKeyDown(KeyCode.L))
        {
            var settings = SystemAPI.GetSingleton<GameSettings>();
            settings.MaxLives++;
            SystemAPI.SetSingleton(settings);
        }
    }
}
```

* **주의** 잡(Job) 내부에서 `SetSingleton` 호출 불가 → 메인 스레드 시스템에서 처리.

---

## 4. 복수 필드 vs 복수 싱글톤

| 상황 | 권장 방안 |
|------|-----------|
| 값 5개 이하 & 업데이트 빈도 낮음 | **단일 싱글톤**(구조체 필드)로 묶기 |
| 필드가 10개 이상 & 변경 주기 다양 | **도메인별 싱글톤 분리** (예: `GameConfig`, `AudioSettings`) |
| 대용량 배열 / 리스트 | 별도 **BlobAsset** 또는 DynamicBuffer 사용 |

---

## 5. 테스트 & 리팩터링

1. **World 단위 테스트**  
   * 테스트 월드를 만들어 `SystemAPI.SetSingleton` 으로 환경 구성.  
2. **데이터‑드리븐 설계**  
   * ScriptableObject → Baker 로 싱글톤 초기 값 전환.  
3. **런타임 가변성**  
   * 네트워크 동기화 필요 시 NetCode **Ghost Component** 로 등록.

---

## 6. 체크리스트

- [ ] `SystemAPI.HasSingleton<T>()` 로 중복 생성 방지  
- [ ] 잡 내부에선 `GetSingletonRO<T>()` 만 사용 (읽기 전용)  
- [ ] 싱글톤 Entity 파괴 금지 — 대신 필드를 덮어써서 초기화  
- [ ] 월드마다 독립 싱글톤 유지(멀티 월드 구조)  
- [ ] 값 자주 바뀌면 **IEnableableComponent** · Buffer · Event 로 대체 고려  

> Singleton Component 패턴을 올바르게 적용하면 **Data‑Oriented 전역 상태**를 안전하게 공유하면서도, `static` 코드 의존을 줄여 테스트와 멀티스레딩 호환성을 확보할 수 있습니다.
