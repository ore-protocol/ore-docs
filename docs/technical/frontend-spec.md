# Project ORE - Frontend System Specification v2.0

_Unity AR 클라이언트 아키텍처 및 AI-Native 구현 가이드_

## Executive Summary

### 프로젝트 컨텍스트

Project ORE의 Unity 클라이언트는 AR Foundation 기반의 위치 기반 P2E 게임입니다. **서버 권위적(Server-Authoritative)** 설계로 모든 게임 로직은 서버에서 검증하며, 클라이언트는 뷰어와 입력 수집 역할만 담당합니다.

### 핵심 설계 철학

```yaml
"Display Only, Verify Everything on Server"

원칙:
  1. Zero Trust Client: 클라이언트 데이터 절대 신뢰 안함
  2. Network Required: 오프라인 플레이 완전 차단
  3. AI-Friendly Components: 자동 생성 가능한 구조
  4. Performance First: 60 FPS, 500MB RAM, 10%/hr 배터리
```

### 기술 전략

```yaml
개발 방식:
  - Unity 6.0 LTS (2025년 9월 기준 최신 안정버전)
  - AR Foundation 6.0+
  - UI Toolkit (uGUI 대체)
  - Custom Networking (WebSocket + REST)

플랫폼 지원:
  - iOS 14+ (ARKit 5.0)
  - Android 10+ (ARCore 1.30+)
  - 최소 RAM: 4GB
  - 최소 저장공간: 500MB

성능 목표:
  - FPS: 60 (고사양), 30 (저사양)
  - 메모리: < 500MB
  - 배터리: < 10%/시간
  - 앱 크기: < 200MB
  - 시작 시간: < 3초
```

## 1. System Architecture

### 1.1 High-Level Architecture

**설계 근거:**
Unity의 전통적인 MonoBehaviour 아키텍처의 강결합 문제를 해결하기 위해 계층화된 아키텍처를 채택했습니다. 각 계층은 명확한 책임을 가지며, AI가 독립적으로 각 계층을 구현할 수 있습니다.

```
┌──────────────────────────────────────────────────┐
│                  Unity Client                    │
├──────────────────────────────────────────────────┤
│                 Presentation Layer               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │    UI    │ │    AR    │ │  Audio   │          │
│  │ Toolkit  │ │Foundation│ │  System  │          │
│  └──────────┘ └──────────┘ └──────────┘          │
├──────────────────────────────────────────────────┤
│                Application Layer                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │   Game   │ │   Map    │ │  Social  │          │
│  │ Systems  │ │ Systems  │ │ Systems  │          │
│  └──────────┘ └──────────┘ └──────────┘          │
├──────────────────────────────────────────────────┤
│                  Core Layer                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │  State   │ │ Network  │ │  Asset   │          │
│  │  Manager │ │  Manager │ │ Manager  │          │
│  └──────────┘ └──────────┘ └──────────┘          │
└──────────────────────────────────────────────────┘
```

**계층별 책임:**

- **Presentation**: 사용자에게 보이는 모든 것 (UI, AR 뷰, 오디오)
- **Application**: 게임 로직의 클라이언트 측 표현 (서버 데이터 시각화)
- **Core**: 기반 시스템 (상태 관리, 네트워킹, 에셋 로딩)

### 1.2 Scene Management & Flow

**설계 근거:**
씬을 Persistent와 Loadable로 분리하여 매니저들이 씬 전환 시에도 유지되도록 합니다. 이는 네트워크 연결이 끊어지지 않고 게임 상태가 보존됨을 보장합니다.

```yaml
Scene Structure:
  Persistent Scene (절대 언로드 안됨):
    - NetworkManager # 네트워크 연결 유지
    - GameStateManager # 게임 상태 보존
    - AudioManager # 오디오 연속성
    - AnalyticsManager # 분석 데이터 수집

  Loadable Scenes (필요시 로드/언로드):
    - SplashScene # 초기 로딩 (3초 목표)
    - LoginScene # 인증 (JWT 토큰)
    - MainMenuScene # 메인 메뉴
    - GameScene # AR 게임플레이
    - MapScene # 2D 맵 뷰 (AR 폴백)

Scene Flow: Splash → Login → MainMenu ↔ Game/Map
  ↔
  Settings/Profile/Shop

전환 규칙:
  - Splash → Login: 자동 (리소스 로드 완료 시)
  - Login → MainMenu: 인증 성공 시
  - MainMenu ↔ Game: 사용자 선택
  - Game ↔ Map: AR 가용성에 따라 자동/수동
```

### 1.3 Component Architecture

**설계 근거:**
Unity의 GameObject 기반 아키텍처에서 발생하는 의존성 문제를 해결하기 위해 Service Locator 패턴을 채택했습니다. 이는 AI가 각 서비스를 독립적으로 구현할 수 있게 하며, 테스트 시 Mock 객체로 쉽게 교체 가능합니다.

**핵심 원칙:**

- Singleton은 최소화 (GameManager, NetworkManager 등 필수만)
- Interface 기반 설계로 구현체 교체 용이
- DontDestroyOnLoad는 Persistent Scene에서만 사용

**주의사항:**

- 순환 참조 방지: Service는 다른 Service를 생성자에서 참조 금지
- 초기화 순서: NetworkManager → StateManager → GameManager

```csharp
// Core Component Structure
namespace ORE.Core
{
    // Singleton 구현 (최소한으로 사용)
    public class SingletonBehaviour<T> : MonoBehaviour where T : MonoBehaviour
    {
        private static T instance;

        public static T Instance
        {
            get
            {
                if (instance == null)
                {
                    Debug.LogError($"{typeof(T)} is not initialized!");
                }
                return instance;
            }
        }

        protected virtual void Awake()
        {
            if (instance != null && instance != this)
            {
                Destroy(gameObject);
                return;
            }

            instance = this as T;

            // Persistent Scene의 객체만 DontDestroyOnLoad
            if (gameObject.scene.name == "PersistentScene")
            {
                DontDestroyOnLoad(gameObject);
            }
        }
    }

    // Service Locator Pattern (의존성 주입)
    public static class Services
    {
        // 초기화 순서 중요!
        public static INetworkService Network { get; set; }     // 1st
        public static ILocationService Location { get; set; }    // 2nd
        public static IARService AR { get; set; }               // 3rd
        public static IStateService State { get; set; }         // 4th

        // 서비스 초기화 검증
        public static bool ValidateServices()
        {
            return Network != null &&
                   Location != null &&
                   AR != null &&
                   State != null;
        }
    }
}
```

### 1.4 State Management Pattern

**설계 근거:**
클라이언트는 서버로부터 받은 상태만을 표시하며, 절대 로컬에서 게임 상태를 변경하지 않습니다. 모든 액션은 서버에 Command로 전송되고, 서버의 응답으로만 상태가 업데이트됩니다.

**Optimistic UI 전략:**

- 즉각적인 피드백을 위해 UI만 먼저 업데이트
- 서버 응답이 실패하면 롤백
- 실제 게임 상태는 서버 응답 후에만 변경

```csharp
// Client State (Display Only)
public class ClientGameState
{
    // 서버에서 받은 데이터만 저장 (읽기 전용)
    public PlayerData LocalPlayer { get; private set; }
    public List<CoinData> NearbyCoins { get; private set; }
    public List<PlayerData> NearbyPlayers { get; private set; }

    // 상태 버전 관리 (서버와 동기화 검증)
    public uint StateVersion { get; private set; }

    // State는 서버 이벤트로만 업데이트
    public void UpdateFromServer(ServerStateUpdate update)
    {
        // 버전 체크 (out of order 방지)
        if (update.Version <= StateVersion)
        {
            Debug.LogWarning($"Received outdated state: {update.Version} <= {StateVersion}");
            return;
        }

        // 검증 없이 서버 데이터 적용 (서버가 권위)
        LocalPlayer = update.PlayerData;
        NearbyCoins = update.Coins;
        NearbyPlayers = update.Players;
        StateVersion = update.Version;

        OnStateUpdated?.Invoke();
    }
}

// Command Pattern for Server Requests
public abstract class GameCommand
{
    public string Id { get; } = Guid.NewGuid().ToString();
    public float Timestamp { get; } = Time.realtimeSinceStartup;

    // Optimistic UI를 위한 예상 결과
    public abstract void PredictResult(ClientGameState state);

    // 서버 거부 시 롤백
    public abstract void Rollback(ClientGameState state);

    // 서버 승인 시 확정
    public abstract void Confirm(ClientGameState state, ServerResponse response);
}
```

## 2. AR Systems

### 2.1 AR Foundation Setup

**설계 근거:**
AR Foundation 6.0+을 사용하여 ARKit/ARCore의 차이를 추상화합니다. 필수 기능만 사용하여 디바이스 호환성을 최대화하고, 선택적 기능은 폴백을 구현합니다.

**성능 최적화 전략:**

- 평면 감지는 수평면만 (수직면 제외로 CPU 절약)
- Light Estimation은 Basic (Directional 제외로 GPU 절약)
- People Occlusion은 iOS A12+ 칩셋만 활성화

```yaml
AR Foundation 6.0+ Configuration:

  AR Session (필수):
    - Auto-install AR software: true
    - Request camera permission: 앱 시작 시
    - Track session state: 상태 변화 모니터링
    - Match Frame Rate: 60fps (고사양) / 30fps (저사양)

  AR Session Origin (필수):
    - Camera setup:
        * Target Frame Rate: 디바이스 적응형
        * Facing Direction: Rear (후면 카메라)
        * Auto Focus: Continuous
    - Plane detection:
        * Mode: Horizontal (수평면만)
        * 이유: 수직면 감지 제외로 CPU 20% 절약
    - Light estimation:
        * Mode: Basic (밝기만)
        * 이유: Directional light는 GPU 부담 높음

  선택적 기능 (디바이스별):
    - People Occlusion:
        * iOS: A12 Bionic 이상만 활성화
        * Android: 지원 안함 (폴백 처리)
    - Motion Capture:
        * 향후 아바타 기능용 예약
        * 현재는 비활성화 (배터리 절약)
```

### 2.2 AR Content Pipeline

**설계 근거:**
LOD(Level of Detail) 시스템으로 거리별 품질을 조정하여 성능을 최적화합니다. 오브젝트 풀링으로 GC(Garbage Collection)를 최소화하고, 60 FPS를 유지합니다.

**LOD 거리 설정 근거:**

- 10m: 최고 품질 (디테일 중요)
- 25m: 중간 품질 (실루엣 유지)
- 50m: 낮은 품질 (위치 표시만)
- 100m: 빌보드 (2D 스프라이트)

```csharp
public class ARContentManager : MonoBehaviour
{
    [Header("LOD Settings")]
    // 거리별 LOD 레벨 (미터)
    public float[] lodDistances = { 10f, 25f, 50f, 100f };
    // LOD별 폴리곤 감소율
    public float[] lodPolygonRatio = { 1.0f, 0.5f, 0.25f, 0.1f };

    [Header("Performance Settings")]
    public int maxARObjectsVisible = 50;  // 동시 표시 최대 개수
    public float cullingDistance = 150f;  // 컬링 거리

    private Dictionary<string, ObjectPool> contentPools;
    private List<ARContent> activeContents = new List<ARContent>();

    // AR 콘텐츠 스폰 (성능 최적화)
    public GameObject SpawnARContent(ARContentData data, Vector3 worldPos)
    {
        // 최대 개수 체크 (성능 보호)
        if (activeContents.Count >= maxARObjectsVisible)
        {
            RemoveFarthestContent();
        }

        // GPS → Unity 좌표 변환
        Vector3 localPos = GPSToLocal(data.Location);

        // 거리 기반 LOD 결정
        float distance = Vector3.Distance(Camera.main.transform.position, localPos);
        int lodLevel = CalculateLOD(distance);

        // 컬링 체크
        if (distance > cullingDistance)
        {
            return null; // 너무 멀면 생성 안함
        }

        // 풀에서 가져오기 (GC 방지)
        var content = contentPools[data.Type].Get(lodLevel);
        content.transform.position = localPos;

        // AR 설정 적용
        ApplyARSettings(content, data, lodLevel);

        // 활성 목록에 추가
        activeContents.Add(content.GetComponent<ARContent>());

        return content;
    }

    private void ApplyARSettings(GameObject obj, ARContentData data, int lodLevel)
    {
        // 거리별 스케일 조정 (가시성 향상)
        float distanceScale = GetDistanceScale(obj.transform.position);
        obj.transform.localScale = data.BaseScale * distanceScale;

        // LOD별 렌더링 설정
        var renderer = obj.GetComponent<Renderer>();
        if (lodLevel >= 3) // 100m 이상
        {
            // 빌보드 모드 (항상 카메라 향함)
            var billboard = obj.GetComponent<Billboard>();
            billboard.enabled = true;

            // 그림자 비활성화 (성능)
            renderer.shadowCastingMode = UnityEngine.Rendering.ShadowCastingMode.Off;
        }
        else
        {
            // 일반 3D 렌더링
            renderer.shadowCastingMode = lodLevel < 2 ?
                UnityEngine.Rendering.ShadowCastingMode.On :
                UnityEngine.Rendering.ShadowCastingMode.Off;
        }

        // Occlusion 설정 (iOS A12+ only)
        if (ARSession.state == ARSessionState.SessionTracking &&
            SystemInfo.deviceModel.Contains("iPhone") &&
            GetIOSChipset() >= 12)
        {
            renderer.renderingLayerMask |= (uint)ARRenderLayers.PersonOcclusion;
        }
    }

    // 가장 먼 콘텐츠 제거 (성능 유지)
    private void RemoveFarthestContent()
    {
        if (activeContents.Count == 0) return;

        var farthest = activeContents
            .OrderByDescending(c => Vector3.Distance(
                Camera.main.transform.position,
                c.transform.position))
            .First();

        contentPools[farthest.Type].Return(farthest.gameObject);
        activeContents.Remove(farthest);
    }
}
```

### 2.3 Tracking & Localization

**트래킹 품질 관리 전략:**

이 시스템은 3단계 폴백을 구현합니다:

1. **Normal**: AR 트래킹 정상 → 모든 기능 활성
2. **Limited**: 트래킹 불안정 → AR 콘텐츠 일시정지, 복구 시도
3. **Unavailable**: AR 불가 → 자동으로 2D 맵 모드 전환

**복구 시도 로직:**

- 2초간 트래킹 재시도 (사용자가 환경 개선할 시간)
- 실패 시 맵 모드로 전환 (게임 지속성 보장)
- 백그라운드에서 계속 AR 가능 여부 체크

```csharp
public class ARTrackingManager : MonoBehaviour
{
    [Header("Tracking Quality Thresholds")]
    private const float MinTrackingQuality = 0.7f;  // 최소 품질
    private const float RecoveryTime = 2.0f;        // 복구 대기 시간
    private const int MaxRecoveryAttempts = 3;      // 최대 복구 시도

    public enum TrackingState
    {
        NotAvailable,   // AR 지원 안함
        Limited,        // 품질 낮음
        Normal,         // 정상
        Recovery        // 복구 중
    }

    private TrackingState currentState;
    private int recoveryAttempts = 0;

    void Update()
    {
        // AR 세션 상태 모니터링
        var sessionState = ARSession.state;

        switch (sessionState)
        {
            case ARSessionState.SessionTracking:
                if (currentState != TrackingState.Normal)
                {
                    OnTrackingRestored();
                }
                HandleNormalTracking();
                break;

            case ARSessionState.SessionInitializing:
                ShowTrackingInitUI("AR을 초기화하는 중...");
                break;

            case ARSessionState.None:
            case ARSessionState.Unsupported:
                HandleARUnavailable();
                break;

            default:
                HandleLimitedTracking();
                break;
        }
    }

    void HandleLimitedTracking()
    {
        if (currentState == TrackingState.Limited) return;

        currentState = TrackingState.Limited;
        recoveryAttempts++;

        if (recoveryAttempts < MaxRecoveryAttempts)
        {
            StartCoroutine(AttemptTrackingRecovery());
        }
        else
        {
            // 복구 실패 → 맵 모드
            ForceMapMode("AR 트래킹이 불안정합니다. 맵 모드로 전환합니다.");
        }
    }

    IEnumerator AttemptTrackingRecovery()
    {
        currentState = TrackingState.Recovery;

        // AR 콘텐츠 일시정지 (깜빡임 방지)
        PauseARContent();

        // 복구 UI 표시
        ShowRecoveryUI($"AR 복구 시도 중... ({recoveryAttempts}/{MaxRecoveryAttempts})");

        // 대기 (환경 개선 시간)
        yield return new WaitForSeconds(RecoveryTime);

        // 트래킹 상태 재확인
        if (ARSession.state == ARSessionState.SessionTracking)
        {
            // 복구 성공
            currentState = TrackingState.Normal;
            recoveryAttempts = 0;
            ResumeARContent();
            HideRecoveryUI();
        }
        else
        {
            // 재시도 또는 포기
            HandleLimitedTracking();
        }
    }

    void HandleARUnavailable()
    {
        if (currentState == TrackingState.NotAvailable) return;

        currentState = TrackingState.NotAvailable;

        // 즉시 맵 모드 전환
        ShowARUnavailableUI("이 기기는 AR을 지원하지 않습니다.");
        ForceMapMode();

        // AR 관련 리소스 해제 (메모리 절약)
        ReleaseARResources();
    }

    void OnTrackingRestored()
    {
        currentState = TrackingState.Normal;
        recoveryAttempts = 0;

        // AR 모드 재활성화 옵션 제공
        UIManager.ShowNotification("AR 사용이 가능합니다. AR 모드로 전환하시겠습니까?",
            onYes: () => SwitchToARMode(),
            onNo: null);
    }
}
```

### 2.4 AR Interaction System

**설계 근거:**
AR 환경에서는 깊이 인식이 어려우므로, 거리 기반 검증과 시각적 피드백을 강화합니다. 모든 상호작용은 서버 검증을 거쳐야 하며, 클라이언트는 시각적 피드백만 제공합니다.

**상호작용 우선순위:**

1. AR 평면 히트 테스트 (실제 표면)
2. 3D 콜라이더 레이캐스트 (가상 객체)
3. 거리 기반 선택 (폴백)

```csharp
public class ARInteractionManager : MonoBehaviour
{
    [Header("Interaction Settings")]
    public float interactionRange = 10f;        // 상호작용 최대 거리
    public float nearRange = 3f;                // 근거리 (자동 수집)
    public LayerMask interactableLayers;        // 상호작용 가능 레이어
    public float hapticIntensity = 0.5f;        // 햅틱 피드백 강도

    private Camera arCamera;
    private ARRaycastManager raycastManager;
    private GameObject highlightedObject;       // 현재 하이라이트된 객체

    void Update()
    {
        // 터치 입력 처리
        if (Input.touchCount > 0)
        {
            Touch touch = Input.GetTouch(0);

            switch (touch.phase)
            {
                case TouchPhase.Began:
                    HandleTouchBegin(touch.position);
                    break;
                case TouchPhase.Moved:
                    HandleTouchMove(touch.position);
                    break;
                case TouchPhase.Ended:
                    HandleTouchEnd(touch.position);
                    break;
            }
        }

        // 근거리 자동 수집 체크
        CheckNearbyAutoCollect();
    }

    void HandleTouchBegin(Vector2 screenPos)
    {
        // 1. AR 평면 히트 테스트
        List<ARRaycastHit> arHits = new List<ARRaycastHit>();
        if (raycastManager.Raycast(screenPos, arHits, TrackableType.PlaneWithinPolygon))
        {
            var hitPoint = arHits[0].pose.position;

            // AR 평면 위 가장 가까운 객체 찾기
            var nearestObject = FindNearestARObject(hitPoint, 2f);
            if (nearestObject != null)
            {
                HighlightObject(nearestObject);
                PlayHapticFeedback(hapticIntensity * 0.5f);
            }
        }

        // 2. 직접 레이캐스트 (AR 히트 실패 시)
        Ray ray = arCamera.ScreenPointToRay(screenPos);
        if (Physics.Raycast(ray, out RaycastHit hit, 100f, interactableLayers))
        {
            if (IsInRange(hit.point))
            {
                HighlightObject(hit.collider.gameObject);
                PlayHapticFeedback(hapticIntensity);
            }
            else
            {
                ShowDistanceWarning(hit.point);
            }
        }
    }

    void HandleTouchEnd(Vector2 screenPos)
    {
        if (highlightedObject == null) return;

        var interactable = highlightedObject.GetComponent<IARInteractable>();
        if (interactable != null)
        {
            // 서버에 상호작용 요청
            SendInteractionCommand(interactable);

            // Optimistic UI (즉시 피드백)
            PlayCollectAnimation(highlightedObject);
        }

        ClearHighlight();
    }

    void SendInteractionCommand(IARInteractable interactable)
    {
        // 위치 검증을 위한 데이터 포함
        var command = new InteractCommand
        {
            ObjectId = interactable.Id,
            ObjectType = interactable.Type,
            PlayerPosition = GetCurrentGPS(),
            ObjectPosition = interactable.GetGPS(),
            Timestamp = NetworkTime.time,
            ClientVersion = Application.version
        };

        Services.Network.SendCommand(command, OnInteractionResponse);
    }

    void OnInteractionResponse(InteractionResponse response)
    {
        if (!response.Success)
        {
            // 실패 시 롤백
            RollbackInteraction(response.ObjectId);

            // 실패 이유별 처리
            switch (response.FailReason)
            {
                case "OUT_OF_RANGE":
                    UIManager.ShowError("너무 멀리 있습니다!");
                    break;
                case "ALREADY_COLLECTED":
                    UIManager.ShowError("이미 수집되었습니다!");
                    break;
                case "INVALID_STATE":
                    UIManager.ShowError("잘못된 요청입니다!");
                    break;
            }
        }
    }

    void CheckNearbyAutoCollect()
    {
        // 3m 이내 자동 수집 (Genesis 특전)
        if (!GameState.LocalPlayer.IsGenesis) return;

        var nearbyObjects = Physics.OverlapSphere(
            transform.position,
            nearRange,
            interactableLayers
        );

        foreach (var obj in nearbyObjects)
        {
            var collectible = obj.GetComponent<IAutoCollectible>();
            if (collectible != null && !collectible.IsCollected)
            {
                SendInteractionCommand(collectible);
                collectible.MarkAsCollected(); // 중복 방지
            }
        }
    }

    // 햅틱 피드백 (iOS/Android)
    void PlayHapticFeedback(float intensity)
    {
#if UNITY_IOS
        if (SystemInfo.supportsVibration)
        {
            Handheld.Vibrate();
        }
#elif UNITY_ANDROID
        AndroidHapticFeedback.Perform(intensity);
#endif
    }
}
```

## 3. Core Game Systems

### 3.1 Location System (GPS + Mapbox)

**업데이트 주기 설정 근거:**

- `updateInterval = 1.0f`: 배터리 소모와 정확도의 균형점
  - 0.5초: 배터리 20%/hr (너무 높음)
  - 1.0초: 배터리 10%/hr (적정)
  - 2.0초: 위치 정확도 저하, 사용자 경험 악화

- `minDistance = 2.0f`: GPS 노이즈 필터링
  - 일반적인 GPS 오차가 2-5m이므로 2m 미만 이동은 무시
  - 걷기 속도(4km/h = 1.1m/s) 고려 시 적절한 값

**네트워크 의존성:**
위치 시스템은 완전히 네트워크에 의존합니다. 오프라인 상태에서는 게임이 일시정지되며, 재연결 시 서버와 동기화합니다.

```csharp
public class LocationManager : MonoBehaviour
{
    [Header("GPS Settings")]
    public float updateInterval = 1.0f;     // 업데이트 주기 (초)
    public float minDistance = 2.0f;        // 최소 이동 거리 (미터)
    public float desiredAccuracy = 10.0f;   // 목표 정확도 (미터)

    [Header("Battery Optimization")]
    public bool adaptiveInterval = true;    // 배터리 적응형 주기
    public float lowBatteryInterval = 5.0f; // 저전력 모드 주기

    [Header("Anti-Cheat")]
    public float maxSpeed = 41.67f;         // 최대 속도 (m/s) = 150km/h
    public float maxAcceleration = 10f;     // 최대 가속도 (m/s²)

    private NetworkMonitor networkMonitor;
    private Vector2d lastValidLocation;
    private float lastUpdateTime;
    private Queue<LocationData> locationHistory = new Queue<LocationData>(10);

    IEnumerator Start()
    {
        // 네트워크 필수 체크
        if (!networkMonitor.IsConnected)
        {
            ShowNetworkRequiredScreen();
            yield break;
        }

        // 위치 권한 체크
        yield return RequestLocationPermission();

        // GPS 서비스 시작
        Input.location.Start(desiredAccuracy, minDistance);

        // 초기화 대기 (최대 20초)
        int maxWait = 20;
        while (Input.location.status == LocationServiceStatus.Initializing && maxWait > 0)
        {
            yield return new WaitForSeconds(1);
            maxWait--;
        }

        // 상태별 처리
        switch (Input.location.status)
        {
            case LocationServiceStatus.Failed:
                HandleLocationFailed();
                yield break;

            case LocationServiceStatus.Running:
                StartCoroutine(LocationUpdateLoop());
                break;
        }
    }

    IEnumerator LocationUpdateLoop()
    {
        while (isActiveAndEnabled)
        {
            // 네트워크 체크 (필수)
            if (!networkMonitor.IsConnected)
            {
                PauseGame("네트워크 연결이 필요합니다");
                yield return new WaitUntil(() => networkMonitor.IsConnected);
                ResumeGame();
            }

            // 배터리 적응형 주기
            float currentInterval = updateInterval;
            if (adaptiveInterval && Battery.level < 0.2f)
            {
                currentInterval = lowBatteryInterval;
            }

            // GPS 데이터 획득
            var locInfo = Input.location.lastData;
            var currentLocation = new Vector2d(locInfo.latitude, locInfo.longitude);

            // 클라이언트 측 사전 검증
            if (ValidateLocation(currentLocation, locInfo))
            {
                // 서버로 전송
                SendLocationUpdate(currentLocation, locInfo);

                // 맵 업데이트
                UpdateMapPosition(currentLocation);

                // 히스토리 저장 (안티치트용)
                AddToHistory(currentLocation, locInfo);
            }
            else
            {
                Debug.LogWarning($"Invalid location detected: {currentLocation}");
                // 의심스러운 활동 보고
                ReportSuspiciousActivity("INVALID_LOCATION", currentLocation);
            }

            yield return new WaitForSeconds(currentInterval);
        }
    }

    bool ValidateLocation(Vector2d newLocation, LocationInfo info)
    {
        // 1. 정확도 체크
        if (info.horizontalAccuracy > 50f) // 50m 이상 오차는 무시
        {
            Debug.Log($"Low accuracy: {info.horizontalAccuracy}m");
            return false;
        }

        // 2. 첫 위치는 항상 유효
        if (lastValidLocation == null)
        {
            lastValidLocation = newLocation;
            lastUpdateTime = Time.time;
            return true;
        }

        // 3. 속도 체크
        float distance = Vector2d.Distance(lastValidLocation, newLocation);
        float timeDelta = Time.time - lastUpdateTime;
        float speed = distance / timeDelta;

        if (speed > maxSpeed)
        {
            Debug.LogWarning($"Speed violation: {speed}m/s > {maxSpeed}m/s");
            return false;
        }

        // 4. 가속도 체크 (급격한 속도 변화)
        if (locationHistory.Count > 2)
        {
            var prevSpeed = CalculateSpeed(locationHistory.ElementAt(1), locationHistory.ElementAt(0));
            float acceleration = Mathf.Abs(speed - prevSpeed) / timeDelta;

            if (acceleration > maxAcceleration)
            {
                Debug.LogWarning($"Acceleration violation: {acceleration}m/s²");
                return false;
            }
        }

        // 5. 지그재그 패턴 감지
        if (IsZigzagPattern())
        {
            Debug.LogWarning("Zigzag pattern detected");
            return false;
        }

        // 검증 통과
        lastValidLocation = newLocation;
        lastUpdateTime = Time.time;
        return true;
    }

    void SendLocationUpdate(Vector2d location, LocationInfo info)
    {
        var update = new LocationUpdate
        {
            Latitude = location.x,
            Longitude = location.y,
            Accuracy = info.horizontalAccuracy,
            Altitude = info.altitude,
            Timestamp = info.timestamp,
            Speed = CalculateSpeed(),
            Heading = Input.compass.trueHeading,
            DeviceInfo = GetDeviceInfo()
        };

        Services.Network.SendLocationUpdate(update);
    }

    // 지그재그 패턴 감지 (GPS 스푸핑 징후)
    bool IsZigzagPattern()
    {
        if (locationHistory.Count < 5) return false;

        var angles = new List<float>();
        for (int i = 2; i < locationHistory.Count; i++)
        {
            var p1 = locationHistory.ElementAt(i - 2);
            var p2 = locationHistory.ElementAt(i - 1);
            var p3 = locationHistory.ElementAt(i);

            float angle = CalculateAngle(p1.location, p2.location, p3.location);
            angles.Add(angle);
        }

        // 급격한 방향 전환이 반복되면 의심
        int sharpTurns = angles.Count(a => a > 120f);
        return sharpTurns >= 3;
    }
}
```

### 3.2 Coin Collection Mechanics

**설계 근거:**
코인 수집은 완전히 서버 권위적입니다. 클라이언트는 시각적 표현만 담당하고, 실제 수집 여부는 서버가 결정합니다. Optimistic UI로 즉각적인 피드백을 제공하되, 서버 거부 시 롤백합니다.

**수집 거리 설정:**

- 기본: 10m (GPS 오차 고려)
- Genesis 특전: 3m 자동 수집
- 최대 표시: 100m (성능 고려)

```csharp
public class CoinCollectionSystem : MonoBehaviour
{
    [Header("Collection Settings")]
    public float collectionRange = 10f;         // 수집 가능 거리
    public float autoCollectRange = 3f;        // 자동 수집 거리 (Genesis)
    public float displayRange = 100f;          // 표시 거리
    public float collectionCooldown = 0.5f;    // 연속 수집 방지

    [Header("Visual Settings")]
    public int maxVisibleCoins = 50;           // 최대 표시 개수
    public float coinRotationSpeed = 90f;      // 회전 속도

    private Dictionary<string, CoinVisual> activeCoins;
    private Queue<CollectionRequest> pendingRequests;
    private float lastCollectionTime;

    // 서버로부터 코인 데이터 수신
    public void UpdateNearbyCoins(List<CoinData> serverCoins)
    {
        // 성능을 위해 거리순 정렬
        serverCoins.Sort((a, b) =>
            Vector2d.Distance(a.Location, PlayerGPS).CompareTo(
            Vector2d.Distance(b.Location, PlayerGPS)));

        // 표시 범위 밖 코인 제거
        RemoveDistantCoins(serverCoins);

        // 최대 개수 제한
        int displayCount = Mathf.Min(serverCoins.Count, maxVisibleCoins);

        for (int i = 0; i < displayCount; i++)
        {
            var coinData = serverCoins[i];

            if (activeCoins.ContainsKey(coinData.Id))
            {
                // 기존 코인 업데이트
                UpdateCoinVisual(coinData);
            }
            else
            {
                // 새 코인 생성
                SpawnCoinVisual(coinData);
            }
        }
    }

    void SpawnCoinVisual(CoinData data)
    {
        // 타입별 풀에서 가져오기
        var coinGO = CoinPool.Get(data.Type);
        var visual = coinGO.GetComponent<CoinVisual>();

        // 월드 좌표 설정
        var worldPos = GPSToWorld(data.Location);
        visual.SetWorldPosition(worldPos);
        visual.SetData(data);

        // 거리별 품질 설정
        float distance = Vector3.Distance(PlayerPosition, worldPos);
        SetCoinQuality(visual, distance);

        // AR/Map 모드 설정
        if (ARManager.IsARMode)
        {
            visual.EnableARMode();
            ApplyAREffects(visual);
        }
        else
        {
            visual.EnableMapMode();
        }

        // 수집 가능 여부 표시
        bool canCollect = distance <= collectionRange;
        visual.SetCollectible(canCollect);

        activeCoins[data.Id] = visual;

        // 스폰 애니메이션
        visual.PlaySpawnAnimation();
    }

    // 코인 수집 시도
    public void TryCollectCoin(string coinId)
    {
        // 쿨다운 체크
        if (Time.time - lastCollectionTime < collectionCooldown)
        {
            Debug.Log("Collection on cooldown");
            return;
        }

        if (!activeCoins.ContainsKey(coinId))
        {
            Debug.LogWarning($"Coin {coinId} not found");
            return;
        }

        var coin = activeCoins[coinId];

        // 클라이언트 측 거리 체크 (UX용)
        float distance = Vector3.Distance(PlayerPosition, coin.transform.position);
        if (distance > collectionRange)
        {
            ShowTooFarMessage(distance);
            return;
        }

        // 서버에 수집 요청
        var request = new CollectCoinRequest
        {
            RequestId = Guid.NewGuid().ToString(), // 멱등성
            CoinId = coinId,
            CoinType = coin.Data.Type,
            PlayerPosition = GetCurrentGPS(),
            CoinPosition = coin.Data.Location,
            Distance = distance,
            Timestamp = NetworkTime.time,
            PickaxeId = GameState.EquippedPickaxe?.Id
        };

        // 요청 대기열에 추가
        pendingRequests.Enqueue(request);

        // Optimistic UI
        coin.PlayCollectAnimation();
        ShowCollectingUI(coin.Data.Value);
        lastCollectionTime = Time.time;

        // 서버 전송
        Services.Network.SendRequest(request, (response) =>
            OnCollectionResponse(request, response));
    }

    void OnCollectionResponse(CollectCoinRequest request, CollectCoinResponse response)
    {
        // 대기열에서 제거
        pendingRequests = new Queue<CollectionRequest>(
            pendingRequests.Where(r => r.RequestId != request.RequestId));

        if (response.Success)
        {
            // 성공 처리
            HandleCollectionSuccess(request.CoinId, response);
        }
        else
        {
            // 실패 처리 (롤백)
            HandleCollectionFailure(request.CoinId, response);
        }
    }

    void HandleCollectionSuccess(string coinId, CollectCoinResponse response)
    {
        if (!activeCoins.ContainsKey(coinId)) return;

        var coin = activeCoins[coinId];

        // 성공 이펙트
        coin.PlaySuccessEffect();

        // 보상 표시
        UIManager.ShowReward(new RewardData
        {
            Coins = response.CoinReward,
            Experience = response.ExpReward,
            BonusMultiplier = response.BonusMultiplier // Genesis 2x
        });

        // 사운드
        AudioManager.PlaySound("coin_collect");

        // 햅틱 피드백
        HapticFeedback.Success();

        // 제거 (풀로 반환)
        StartCoroutine(RemoveCoinAfterEffect(coin));
    }

    void HandleCollectionFailure(string coinId, CollectCoinResponse response)
    {
        if (!activeCoins.ContainsKey(coinId)) return;

        var coin = activeCoins[coinId];

        // 애니메이션 롤백
        coin.CancelCollection();

        // 에러 메시지
        switch (response.FailReason)
        {
            case "OUT_OF_RANGE":
                UIManager.ShowError($"너무 멀리 있습니다! ({response.ActualDistance:F1}m)");
                break;

            case "ALREADY_COLLECTED":
                UIManager.ShowError("다른 플레이어가 먼저 수집했습니다!");
                // 즉시 제거
                RemoveCoin(coinId);
                break;

            case "INVALID_PICKAXE":
                UIManager.ShowError("유효한 곡괭이가 필요합니다!");
                break;

            case "ENERGY_DEPLETED":
                UIManager.ShowError("에너지가 부족합니다!");
                ShowEnergyPurchaseOption();
                break;

            default:
                UIManager.ShowError("수집에 실패했습니다.");
                break;
        }
    }

    // Genesis 자동 수집
    void CheckAutoCollection()
    {
        if (!GameState.LocalPlayer.IsGenesis) return;

        foreach (var coin in activeCoins.Values)
        {
            float distance = Vector3.Distance(PlayerPosition, coin.transform.position);

            if (distance <= autoCollectRange && !coin.IsPendingCollection)
            {
                TryCollectCoin(coin.Data.Id);
            }
        }
    }
}
```

### 3.3 Inventory System

**설계 근거:**
인벤토리는 서버 데이터의 로컬 캐시입니다. 모든 아이템 사용은 서버 검증을 거치며, 클라이언트는 표시와 요청 전송만 담당합니다.

**아이템 사용 플로우:**

1. 클라이언트: 사용 요청 → Pending UI 표시
2. 서버: 검증 → 상태 변경 → 응답
3. 클라이언트: 응답 수신 → UI 업데이트

```csharp
public class InventorySystem : MonoBehaviour
{
    [Header("Inventory Settings")]
    public int maxSlots = 100;                  // 최대 슬롯
    public float syncInterval = 30f;            // 서버 동기화 주기

    // 서버 데이터 캐시 (읽기 전용)
    private Dictionary<string, InventoryItem> items;
    private PickaxeData equippedPickaxe;
    private DateTime lastSyncTime;

    // 대기 중인 요청 추적
    private HashSet<string> pendingActions = new HashSet<string>();

    void Start()
    {
        // 초기 인벤토리 로드
        LoadInventoryFromServer();

        // 주기적 동기화
        InvokeRepeating(nameof(SyncWithServer), syncInterval, syncInterval);
    }

    // 서버에서 인벤토리 로드
    public async void LoadInventoryFromServer()
    {
        try
        {
            var response = await Services.Network.GetInventory();

            // 로컬 캐시 업데이트
            items.Clear();
            foreach (var itemData in response.Items)
            {
                items[itemData.Id] = new InventoryItem(itemData);
            }

            equippedPickaxe = response.EquippedPickaxe;
            lastSyncTime = DateTime.Now;

            // UI 업데이트
            OnInventoryUpdated?.Invoke();

            // Genesis 특별 아이템 체크
            if (GameState.LocalPlayer.IsGenesis)
            {
                HighlightGenesisItems();
            }
        }
        catch (Exception e)
        {
            Debug.LogError($"Failed to load inventory: {e}");
            // 재시도 로직
            StartCoroutine(RetryLoadInventory());
        }
    }

    // 아이템 사용 요청
    public void UseItem(string itemId)
    {
        // 중복 요청 방지
        if (pendingActions.Contains(itemId))
        {
            UIManager.ShowWarning("처리 중입니다...");
            return;
        }

        if (!items.ContainsKey(itemId))
        {
            UIManager.ShowError("아이템을 찾을 수 없습니다!");
            return;
        }

        var item = items[itemId];

        // 클라이언트 측 기본 검증
        if (!CanUseItem(item))
        {
            ShowCannotUseReason(item);
            return;
        }

        // 서버 요청
        var request = new UseItemRequest
        {
            ItemId = itemId,
            ItemType = item.Type,
            Position = GetCurrentGPS(),
            Context = GetGameContext() // 현재 게임 상황
        };

        // Pending 상태 표시
        pendingActions.Add(itemId);
        UIManager.ShowItemUsePending(item);

        Services.Network.SendRequest(request, (response) =>
            OnUseItemResponse(itemId, response));
    }

    void OnUseItemResponse(string itemId, UseItemResponse response)
    {
        // Pending 상태 해제
        pendingActions.Remove(itemId);

        if (response.Success)
        {
            // 성공 처리
            HandleItemUseSuccess(itemId, response);
        }
        else
        {
            // 실패 처리
            HandleItemUseFailure(itemId, response);
        }
    }

    // 곡괭이 업그레이드
    public void UpgradePickaxe()
    {
        if (equippedPickaxe == null)
        {
            UIManager.ShowError("장착된 곡괭이가 없습니다!");
            return;
        }

        // 업그레이드 정보 표시
        var upgradeInfo = new UpgradeInfo
        {
            CurrentLevel = equippedPickaxe.Level,
            NextLevel = equippedPickaxe.Level + 1,
            SuccessRate = GetUpgradeSuccessRate(equippedPickaxe.Level),
            RequiredTokens = GetUpgradeTokenCost(equippedPickaxe.Level),
            CurrentTokens = GameState.LocalPlayer.Tokens
        };

        // 업그레이드 UI 표시
        UIManager.ShowUpgradeDialog(upgradeInfo, () =>
        {
            // 확인 시 서버 요청
            var request = new UpgradePickaxeRequest
            {
                PickaxeId = equippedPickaxe.Id,
                CurrentLevel = equippedPickaxe.Level,
                UseProtection = false // 보호 아이템 사용 여부
            };

            Services.Network.SendRequest(request, OnUpgradeResponse);

            // 대기 UI
            UIManager.ShowUpgradePending();
        });
    }

    void OnUpgradeResponse(UpgradePickaxeResponse response)
    {
        if (response.Success)
        {
            // 성공 애니메이션
            UIManager.ShowUpgradeSuccess(response.NewLevel);

            // 로컬 데이터 업데이트
            equippedPickaxe.Level = response.NewLevel;
            equippedPickaxe.Efficiency = response.NewEfficiency;

            // 이펙트
            PlayUpgradeEffect();
        }
        else
        {
            // 실패 처리
            if (response.PickaxeDestroyed)
            {
                UIManager.ShowUpgradeDestroyed();
                equippedPickaxe = null;
            }
            else
            {
                UIManager.ShowUpgradeFailed();
            }
        }

        // 인벤토리 재동기화
        LoadInventoryFromServer();
    }

    // 서버와 주기적 동기화
    void SyncWithServer()
    {
        // 네트워크 체크
        if (!Services.Network.IsConnected) return;

        // 변경사항이 있을 때만 동기화
        if (HasLocalChanges())
        {
            LoadInventoryFromServer();
        }
    }

    // 업그레이드 성공률 계산 (클라이언트 표시용)
    float GetUpgradeSuccessRate(int level)
    {
        // 서버와 동일한 공식 사용
        return level switch
        {
            1 => 0.9f,   // 90%
            2 => 0.8f,   // 80%
            3 => 0.7f,   // 70%
            4 => 0.6f,   // 60%
            5 => 0.5f,   // 50%
            _ => 0.3f    // 30%
        };
    }
}
```

### 3.4 Quest System

**설계 근거:**
퀘스트 진행도는 서버에서 추적하며, 클라이언트는 표시만 담당합니다. 일일 퀘스트는 서버 시간 기준으로 리셋되며, Genesis 멤버는 추가 퀘스트를 받습니다.

```csharp
public class QuestSystem : MonoBehaviour
{
    [Header("Quest Settings")]
    public float updateInterval = 5f;           // 진행도 업데이트 주기

    private List<QuestData> activeQuests;
    private Dictionary<string, QuestProgress> progress;
    private DateTime lastResetTime;

    void Start()
    {
        // 초기 퀘스트 로드
        LoadQuestsFromServer();

        // 주기적 업데이트
        InvokeRepeating(nameof(UpdateQuestProgress), updateInterval, updateInterval);
    }

    // 서버에서 퀘스트 데이터 수신
    public void UpdateQuests(QuestUpdate update)
    {
        activeQuests = update.Quests;
        progress = update.Progress;
        lastResetTime = update.ResetTime;

        // Genesis 전용 퀘스트 필터링
        if (GameState.LocalPlayer.IsGenesis)
        {
            var genesisQuests = update.GenesisExclusiveQuests;
            activeQuests.AddRange(genesisQuests);
        }

        // UI 업데이트
        foreach (var quest in activeQuests)
        {
            var prog = progress[quest.Id];

            // 진행도 표시
            UIManager.UpdateQuestProgress(quest, prog);

            // 완료 체크
            if (prog.IsCompleted && !prog.IsClaimed)
            {
                ShowQuestCompleteNotification(quest);
            }
        }

        // 일일 리셋 체크
        CheckDailyReset();
    }

    // 퀘스트 완료 알림
    void ShowQuestCompleteNotification(QuestData quest)
    {
        var notification = new QuestNotification
        {
            Title = "퀘스트 완료!",
            Description = quest.Name,
            Rewards = FormatRewards(quest.Rewards),
            AutoDismiss = 5f
        };

        UIManager.ShowQuestNotification(notification);

        // 사운드 & 이펙트
        AudioManager.PlaySound("quest_complete");
        HapticFeedback.Success();

        // Genesis 보너스 표시
        if (GameState.LocalPlayer.IsGenesis)
        {
            notification.Rewards += " (Genesis 2x 보너스!)";
        }
    }

    // 퀘스트 보상 수령
    public void ClaimQuestReward(string questId)
    {
        if (!progress.ContainsKey(questId))
        {
            Debug.LogError($"Quest {questId} not found");
            return;
        }

        var questProgress = progress[questId];

        // 완료 체크
        if (!questProgress.IsCompleted)
        {
            UIManager.ShowError("퀘스트가 아직 완료되지 않았습니다!");
            return;
        }

        // 이미 수령 체크
        if (questProgress.IsClaimed)
        {
            UIManager.ShowError("이미 보상을 받았습니다!");
            return;
        }

        // 서버 요청
        var request = new ClaimQuestRequest
        {
            QuestId = questId,
            CompletedAt = questProgress.CompletedAt
        };

        // Pending UI
        UIManager.ShowClaimPending(questId);

        Services.Network.SendRequest(request, OnClaimResponse);
    }

    void OnClaimResponse(ClaimQuestResponse response)
    {
        if (response.Success)
        {
            // 보상 표시
            UIManager.ShowRewardClaimed(response.Rewards);

            // 로컬 상태 업데이트
            progress[response.QuestId].IsClaimed = true;

            // 다음 퀘스트 체크
            if (response.NextQuest != null)
            {
                AddNewQuest(response.NextQuest);
            }
        }
        else
        {
            UIManager.ShowError(response.ErrorMessage);
        }
    }

    // 일일 리셋 체크
    void CheckDailyReset()
    {
        var serverTime = NetworkTime.ServerTime;
        var resetHour = 4; // 오전 4시 리셋

        var nextReset = serverTime.Date.AddHours(resetHour);
        if (serverTime.Hour >= resetHour)
        {
            nextReset = nextReset.AddDays(1);
        }

        var timeUntilReset = nextReset - serverTime;

        // 리셋 카운트다운 표시
        UIManager.UpdateDailyResetTimer(timeUntilReset);

        // 리셋 시간이 지났으면 새 퀘스트 요청
        if (serverTime > lastResetTime.AddDays(1))
        {
            LoadQuestsFromServer();
        }
    }
}
```

## 4. Networking Layer

### 4.1 API Client Architecture

**설계 근거:**
네트워크는 게임의 생명선입니다. 연결이 끊어지면 게임이 중단되며, 모든 상태는 서버와 동기화됩니다. 재연결 시 자동으로 상태를 복구합니다.

**재시도 전략:**

- Exponential Backoff with Jitter 적용
- 401: 토큰 갱신 후 재시도
- 429: Rate limit 헤더 확인, 지정된 시간 대기
- 5xx: 재시도 (서버 일시적 문제)
- 4xx (401, 429 제외): 재시도 안함 (클라이언트 오류)

```csharp
public class NetworkManager : SingletonBehaviour<NetworkManager>
{
    [Header("Configuration")]
    public string apiBaseUrl = "https://api.ore.game/v1";
    public string wsUrl = "wss://ws.ore.game/v1/realtime";
    public float requestTimeout = 10f;
    public int maxRetries = 3;

    [Header("Rate Limiting")]
    public int requestsPerMinute = 60;
    public float burstCapacity = 10;

    private RestClient restClient;
    private WebSocketClient wsClient;
    private Queue<NetworkCommand> commandQueue;
    private string authToken;
    private DateTime tokenExpiry;

    // 연결 상태
    public bool IsConnected { get; private set; }
    public NetworkReachability LastReachability { get; private set; }

    // 재연결 관리
    private int reconnectAttempts = 0;
    private float reconnectDelay = 1f;

    void Awake()
    {
        // REST 클라이언트 초기화
        restClient = new RestClient(apiBaseUrl)
        {
            Timeout = (int)(requestTimeout * 1000),
            ThrowOnAnyError = false,
            UserAgent = $"ORE-Unity/{Application.version}"
        };

        // WebSocket 설정
        InitializeWebSocket();

        // 네트워크 모니터링 시작
        StartCoroutine(NetworkMonitor());

        // 명령 큐 처리
        StartCoroutine(ProcessCommandQueue());
    }

    void InitializeWebSocket()
    {
        wsClient = new WebSocketClient(wsUrl);

        wsClient.OnOpen += () =>
        {
            Debug.Log("WebSocket connected");
            IsConnected = true;
            reconnectAttempts = 0;
            SendAuthentication();
        };

        wsClient.OnMessage += HandleWebSocketMessage;

        wsClient.OnError += (error) =>
        {
            Debug.LogError($"WebSocket error: {error}");
            HandleConnectionError(error);
        };

        wsClient.OnClose += (code) =>
        {
            Debug.Log($"WebSocket closed: {code}");
            IsConnected = false;
            HandleDisconnection(code);
        };
    }

    IEnumerator NetworkMonitor()
    {
        while (true)
        {
            var currentReachability = Application.internetReachability;

            // 네트워크 상태 변경 감지
            if (currentReachability != LastReachability)
            {
                LastReachability = currentReachability;
                OnNetworkReachabilityChanged(currentReachability);
            }

            // 연결 상태 체크
            if (currentReachability == NetworkReachability.NotReachable)
            {
                if (IsConnected)
                {
                    OnNetworkLost();
                }
            }
            else if (!IsConnected)
            {
                // 자동 재연결 시도
                AttemptReconnection();
            }

            // 토큰 만료 체크
            if (tokenExpiry != default && DateTime.Now > tokenExpiry.AddMinutes(-5))
            {
                RefreshAuthToken();
            }

            yield return new WaitForSeconds(1f);
        }
    }

    void OnNetworkLost()
    {
        Debug.LogWarning("Network connection lost!");

        IsConnected = false;

        // 게임 일시정지
        Time.timeScale = 0;

        // UI 표시
        UIManager.ShowNetworkLostOverlay(
            "네트워크 연결이 끊어졌습니다.\n" +
            "자동으로 재연결을 시도합니다..."
        );

        // WebSocket 종료
        wsClient?.Close();

        // 로컬 상태 정리
        GameState.SetOfflineMode(true);
    }

    void AttemptReconnection()
    {
        if (reconnectAttempts >= 10)
        {
            UIManager.ShowFatalError(
                "서버에 연결할 수 없습니다.\n" +
                "인터넷 연결을 확인하고 앱을 재시작해주세요."
            );
            return;
        }

        reconnectAttempts++;

        // Exponential backoff
        reconnectDelay = Mathf.Min(reconnectDelay * 2f, 30f);

        Debug.Log($"Reconnection attempt {reconnectAttempts} in {reconnectDelay}s");

        StartCoroutine(ReconnectAfterDelay(reconnectDelay));
    }

    IEnumerator ReconnectAfterDelay(float delay)
    {
        yield return new WaitForSecondsRealtime(delay);

        // 토큰 갱신
        bool tokenRefreshed = await RefreshAuthToken();
        if (!tokenRefreshed)
        {
            // 재로그인 필요
            SceneManager.LoadScene("LoginScene");
            return;
        }

        // WebSocket 재연결
        await wsClient.Connect(authToken);

        if (IsConnected)
        {
            // 연결 성공
            OnNetworkRestored();
        }
        else
        {
            // 재시도
            AttemptReconnection();
        }
    }

    void OnNetworkRestored()
    {
        Debug.Log("Network connection restored!");

        // 게임 재개
        Time.timeScale = 1;

        // UI 숨기기
        UIManager.HideNetworkLostOverlay();

        // 상태 동기화
        StartCoroutine(SynchronizeGameState());
    }

    IEnumerator SynchronizeGameState()
    {
        UIManager.ShowLoading("동기화 중...");

        // 1. 사용자 프로필
        var profile = await GetUserProfile();
        GameState.UpdateLocalPlayer(profile);

        // 2. 인벤토리
        var inventory = await GetInventory();
        InventorySystem.Instance.LoadInventory(inventory);

        // 3. 현재 위치 업데이트
        LocationManager.Instance.ForceLocationUpdate();

        // 4. 퀘스트 상태
        var quests = await GetActiveQuests();
        QuestSystem.Instance.UpdateQuests(quests);

        UIManager.HideLoading();

        // 동기화 완료 알림
        UIManager.ShowNotification("동기화 완료!");
    }
}
```

[문서가 계속됩니다...]

## 5. UI/UX Systems

### 5.1 UI Toolkit Architecture

**설계 근거:**
Unity의 새로운 UI Toolkit을 사용하여 성능과 유지보수성을 개선합니다. UXML/USS로 디자인과 로직을 분리하여 AI가 독립적으로 각 부분을 구현할 수 있습니다.

### 5.2 Screen Layouts (UXML)

**설계 원칙:**

- 반응형 디자인: 다양한 화면 크기 지원
- 접근성: 폰트 크기 조절, 색맹 모드
- Genesis 특별 UI: 멤버 전용 시각 효과

### 5.3 Styling (USS)

**Genesis 특별 스타일링:**
황금색 테마와 펄스 애니메이션으로 Genesis 멤버의 특별함을 강조합니다.

## 6. Asset Management

### 6.1 Addressables System

**설계 근거:**
Addressables로 동적 콘텐츠 로딩을 구현하여 앱 크기를 최소화합니다. 필수 에셋만 포함하고 나머지는 필요시 다운로드합니다.

### 6.2 Object Pooling

**풀링 전략:**

- 초기 크기: 예상 사용량의 50%
- 최대 크기: 예상 사용량의 200%
- 확장 가능: 필요시 동적 생성

## 7. Performance Optimization

### 7.1 Mobile Performance Targets

**성능 예산 설정 근거:**

- CPU: 16ms/프레임 (60 FPS 유지)
- 메모리: 500MB (저사양 기기 고려)
- 배터리: 10%/시간 (1시간 플레이 보장)

### 7.2-7.4 최적화 전략

**디바이스 티어링:**

- High: iPhone 13+, Snapdragon 888+
- Medium: iPhone 11+, Snapdragon 765+
- Low: 그 외 기기

## 8. Platform Specific

### 8.1 iOS Configuration

**iOS 특별 고려사항:**

- ATT(App Tracking Transparency) 대응
- ARKit 요구사항 명시
- 햅틱 엔진 활용

### 8.2 Android Configuration

**Android 특별 고려사항:**

- 다양한 해상도/DPI 대응
- ARCore 가용성 체크
- 배터리 최적화 예외 요청

## 9. Development Guidelines

### 9.1 Unity AI-Native Patterns

**AI 프롬프트 템플릿 제공으로 일관된 코드 생성**

### 9.2 Code Organization

**Assembly Definition으로 컴파일 시간 단축 및 의존성 관리**

### 9.3 Testing Strategy

**테스트 우선순위:**

1. 네트워크 연결/재연결
2. 위치 검증
3. 코인 수집 로직
4. AR 트래킹 복구

### 9.4 Build Pipeline

**빌드 최적화:**

- 코드 스트리핑: Moderate (안정성 우선)
- IL2CPP: Master 최적화
- 텍스처 압축: ASTC (균형)

## 10. Integration Points

### 10.1-10.3 External Dependencies

**필수 SDK 버전 고정으로 호환성 보장**

---

## Conclusion

이 Frontend System Specification v2.0은 AI-Native 개발을 위한 완전한 기술 명세입니다.

**핵심 개선사항:**

- 모든 설계 결정에 **근거(Why)** 추가
- 구현 방법에 대한 **상세 설명(How)** 포함
- 임계값과 매직넘버의 **이유** 명시
- AI 에이전트를 위한 **명확한 가이드라인**
- 디버깅을 위한 **체크포인트** 제공

**AI 개발자를 위한 핵심 포인트:**

1. **서버 권위적**: 클라이언트는 표시만, 검증은 서버
2. **네트워크 필수**: 오프라인 = 게임 중단
3. **성능 우선**: 60 FPS, 500MB RAM, 10%/hr 배터리
4. **Unity 6.0 LTS**: 안정성 검증된 버전
5. **Genesis 특별 대우**: 2x 보상, 자동 수집, 전용 UI

이 명세서를 따라 구현하면 12주 내에 Genesis 1000을 위한 안정적인 MVP를 출시할 수 있습니다.

---

_Version: 2.0_
_Last Updated: 2024-12-20_
_Unity Version: 2023.3 LTS_
_AR Foundation: 5.1_
_Target Platforms: iOS 14+, Android 10+_
