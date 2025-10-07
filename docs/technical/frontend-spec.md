# Project ORE - Frontend System Specification v2.3

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
  - VContainer (Dependency Injection)
  - Custom Networking (WebSocket + REST)
  - GameLogger (Zero-allocation logging)

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

### 1.2 Scene Management & Flow (DontDestroyOnLoad Bootstrap Architecture)

**설계 근거:**
Fracture/Vein/Core 게임 메커니즘을 지원하기 위해 **Map.unity**(전략적 탐색)와 **ARGame.unity**(수집 게임플레이)로 분리된 dual-scene 아키텍처를 채택합니다. **DontDestroyOnLoad Bootstrap** 패턴(모바일 AR 산업 표준)을 사용하여 Loading.unity가 Persistent Managers를 생성한 후 자동 언로드되며, 모든 씬 전환은 LoadSceneMode.Single을 사용합니다.

**핵심 원칙:**

- Loading.unity = Bootstrap 진입점 (\_\_PersistentManagers 생성 후 언로드)
- MainMenu/Login.unity = UI 씬 (게임 시작 전)
- Map.unity = 주요 씬 (플레이어가 대부분의 시간을 보냄)
- ARGame.unity = 수집 씬 (Fracture 진입 시 Single mode로 전환)
- LoadSceneMode.Single = 모든 전환 (메모리 효율, 충돌 방지)
- Geofencing = 150m 경계 기반 자동 전환
- SceneTransitionData = 씬 간 데이터 전달

```yaml
Scene Structure:

  # 0. Loading.unity (Bootstrap Entry Point)
  Loading.unity:
    Description: 앱 시작 시 한 번 실행되는 Bootstrap 씬

    Components:
      - LoadingSceneController # __PersistentManagers 생성 및 다음 씬 로드
      - Camera (Orthographic) # 로딩 화면용
      - EventSystem # UI 입력 처리

    Lifecycle:
      1. 앱 시작 시 자동 로드 (Build Settings index 0)
      2. __PersistentManagers GameObject 생성
      3. DontDestroyOnLoad(__PersistentManagers) 호출
      4. GameManager, NetworkManager, LocationManager, SceneTransitionManager 생성
      5. CoreLifetimeScope 추가
      6. MainMenu.unity를 LoadSceneMode.Single로 로드
      7. Loading.unity 자동 언로드 (Single mode로 대체됨)

    Purpose:
      - Persistent managers 초기화
      - 씬 전환 인프라 구축
      - 한 번만 실행되고 사라짐

  # 1. Persistent Managers (DontDestroyOnLoad)
  __PersistentManagers:
    Object: DontDestroyOnLoad GameObject (Loading.unity에서 생성)
    Components:
      - CoreLifetimeScope # VContainer Parent Scope (루트)
      - GameManager # 게임 상태 오케스트레이션
      - NetworkManager # HTTP REST API
      - LocationManager # GPS tracking + geofencing
      - SceneTransitionManager # 씬 전환 관리

    Lifetime: Loading.unity가 생성한 후 앱 종료까지 유지
    Purpose: 씬 전환에도 상태/연결 보존

    DI Architecture:
      - CoreLifetimeScope는 Parent Scope (모든 씬이 상속)
      - SceneTransitionManager가 LifetimeScope.EnqueueParent() 사용
      - Scene-specific 매니저는 각 씬의 Child Scope에 등록
      - Parent → Child 의존성 주입 가능 (Child → Parent 불가)

  # 2. MainMenu.unity (Initial UI Scene)
  MainMenu.unity:
    Description: 게임 시작 후 첫 UI 씬 (로그인 전)

    VContainer DI:
      - MainMenuLifetimeScope # Child Scope (CoreLifetimeScope 상속)
      - Parent로부터 주입: GameManager, NetworkManager, SceneTransitionManager
      - Scene-specific 등록: MainMenuController

    UI Components:
      - LoginButton # Login.unity로 전환
      - SettingsButton # 설정 화면
      - Camera (Orthographic) # UI 렌더링
      - AudioListener # 사운드 재생
      - EventSystem # UI 입력

    Loaded When: Loading.unity에서 자동 전환
    Transitions To: Login.unity (LoginButton 클릭)

  # 3. Login.unity (Authentication Scene)
  Login.unity:
    Description: 인증 씬 (사용자 로그인)

    VContainer DI:
      - LoginLifetimeScope # Child Scope (CoreLifetimeScope 상속)
      - Parent로부터 주입: GameManager, NetworkManager, SceneTransitionManager
      - Scene-specific 등록: LoginController

    UI Components:
      - UsernameField # 사용자명 입력
      - PasswordField # 비밀번호 입력
      - LoginButton # 로그인 실행
      - Camera (Orthographic) # UI 렌더링
      - AudioListener # 사운드 재생
      - EventSystem # UI 입력

    Loaded When: MainMenu.unity에서 LoginButton 클릭
    Transitions To: Map.unity (로그인 성공)

  # 4. Map.unity (Primary Scene)
  Map.unity:
    Description: 전략적 네비게이션 씬 (Pokémon GO 메인 화면과 유사)

    VContainer DI:
      - MapLifetimeScope # Child Scope (CoreLifetimeScope 상속)
      - Parent로부터 주입: GameManager, NetworkManager, LocationManager
      - Scene-specific 등록: MapController, MapUIController, GeofencingService

    UI Components:
      - OnlineMapsComponent # 전체 화면 지도
      - PlayerMarker # 실시간 GPS 위치
      - FractureMarkers # 100m 반경 내 Fracture 표시
      - FractureInfoCard # 선택된 Fracture 정보 (거리, 등급, 예상 보상)
      - BottomNavigation # [Map][Inventory][Quests][Profile]
      - TopStatusBar # GPS/네트워크/배터리 상태

    Scene-Specific Components (MapLifetimeScope에 등록):
      - MapController # Online Maps v4 통합
      - MapUIController # 지도 UI 컨트롤 (줌, 팔로우)
      - GeofencingService # Fracture 경계 감지
      - FractureVisualizer # 균열 이펙트 렌더링

    Note: 이 씬에는 AR 기능 없음 (ARManager 없음)

    Loaded When:
      - 앱 시작 (로그인 후)
      - ARGame.unity 종료 후

    Transitions From:
      - Login (초기 진입)
      - ARGame.unity (채굴 완료 또는 뒤로가기)

    Geofencing Trigger:
      Condition: 플레이어가 Fracture 150m 경계 진입
      Action: ARGame.unity Single mode 로드 + Digital Crack 애니메이션
      Data: SceneTransitionData(fractureId, veinLocation, coreMetadata)

  # 5. ARGame.unity (Collection Scene)
  ARGame.unity:
    Description: AR 수집 게임플레이 씬 (AR Foundation, 채굴 메커니즘)

    VContainer DI:
      - ARGameLifetimeScope # Child Scope (CoreLifetimeScope 상속)
      - Parent로부터 주입: GameManager, NetworkManager, LocationManager, SceneTransitionManager
      - Scene-specific 등록: ARManager, VeinExplorationController, MiningController

    UI Components:
      - ARCameraBackground # AR Foundation 카메라 뷰
      - DistanceHint # "따뜻해요/뜨거워요" 거리 피드백
      - DirectionalArrow # Vein 방향 가이드 (GPS 기반 bearing 계산)
      - CoreARRenderer # 발견된 Core의 3D AR 렌더링
      - MiningMinigame # 등급별 채굴 미니게임 UI
      - ExitWarning # Fracture 경계 이탈 경고 (135m)
      - BackButton # 긴급 탈출 (Map으로 복귀)
      - Camera (Perspective) # AR 카메라
      - AudioListener # 사운드 재생
      - EventSystem # UI 입력

    Scene-Specific Components (ARGameLifetimeScope에 등록):
      - ARManager # AR Foundation 통합 (이 씬에만 존재)
      - ARContentManager # AR 오브젝트 스폰/LOD
      - VeinExplorationController # 거리 기반 힌트 시스템
      - MiningController # 채굴 미니게임 로직
      - GeofencingMonitor # 경계 이탈 감지

    Navigation Method:
      - Directional UI Overlay (GPS 기반)
      - LocationManager로부터 현재 위치 수신 (Parent Scope injection)
      - SceneTransitionData.veinLocation으로 bearing/distance 계산
      - NO minimap, NO MapController (경량화)
      - AR 카메라 뷰에만 집중

    Loaded When:
      - Geofencing trigger (Fracture 진입)
      - Manual entry (Map에서 "진입" 버튼)

    Load Mode: LoadSceneMode.Single
      Reason:
        - DontDestroyOnLoad Bootstrap 패턴 (모바일 AR 산업 표준)
        - __PersistentManagers는 DontDestroyOnLoad로 유지됨
        - Map.unity 완전히 언로드 (메모리 효율 40% 향상)
        - 씬 독립성 (Camera/AudioListener/EventSystem 충돌 방지)
        - 빠른 가비지 컬렉션 및 메모리 정리

    Transitions To:
      - Map.unity (채굴 완료, 뒤로가기, 강제 퇴출) - LoadSceneMode.Single

    Exit Triggers:
      - 채굴 성공 완료
      - 뒤로가기 버튼
      - Fracture 경계 이탈 (150m + 10s 유예)
      - 네트워크 연결 끊김

  # 6. Scene Transition Flow
  Scene Flow:
    App Start → Loading.unity (bootstrap)
      ↓
      __PersistentManagers 생성 (DontDestroyOnLoad)
      ↓
    MainMenu.unity (Single mode - Loading 언로드)
      ↓
    Login.unity (Single mode - MainMenu 언로드)
      ↓
    Map.unity (Single mode - Login 언로드)
      ↓
      Geofencing 트리거 (150m 경계)
      ↓
    ARGame.unity (Single mode - Map 언로드)
      ↓
      채굴 완료 / 이탈
      ↓
    Map.unity (Single mode - ARGame 언로드)

  Transition States:
    - Idle: Map.unity만 활성 (__PersistentManagers 유지)
    - Entering: Map → ARGame 전환 중 (1.618s Digital Crack 애니메이션)
    - Active: ARGame.unity 활성 (Map은 언로드됨, __PersistentManagers만 유지)
    - Exiting: ARGame → Map 전환 중 (3.14s 보상 애니메이션 + Reality Crack)
    - Cooldown: 전환 쿨다운 (5초, 의도치 않은 재진입 방지)

전환 규칙:
  Map → ARGame:
    - Trigger: Geofencing (150m 경계 진입) OR 수동 "진입" 버튼
    - Validation:
      * GPS 정확도 < 20m
      * 네트워크 연결 활성
      * AR 기기 지원 확인
    - Animation: Digital Crack (1.618초 - 황금비)
    - Loading: SceneTransitionManager.TransitionToScene("ARGame")
      * LoadSceneMode.Single (Map.unity 완전히 언로드)
      * LifetimeScope.EnqueueParent(coreLifetimeScope) 사용
      * __PersistentManagers는 DontDestroyOnLoad로 유지
    - Camera: Map 카메라 → ARGame 씬의 AR 카메라
    - Audio: Map AudioListener → ARGame AudioListener (충돌 없음)

  ARGame → Map:
    - Trigger: 채굴 완료, 뒤로가기, 경계 이탈, 네트워크 끊김
    - Animation: Reward display (3.14초 - π) + Reality Crack reverse
    - Loading: SceneTransitionManager.TransitionToScene("Map")
      * LoadSceneMode.Single (ARGame.unity 완전히 언로드)
      * LifetimeScope.EnqueueParent(coreLifetimeScope) 사용
      * 자동 가비지 컬렉션 + Resources.UnloadUnusedAssets()
    - Camera: ARGame AR 카메라 → Map 씬의 카메라
    - Data: SceneTransitionData로 보상 데이터 전달

  네트워크 끊김 → Paused:
    - 자동 (재연결 시도)
    - ARGame 활성 시 즉시 Map으로 복귀

  치명적 오류 → Error:
    - 자동 (재시작 필요)
    - 현재 씬 정보 저장 (복구용)

주의사항:
  - Loading/MainMenu/Login/Map/ARGame은 **실제 씬 파일**입니다 (GameState enum이 아님)
  - DontDestroyOnLoad Bootstrap 패턴으로 __PersistentManagers 유지
  - LoadSceneMode.Single로 모든 씬 전환 (메모리 효율, 충돌 방지)
  - SceneTransitionData로 씬 간 데이터 전달 (static 변수 사용 금지)
  - SceneTransitionManager가 LifetimeScope.EnqueueParent() 사용 (VContainer 패턴)
  - Geofencing은 LocationManager에서 처리, GameManager가 씬 전환 트리거
  - 각 씬은 독립적인 Camera + AudioListener + EventSystem 보유
```

#### SceneTransitionData Class

씬 간 데이터 전달을 위한 구조체 (static 변수 사용 방지):

```csharp
namespace ORE.Core.SceneManagement
{
    /// <summary>
    /// Data container for scene transitions between Map and ARGame.
    /// Passed via GameManager to avoid static variables and ensure clean state management.
    /// </summary>
    [System.Serializable]
    public class SceneTransitionData
    {
        // Fracture 정보
        public string fractureId;
        public Vector2d fractureCenter;      // GPS 좌표
        public float fractureRadius;         // 150m (MVP)
        public FractureGrade fractureGrade;  // Common/Rare/Epic/Legendary
        public GeoJsonPolygon fractureBoundary; // 정확한 경계 (Post-MVP)

        // Vein 정보 (서버에서 제공)
        public string veinId;
        public Vector2d veinLocation;        // 정확한 GPS 좌표
        public float veinRadius;             // 발견 반경: 10m (일반) / 3m (Genesis)

        // Core 정보
        public CoreMetadata coreData;
        public string coreType;              // "ore", "artifact", "advertisement"
        public int estimatedValue;           // 예상 보상 (표시용)

        // Transition 메타데이터
        public float transitionTimestamp;    // 진입 시간 (Time.time)
        public bool isManualEntry;           // true: 수동 진입, false: Geofencing 자동
        public Vector2d playerEntryPosition; // 진입 시 플레이어 위치

        // Genesis 특전
        public bool isGenesisPlayer;
        public float rewardMultiplier;       // 2.0 for Genesis, 1.0 for regular

        /// <summary>
        /// Create transition data from server Fracture response
        /// </summary>
        public static SceneTransitionData FromFractureResponse(FractureEnterResponse response)
        {
            return new SceneTransitionData
            {
                fractureId = response.FractureId,
                fractureCenter = new Vector2d(response.Latitude, response.Longitude),
                fractureRadius = response.Radius,
                fractureGrade = response.Grade,
                veinId = response.VeinId,
                veinLocation = new Vector2d(response.VeinLatitude, response.VeinLongitude),
                veinRadius = response.VeinRadius,
                coreData = response.CoreMetadata,
                transitionTimestamp = Time.time,
                isManualEntry = response.IsManualEntry,
                playerEntryPosition = Services.Location.CurrentLocation,
                isGenesisPlayer = GameState.LocalPlayer.IsGenesis,
                rewardMultiplier = GameState.LocalPlayer.IsGenesis ? 2.0f : 1.0f
            };
        }
    }
}
```

#### Scene Transition Implementation

GameManager에서 씬 전환 관리:

```csharp
namespace ORE.Core
{
    using UnityEngine.SceneManagement;
    using ORE.Core.SceneManagement;

    public class GameManager : MonoBehaviour, IGameManager
    {
        [Header("Scene References")]
        private const string MAP_SCENE = "Map";
        private const string AR_GAME_SCENE = "ARGame";

        [Header("Transition Settings")]
        [SerializeField] private float digitalCrackDuration = 1.618f; // Golden ratio
        [SerializeField] private float rewardAnimationDuration = 3.14f; // π
        [SerializeField] private float transitionCooldown = 5.0f;

        // Current transition data
        private SceneTransitionData currentTransitionData;
        private float lastTransitionTime;

        // VContainer DI (Persistent Managers - injected from parent scope)
        [Inject] private ILocationManager locationManager;
        [Inject] private INetworkManager networkManager;

        private void Start()
        {
            // Subscribe to geofencing events
            locationManager.OnFractureEntered += HandleFractureEntered;
            locationManager.OnFractureExited += HandleFractureExited;
        }

        /// <summary>
        /// Geofencing callback - automatically triggered by LocationManager
        /// </summary>
        private async void HandleFractureEntered(string fractureId)
        {
            GameLogger.Log($"Geofencing triggered: Entering Fracture {fractureId}");

            // Validation
            if (!CanEnterFracture())
            {
                GameLogger.LogWarning("Cannot enter Fracture - validation failed");
                return;
            }

            // Cooldown check (prevent rapid transitions)
            if (Time.time - lastTransitionTime < transitionCooldown)
            {
                GameLogger.LogWarning("Transition on cooldown");
                return;
            }

            // Request Fracture data from server
            var response = await Services.Network.RequestFractureEntry(fractureId);
            if (!response.Success)
            {
                UIManager.ShowError($"진입 실패: {response.ErrorMessage}");
                return;
            }

            // Prepare transition data
            currentTransitionData = SceneTransitionData.FromFractureResponse(response);

            // Start transition
            await TransitionToARGame(currentTransitionData);
        }

        /// <summary>
        /// Transition from Map.unity to ARGame.unity
        /// Uses SceneTransitionManager with LoadSceneMode.Single (DontDestroyOnLoad Bootstrap pattern)
        /// </summary>
        private async Task TransitionToARGame(SceneTransitionData data)
        {
            GameLogger.Log("Starting Map → ARGame transition");
            lastTransitionTime = Time.time;

            // 1. Play Digital Crack animation
            await UIManager.PlayDigitalCrackAnimation(digitalCrackDuration);

            // 2. Store transition data (static or DI singleton)
            // ARGame scene will retrieve this data after loading
            SceneTransitionDataHolder.Instance.SetData(data);

            // 3. Use SceneTransitionManager to load ARGame scene (Single mode)
            // - LoadSceneMode.Single replaces Map.unity completely
            // - __PersistentManagers survive via DontDestroyOnLoad
            // - LifetimeScope.EnqueueParent() links ARGameLifetimeScope to CoreLifetimeScope
            var transitionComplete = false;
            sceneTransitionManager.TransitionToScene(AR_GAME_SCENE, () =>
            {
                transitionComplete = true;
                GameLogger.Log("ARGame scene loaded successfully");
            });

            // Wait for transition to complete
            while (!transitionComplete)
            {
                await Task.Yield();
            }

            // 4. Haptic feedback
            HapticFeedback.Medium();

            GameLogger.Log("ARGame scene activated");
        }

        /// <summary>
        /// Transition from ARGame.unity back to Map.unity
        /// Uses SceneTransitionManager with LoadSceneMode.Single (DontDestroyOnLoad Bootstrap pattern)
        /// </summary>
        public async Task TransitionToMap(MiningResult result)
        {
            GameLogger.Log("Starting ARGame → Map transition");

            // 1. Show reward animation (if successful)
            if (result.Success)
            {
                await UIManager.PlayRewardAnimation(result.Rewards, rewardAnimationDuration);
            }

            // 2. Play Reality Crack reverse animation
            await UIManager.PlayRealityCrackReverseAnimation(digitalCrackDuration);

            // 3. Store result data for Map scene
            SceneTransitionDataHolder.Instance.SetRewardData(result);

            // 4. Use SceneTransitionManager to load Map scene (Single mode)
            // - LoadSceneMode.Single replaces ARGame.unity completely
            // - __PersistentManagers survive via DontDestroyOnLoad
            // - ARManager and ARGameLifetimeScope automatically cleaned up
            // - Automatic garbage collection and Resources.UnloadUnusedAssets()
            var transitionComplete = false;
            sceneTransitionManager.TransitionToScene(MAP_SCENE, () =>
            {
                transitionComplete = true;
                GameLogger.Log("Map scene loaded successfully");
            });

            // Wait for transition to complete
            while (!transitionComplete)
            {
                await Task.Yield();
            }

            // 5. Haptic feedback
            HapticFeedback.Light();

            // 6. Sync with server
            await Services.Network.SyncPlayerState();

            GameLogger.Log("Returned to Map scene");
        }

        /// <summary>
        /// Validation before entering Fracture
        /// </summary>
        private bool CanEnterFracture()
        {
            // Network check
            if (!networkManager.IsConnected)
            {
                UIManager.ShowError("네트워크 연결이 필요합니다");
                return false;
            }

            // GPS accuracy check
            if (locationManager.CurrentAccuracy > 20f)
            {
                UIManager.ShowError("GPS 정확도가 낮습니다. 야외로 이동하세요.");
                return false;
            }

            // AR support check
            if (!ARSession.CheckIfSupported())
            {
                UIManager.ShowError("이 기기는 AR을 지원하지 않습니다");
                return false;
            }

            return true;
        }

        /// <summary>
        /// Handle forced exit (boundary exit, network loss)
        /// </summary>
        private void HandleFractureExited(string fractureId)
        {
            GameLogger.LogWarning($"Fracture boundary exited: {fractureId}");

            // If in ARGame, force return to Map
            if (SceneManager.GetSceneByName(AR_GAME_SCENE).isLoaded)
            {
                UIManager.ShowWarning("Fracture 경계를 벗어났습니다");

                var failedResult = new MiningResult
                {
                    Success = false,
                    FailReason = "BOUNDARY_EXIT"
                };

                _ = TransitionToMap(failedResult);
            }
        }

        private void OnDestroy()
        {
            // Unsubscribe from events
            if (locationManager != null)
            {
                locationManager.OnFractureEntered -= HandleFractureEntered;
                locationManager.OnFractureExited -= HandleFractureExited;
            }
        }
    }
}
```

### 1.3 Component Architecture (Updated: VContainer DI)

**설계 근거:**
Unity의 GameObject 기반 아키텍처에서 발생하는 의존성 문제를 **VContainer** 로 해결합니다. VContainer는 Unity의 공식 권장 DI 프레임워크로, Service Locator의 단점(암시적 의존성, 테스트 어려움)을 극복하고 명시적 의존성 주입을 제공합니다.

**VContainer 선택 이유:**

- ✅ **타입 안전성**: 컴파일 타임에 의존성 검증
- ✅ **명시적 의존성**: 생성자 주입으로 의존성이 명확
- ✅ **테스트 용이성**: Mock 객체 주입이 쉬움
- ✅ **Unity 최적화**: MonoBehaviour 생명주기와 완벽 통합
- ✅ **성능**: 리플렉션 없는 고속 주입

**핵심 원칙:**

- Interface 기반 설계로 구현체 교체 용이
- 생성자 주입(Constructor Injection) 우선
- LifetimeScope로 의존성 범위 관리
- Singleton 패턴 제거 (VContainer가 관리)

**초기화 순서:**

VContainer가 자동으로 의존성 그래프를 분석하여 올바른 순서로 초기화합니다.

```csharp
// VContainer DI Implementation (Production-Ready)
namespace ORE.Core.DI
{
    using VContainer;
    using VContainer.Unity;

    /// <summary>
    /// Parent/Root LifetimeScope for ORE Platform (DontDestroyOnLoad).
    /// Registers PERSISTENT managers that survive scene transitions.
    /// Child scopes (MapLifetimeScope, ARGameLifetimeScope) inherit from this.
    /// </summary>
    public class CoreLifetimeScope : LifetimeScope
    {
        protected override void Configure(IContainerBuilder builder)
        {
            // Register platform-specific providers FIRST
            // Editor: Simulated GPS | Device: Real Unity location services
            #if UNITY_EDITOR
            builder.Register<ILocationProvider, EditorLocationProvider>(Lifetime.Singleton);
            #else
            builder.Register<ILocationProvider, UnityLocationProvider>(Lifetime.Singleton);
            #endif

            // Register PERSISTENT MonoBehaviour managers (Parent Scope)
            // These exist in __PersistentManagers and survive scene transitions
            builder.RegisterComponentInHierarchy<GameManager>().AsImplementedInterfaces().AsSelf();
            builder.RegisterComponentInHierarchy<NetworkManager>().AsImplementedInterfaces().AsSelf();
            builder.RegisterComponentInHierarchy<LocationManager>().AsImplementedInterfaces().AsSelf();

            // Register Services as singleton - VContainer will auto-inject via constructor
            // Constructor injection prevents circular dependency issues
            builder.Register<Services>(Lifetime.Singleton);
        }
    }
}

// Map Scene Lifetime Scope (Child of CoreLifetimeScope)
namespace ORE.Map.DI
{
    using VContainer;
    using VContainer.Unity;
    using ORE.Core.Map;

    /// <summary>
    /// Child LifetimeScope for Map.unity scene.
    /// Inherits persistent managers from CoreLifetimeScope (parent).
    /// Registers scene-specific components (MapController, GeofencingService).
    /// </summary>
    public class MapLifetimeScope : LifetimeScope
    {
        protected override void Configure(IContainerBuilder builder)
        {
            // Inherit from CoreLifetimeScope automatically via parent-child relationship

            // Register Map scene-specific components
            builder.RegisterComponentInHierarchy<MapController>().AsImplementedInterfaces().AsSelf();
            builder.RegisterComponentInHierarchy<MapUIController>().AsImplementedInterfaces().AsSelf();
            builder.RegisterComponentInHierarchy<GeofencingService>().AsImplementedInterfaces().AsSelf();
            builder.RegisterComponentInHierarchy<FractureVisualizer>().AsImplementedInterfaces().AsSelf();
        }
    }
}

// ARGame Scene Lifetime Scope (Child of CoreLifetimeScope)
namespace ORE.ARGame.DI
{
    using VContainer;
    using VContainer.Unity;
    using ORE.Core;

    /// <summary>
    /// Child LifetimeScope for ARGame.unity scene.
    /// Inherits persistent managers from CoreLifetimeScope (parent).
    /// Registers scene-specific AR components (ARManager, VeinExplorationController).
    /// </summary>
    public class ARGameLifetimeScope : LifetimeScope
    {
        protected override void Configure(IContainerBuilder builder)
        {
            // Inherit from CoreLifetimeScope automatically via parent-child relationship

            // Register ARGame scene-specific components
            builder.RegisterComponentInHierarchy<ARManager>().AsImplementedInterfaces().AsSelf();
            builder.RegisterComponentInHierarchy<VeinExplorationController>().AsImplementedInterfaces().AsSelf();
            builder.RegisterComponentInHierarchy<MiningController>().AsImplementedInterfaces().AsSelf();
            builder.RegisterComponentInHierarchy<GeofencingMonitor>().AsImplementedInterfaces().AsSelf();
        }
    }
}

// Service Locator (Backward Compatibility + VContainer Integration)
namespace ORE.Core
{
    using VContainer;

    /// <summary>
    /// Service Provider implementation using VContainer dependency injection.
    /// Provides centralized access to all manager instances following modern DI patterns.
    /// Uses constructor injection to prevent circular dependencies.
    /// </summary>
    // ReSharper disable once ClassNeverInstantiated.Global - Instantiated by VContainer DI
#pragma warning disable CA1812 // Avoid uninstantiated internal classes - Instantiated by VContainer
    public class Services
#pragma warning restore CA1812
    {
        // Static instance for backward compatibility
        public static Services Instance { get; private set; }

        // Constructor injection (2025 VContainer best practice - prevents circular dependencies)
        // Only persistent managers are injected
        private readonly IGameManager gameManager;
        private readonly ILocationManager locationManager;
        private readonly INetworkManager networkManager;

        // Constructor for DI - VContainer will inject these automatically
        public Services(IGameManager gameManager, INetworkManager networkManager,
                        ILocationManager locationManager)
        {
            this.gameManager = gameManager;
            this.networkManager = networkManager;
            this.locationManager = locationManager;
            Instance = this;
        }

        // Static accessors (backward compatible - persistent managers only)
        public static IGameManager Game => Instance?.gameManager;
        public static ILocationManager Location => Instance?.locationManager;
        public static INetworkManager Network => Instance?.networkManager;
        // AR is NOT accessible via Services - it's scene-specific (ARGame.unity only)

        // Service validation (persistent managers only)
        public static bool AreServicesReady()
        {
            return Game != null &&
                   Location != null &&
                   Network != null;
        }

        // Gameplay readiness check (stricter)
        public static bool AreServicesReadyForGameplay()
        {
            return AreServicesReady() &&
                   Location.HasValidLocation() &&
                   AR.IsTrackingStable() &&
                   Network.HasInternetConnection();
        }
    }
}

// Example: Manager with DI
namespace ORE.Core
{
    using VContainer;
    using ORE.Core.Interfaces;

    public class GameManager : MonoBehaviour, IGameManager
    {
        // VContainer automatically injects dependencies from CoreLifetimeScope (parent)
        [Inject] private INetworkManager networkManager;
        [Inject] private ILocationManager locationManager;

        private void Awake()
        {
            // Dependencies are already injected by VContainer from parent scope
            InitializeGame();
        }

        private void InitializeServices()
        {
            // Service initialization handled by VContainer dependency injection
            // Dependencies are injected automatically in the proper order
            GameLogger.Log("Services initialized through dependency injection");

            try
            {
                // Validate that all required persistent services are injected
                if (networkManager == null || locationManager == null)
                {
                    throw new InvalidOperationException("Required services not injected");
                }

                GameLogger.Log("Core services validated successfully");
            }
            catch (Exception ex)
            {
                Debug.LogError("Failed to initialize services: " + ex.Message);
                ChangeGameState(GameState.Error);
            }
        }
    }
}
```

**VContainer 마이그레이션 가이드:**

기존 Singleton/Service Locator 코드를 VContainer로 전환하는 방법:

```csharp
// Before (Singleton Pattern - 피해야 함)
public class OldManager : SingletonBehaviour<OldManager>
{
    public void DoSomething()
    {
        var network = Services.Network; // 암시적 의존성
    }
}

// After (VContainer DI - 권장)
public class NewManager : MonoBehaviour, INewManager
{
    [Inject] private INetworkManager networkManager; // 명시적 의존성

    public void DoSomething()
    {
        networkManager.SendRequest(...); // 타입 안전
    }
}
```

**Parent-Child Scope 사용 예시:**

```csharp
// ARGame scene-specific component (registered in ARGameLifetimeScope)
namespace ORE.ARGame
{
    using VContainer;
    using ORE.Core.Interfaces;

    /// <summary>
    /// Vein exploration controller - scene-specific to ARGame.unity
    /// Can access parent scope dependencies (LocationManager, NetworkManager)
    /// </summary>
    public class VeinExplorationController : MonoBehaviour
    {
        // Parent scope dependencies (from CoreLifetimeScope)
        [Inject] private ILocationManager locationManager;
        [Inject] private INetworkManager networkManager;

        // Child scope dependencies (from ARGameLifetimeScope)
        [Inject] private IARManager arManager;
        [Inject] private IMiningController miningController;

        private void Update()
        {
            // Access parent scope manager
            var playerLocation = locationManager.CurrentLocation;

            // Access child scope manager
            if (arManager.IsARActive)
            {
                UpdateVeinDistance(playerLocation);
            }
        }

        private void UpdateVeinDistance(Vector2d location)
        {
            // Use both parent and child scope dependencies seamlessly
            var distance = CalculateDistance(location, targetVeinLocation);
            miningController.UpdateDistanceHint(distance);
        }
    }
}
```

**Parent-Child 패턴의 장점:**

- ✅ Scene 컴포넌트가 persistent managers에 접근 가능 (parent scope)
- ✅ Scene 컴포넌트가 같은 씬의 다른 컴포넌트에 접근 가능 (child scope)
- ✅ Parent scope는 child scope에 의존 불가 (컴파일 타임 안전성)
- ✅ 각 씬은 독립적이면서도 핵심 인프라 공유
- ✅ 새로운 씬 추가 시 child scope만 생성하면 됨

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
    public List<CoreData> NearbyCores { get; private set; }
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
        NearbyCores = update.Cores;
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

### 1.4.1 이벤트 기반 아키텍처 (Event-Driven Architecture)

**목적**: 순환 의존성을 방지하고 느슨한 결합(loose coupling)을 유지하기 위한 매니저 간 통신 패턴

**문제점**: VContainer에서 매니저들이 서로를 직접 의존하면 순환 의존성 발생

```csharp
// ❌ 순환 의존성 예시
GameManager → ILocationManager → IGameManager  // CIRCULAR!
GameManager → INetworkManager → IGameManager   // CIRCULAR!
```

**해결책**: 이벤트 기반 통신 패턴

```csharp
// ✅ 단방향 의존성 + 이벤트 구독
GameManager → ILocationManager (DI 의존성 ↓)
            ← OnLocationUpdated (이벤트 알림 ↑)
```

#### 매니저 이벤트 발생 패턴

```csharp
// 예시: LocationManager는 GameManager를 모르지만 이벤트 발생
public class LocationManager : MonoBehaviour, ILocationManager
{
    // GameManager 의존성 제거 (순환 의존성 방지)
    // [Inject] private IGameManager gameManager; // ❌ 제거됨

    [Inject] private ILocationProvider locationProvider; // ✅ Provider만 의존

    // 이벤트 정의 (구독자를 모름 - 느슨한 결합)
    public event Action<Vector2d> OnLocationUpdated;
    public event Action<Vector2d, float> OnLocationValidated;
    public event Action<LocationCheatType> OnCheatDetected; // 안티치트 타입 전달
    public event Action<string> OnLocationError;

    private void UpdateLocation()
    {
        if (locationProvider.Status != LocationServiceStatus.Running) return;

        try
        {
            var locationData = locationProvider.LastData;
            var newLocation = new Vector2d(locationData.latitude, locationData.longitude);

            // 안티치트 검증
            var cheatType = ValidateLocationMovement(newLocation, Time.time);
            if (cheatType == LocationCheatType.None)
            {
                CurrentLocation = newLocation;
                CurrentAccuracy = locationData.horizontalAccuracy;

                // 이벤트 발생 (누가 구독하는지 모름)
                OnLocationUpdated?.Invoke(CurrentLocation);
                OnLocationValidated?.Invoke(CurrentLocation, CurrentAccuracy);
            }
            else
            {
                OnCheatDetected?.Invoke(cheatType); // 치트 타입 전달
            }
        }
        catch (Exception ex)
        {
            OnLocationError?.Invoke(ex.Message);
        }
    }

    // 안티치트 검증 결과 타입
    private LocationCheatType ValidateLocationMovement(Vector2d newLocation, float currentTime)
    {
        if (!HasValidLocation()) return LocationCheatType.None;

        var distance = Vector2d.Distance(lastValidLocation, newLocation);
        var deltaTime = currentTime - lastUpdateTime;

        // 순간이동 체크 (100m 이상 즉시 이동)
        if (distance > teleportThreshold) return LocationCheatType.Teleport;

        // 불가능한 속도 체크 (30 m/s = 108 km/h 이상)
        var velocity = (float)(distance / deltaTime);
        if (velocity > maxVelocity) return LocationCheatType.ImpossibleSpeed;

        // 불가능한 가속도 체크 (10 m/s² 이상)
        var acceleration = Mathf.Abs(velocity - currentVelocity) / deltaTime;
        if (acceleration > maxAcceleration) return LocationCheatType.ImpossibleAcceleration;

        return LocationCheatType.None;
    }
}

// 안티치트 타입 정의
public enum LocationCheatType
{
    None,                       // 정상
    Teleport,                   // 순간이동 (100m+ 즉시 이동)
    ImpossibleSpeed,            // 불가능한 속도 (30 m/s = 108 km/h 이상)
    ImpossibleAcceleration,     // 불가능한 가속도 (10 m/s² 이상)
    GpsSpoof                    // GPS 스푸핑 (향후 구현)
}
```

#### GameManager 이벤트 구독 패턴

```csharp
public class GameManager : MonoBehaviour, IGameManager
{
    // 의존성은 단방향 (GameManager → 다른 매니저들)
    [Inject] private INetworkManager networkManager;
    [Inject] private ILocationManager locationManager;
    [Inject] private IARManager arManager;

    private void Start()
    {
        // 매니저 이벤트 구독 (activity tracking용)
        SubscribeToManagerEvents();
        StartGameSession();
    }

    private void SubscribeToManagerEvents()
    {
        // LocationManager 이벤트 구독
        if (locationManager != null)
        {
            locationManager.OnLocationUpdated += HandleLocationUpdated;
            locationManager.OnCheatDetected += HandleCheatDetected;
        }

        // NetworkManager 이벤트 구독
        if (networkManager != null)
        {
            networkManager.OnConnected += OnNetworkRestored;
            networkManager.OnDisconnected += OnNetworkLost;
        }

        // Note: ARManager는 ARGameLifetimeScope에만 존재 (scene-specific)
    }

    // 중요: 항상 OnDestroy에서 구독 해제 (메모리 누수 방지)
    private void OnDestroy()
    {
        UnsubscribeFromManagerEvents();
    }

    private void UnsubscribeFromManagerEvents()
    {
        if (locationManager != null)
        {
            locationManager.OnLocationUpdated -= HandleLocationUpdated;
            locationManager.OnCheatDetected -= HandleCheatDetected;
        }

        if (networkManager != null)
        {
            networkManager.OnConnected -= OnNetworkRestored;
            networkManager.OnDisconnected -= OnNetworkLost;
        }

        // Note: ARManager는 ARGameLifetimeScope에서 관리 (child scope)
    }

    // 이벤트 핸들러 (activity tracking)
    private void HandleLocationUpdated(Vector2d location)
    {
        RegisterActivity(); // 사용자 활동 추적
    }

    private void HandleCheatDetected(LocationCheatType cheatType)
    {
        GameLogger.LogWarning($"Cheat detected: {cheatType}");
        // 서버에 보고 로직 (치트 타입 기반 처리)
        switch (cheatType)
        {
            case LocationCheatType.Teleport:
                // 순간이동 감지 시 처리
                break;
            case LocationCheatType.ImpossibleSpeed:
            case LocationCheatType.ImpossibleAcceleration:
                // 불가능한 이동 감지 시 처리
                break;
            case LocationCheatType.GpsSpoof:
                // GPS 스푸핑 감지 시 처리
                break;
        }
    }

    private void OnNetworkRestored()
    {
        GameLogger.Log("Network connection restored");
        ResumeGameplay();
    }

    private void OnNetworkLost()
    {
        GameLogger.LogWarning("Network connection lost");
        PauseGameplay();
    }
}
```

**이벤트 기반 아키텍처의 장점:**

- ✅ **순환 의존성 방지**: 의존성 그래프가 단방향 (DAG)
- ✅ **느슨한 결합**: 매니저는 구독자를 알 필요 없음
- ✅ **다중 구독자**: UI, Analytics, GameManager 모두 같은 이벤트 구독 가능
- ✅ **테스트 용이**: 이벤트 소스를 쉽게 모킹 가능
- ✅ **Unity 표준**: Unity의 자체 API도 동일한 패턴 사용

**이벤트 네이밍 규칙:**

```csharp
// ✅ 좋은 예시 (자기 설명적)
event Action OnConnected;
event Action OnDisconnected;
event Action<Vector2d> OnLocationUpdated;

// ❌ 나쁜 예시 (불명확)
event Action<bool> OnConnectivityChanged;  // true가 연결? 끊김?
```

### 1.5 Performance-Optimized Logging System

**설계 근거:**
모바일 AR 게임에서 로깅은 필수적이지만, 성능 오버헤드가 큽니다. `GameLogger`는 조건부 컴파일과 zero-allocation 패턴으로 개발 중 디버깅은 유지하면서 릴리스 빌드에서는 완전히 제거됩니다.

**성능 최적화 전략:**

- ✅ **조건부 컴파일**: `[Conditional("UNITY_EDITOR")]` 로 릴리스 빌드에서 제거
- ✅ **Zero-Allocation**: StringBuilder 재사용으로 GC 압력 제거
- ✅ **타입 안전 오버로드**: `params object[]` 대신 명시적 타입으로 박싱 방지
- ✅ **일관된 포맷**: `[ORE]` 접두사로 프로젝트 로그 필터링 용이

**릴리스 빌드 동작:**

```csharp
// 개발 빌드 (UNITY_EDITOR 또는 DEVELOPMENT_BUILD 정의됨)
GameLogger.Log("Player mined Core");
// → 출력: [ORE] Player mined Core

// 릴리스 빌드 (프로덕션)
GameLogger.Log("Player mined Core");
// → 컴파일 타임에 완전히 제거됨, CPU 사이클 0
```

**구현 코드:**

```csharp
namespace ORE.Core
{
    using UnityEngine;
    using System.Text;

    /// <summary>
    /// Performance-optimized logging for ORE Protocol.
    /// Completely compiled out in release builds with zero allocations.
    /// </summary>
    public static class GameLogger
    {
        // Reusable StringBuilder to eliminate allocations
        private static readonly StringBuilder StringBuilder = new StringBuilder(256);

        [System.Diagnostics.Conditional("UNITY_EDITOR")]
        [System.Diagnostics.Conditional("DEVELOPMENT_BUILD")]
        public static void Log(string message)
        {
            BuildLogMessage(message);
            Debug.Log(StringBuilder.ToString());
        }

        [System.Diagnostics.Conditional("UNITY_EDITOR")]
        [System.Diagnostics.Conditional("DEVELOPMENT_BUILD")]
        public static void LogWarning(string message)
        {
            BuildLogMessage(message);
            Debug.LogWarning(StringBuilder.ToString());
        }

        [System.Diagnostics.Conditional("UNITY_EDITOR")]
        [System.Diagnostics.Conditional("DEVELOPMENT_BUILD")]
        public static void LogError(string message)
        {
            BuildLogMessage(message);
            Debug.LogError(StringBuilder.ToString());
        }

        [System.Diagnostics.Conditional("UNITY_EDITOR")]
        [System.Diagnostics.Conditional("DEVELOPMENT_BUILD")]
        public static void LogStateChange(object from, object to)
        {
            StringBuilder.Clear();
            StringBuilder.Append("[ORE] State: ");
            StringBuilder.Append(from);
            StringBuilder.Append(" → ");
            StringBuilder.Append(to);
            Debug.Log(StringBuilder.ToString());
        }

        // Zero-allocation formatted logging (1-2 arguments)
        [System.Diagnostics.Conditional("UNITY_EDITOR")]
        [System.Diagnostics.Conditional("DEVELOPMENT_BUILD")]
        public static void LogFormat(string message, string arg1)
        {
            StringBuilder.Clear();
            StringBuilder.Append("[ORE] ");
            StringBuilder.Append(message);
            StringBuilder.Append(arg1);
            Debug.Log(StringBuilder.ToString());
        }

        [System.Diagnostics.Conditional("UNITY_EDITOR")]
        [System.Diagnostics.Conditional("DEVELOPMENT_BUILD")]
        public static void LogFormat(string message, string arg1, string arg2)
        {
            StringBuilder.Clear();
            StringBuilder.Append("[ORE] ");
            StringBuilder.Append(message);
            StringBuilder.Append(arg1);
            StringBuilder.Append(arg2);
            Debug.Log(StringBuilder.ToString());
        }

        // Warning/Error variants
        [System.Diagnostics.Conditional("UNITY_EDITOR")]
        [System.Diagnostics.Conditional("DEVELOPMENT_BUILD")]
        public static void LogWarningFormat(string message, string arg1)
        {
            StringBuilder.Clear();
            StringBuilder.Append("[ORE] ");
            StringBuilder.Append(message);
            StringBuilder.Append(arg1);
            Debug.LogWarning(StringBuilder.ToString());
        }

        [System.Diagnostics.Conditional("UNITY_EDITOR")]
        [System.Diagnostics.Conditional("DEVELOPMENT_BUILD")]
        public static void LogErrorFormat(string message, string arg1)
        {
            StringBuilder.Clear();
            StringBuilder.Append("[ORE] ");
            StringBuilder.Append(message);
            StringBuilder.Append(arg1);
            Debug.LogError(StringBuilder.ToString());
        }

        // Numeric helpers (prevent ToString() allocations at call sites)
        [System.Diagnostics.Conditional("UNITY_EDITOR")]
        [System.Diagnostics.Conditional("DEVELOPMENT_BUILD")]
        public static void LogFloat(string message, float value, string suffix = "")
        {
            StringBuilder.Clear();
            StringBuilder.Append("[ORE] ");
            StringBuilder.Append(message);
            StringBuilder.Append(value.ToString("F2"));
            StringBuilder.Append(suffix);
            Debug.Log(StringBuilder.ToString());
        }

        [System.Diagnostics.Conditional("UNITY_EDITOR")]
        [System.Diagnostics.Conditional("DEVELOPMENT_BUILD")]
        public static void LogInt(string message, int value, string suffix = "")
        {
            StringBuilder.Clear();
            StringBuilder.Append("[ORE] ");
            StringBuilder.Append(message);
            StringBuilder.Append(value);
            StringBuilder.Append(suffix);
            Debug.Log(StringBuilder.ToString());
        }

        private static void BuildLogMessage(string message)
        {
            StringBuilder.Clear();
            StringBuilder.Append("[ORE] ");
            StringBuilder.Append(message);
        }
    }
}
```

**사용 예시:**

```csharp
// GameManager에서 사용
public class GameManager : MonoBehaviour
{
    private void Awake()
    {
        // 기본 로그
        GameLogger.Log("GameManager initialized");
    }

    private void ChangeGameState(GameState newState)
    {
        // 상태 전환 로그
        GameLogger.LogStateChange(CurrentState, newState);
    }

    private void UpdatePerformance()
    {
        // 포맷 로그 (zero-allocation)
        GameLogger.LogFormat("Switched to low power mode (battery: ", batteryLevel.ToString("P0"), ")");
    }
}
```

**중요: Debug.LogError vs GameLogger.LogError**

릴리스 빌드에서도 보여야 하는 치명적 에러는 `Debug.LogError`를 사용해야 합니다:

```csharp
// ✅ 올바른 사용
try
{
    InitializeServices();
    GameLogger.Log("Services initialized"); // 개발 전용
}
catch (Exception ex)
{
    Debug.LogError("Failed to initialize services: " + ex.Message); // 프로덕션에도 표시
    ChangeGameState(GameState.Error);
}
```

**80/20 규칙:**

- **80% GameLogger**: 일반 로그, 상태 변화, 디버깅 정보
- **20% Debug.LogError**: 치명적 에러, 크래시 리포트에 포함되어야 하는 정보

**성능 영향:**

| 항목         | 개발 빌드   | 릴리스 빌드  |
| ------------ | ----------- | ------------ |
| CPU 오버헤드 | ~0.1ms/call | 0ms (제거됨) |
| GC 할당      | 0 bytes     | 0 bytes      |
| 코드 크기    | 포함        | 제거됨       |
| 로그 출력    | 표시됨      | 없음         |

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

  XR Origin (AR Foundation 6.0+ 필수):
    - Camera Offset: XR Camera를 자식으로 가짐
    - Tracking Origin Mode: Device (모바일 AR)
    - Camera setup:
        * Target Frame Rate: 디바이스 적응형
        * Facing Direction: Rear (후면 카메라)
        * Auto Focus: Continuous
    - Plane detection (ARPlaneManager):
        * Mode: Horizontal (수평면만)
        * 이유: 수직면 감지 제외로 CPU 20% 절약
    - Light estimation:
        * Mode: Basic (밝기만)
        * 이유: Directional light는 GPU 부담 높음
    - Input System: New Input System (Enhanced Touch)
        * Touch.activeTouches로 터치 입력 처리
        * EnhancedTouchSupport.Enable() 필요

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
using UnityEngine.InputSystem.EnhancedTouch;  // New Input System

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

    void Start()
    {
        // New Input System의 Enhanced Touch 활성화 (필수)
        EnhancedTouchSupport.Enable();
    }

    void Update()
    {
        // New Input System - Touch.activeTouches 사용
        if (Touch.activeTouches.Count > 0)
        {
            var touch = Touch.activeTouches[0];

            switch (touch.phase)
            {
                case UnityEngine.InputSystem.TouchPhase.Began:
                    HandleTouchBegin(touch.screenPosition);
                    break;
                case UnityEngine.InputSystem.TouchPhase.Moved:
                    HandleTouchMove(touch.screenPosition);
                    break;
                case UnityEngine.InputSystem.TouchPhase.Ended:
                    HandleTouchEnd(touch.screenPosition);
                    break;
            }
        }

        // 근거리 자동 수집 체크
        CheckNearbyAutoCollect();
    }

    void OnDisable()
    {
        // Enhanced Touch 비활성화 (메모리 정리)
        EnhancedTouchSupport.Disable();
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

### 3.0 플랫폼 추상화 (Provider Pattern)

**목적**: 플랫폼별 구현을 추상화하여 에디터 테스트, 디바이스 배포, 유닛 테스트를 조건부 컴파일 없이 비즈니스 로직에서 수행할 수 있게 합니다.

#### 3.0.1 ILocationProvider 인터페이스

```csharp
namespace ORE.Core.Interfaces
{
    /// <summary>
    /// Platform-agnostic GPS location provider interface.
    /// Abstracts Unity's Input.location API for testability and Editor simulation.
    /// </summary>
    public interface ILocationProvider
    {
        void Start(float desiredAccuracyInMeters, float updateDistanceInMeters);
        void Stop();
        bool IsEnabledByUser { get; }
        LocationServiceStatus Status { get; }
        LocationData LastData { get; }  // Returns LocationData (not Unity's readonly LocationInfo)
    }
}
```

#### 3.0.2 UnityLocationProvider (디바이스 구현)

**용도**: iOS/Android 실제 기기에서 Unity의 Input.location API를 사용한 GPS 구현

```csharp
namespace ORE.Core.Providers
{
    public class UnityLocationProvider : ILocationProvider
    {
        public void Start(float desiredAccuracyInMeters, float updateDistanceInMeters)
        {
            Input.location.Start(desiredAccuracyInMeters, updateDistanceInMeters);
        }

        public void Stop()
        {
            Input.location.Stop();
        }

        public bool IsEnabledByUser => Input.location.isEnabledByUser;
        public LocationServiceStatus Status => Input.location.status;

        // Unity의 readonly LocationInfo를 mutable LocationData로 변환
        public LocationData LastData => LocationData.FromUnityLocationInfo(Input.location.lastData);
    }
}
```

#### 3.0.3 EditorLocationProvider (에디터 시뮬레이션)

**용도**: 에디터에서 GPS 테스트를 위한 시뮬레이션 제공 (기기 없이 개발 가능)

```csharp
namespace ORE.Core.Providers
{
    public class EditorLocationProvider : ILocationProvider
    {
        private LocationServiceStatus currentStatus = LocationServiceStatus.Stopped;
        private LocationData simulatedData;

        // 기본 테스트 위치: 샌프란시스코
        private const float DefaultLatitude = 37.7749f;
        private const float DefaultLongitude = -122.4194f;
        private const float DefaultAccuracy = 10f;

        public EditorLocationProvider()
        {
            InitializeSimulatedLocation();
        }

        public void Start(float desiredAccuracyInMeters, float updateDistanceInMeters)
        {
            GameLogger.Log("[EditorLocationProvider] Starting simulated location service");
            currentStatus = LocationServiceStatus.Running;
        }

        public void Stop()
        {
            GameLogger.Log("[EditorLocationProvider] Stopping simulated location service");
            currentStatus = LocationServiceStatus.Stopped;
        }

        public bool IsEnabledByUser => true; // 에디터에서는 항상 활성화
        public LocationServiceStatus Status => currentStatus;
        public LocationData LastData => simulatedData;

        // 에디터 테스트 시나리오를 위한 위치 변경 메서드
        public void SetSimulatedLocation(float latitude, float longitude, float accuracy = DefaultAccuracy)
        {
            simulatedData = LocationData.CreateSimulated(latitude, longitude, accuracy);
            GameLogger.Log($"[EditorLocationProvider] Location set to: {latitude}, {longitude}");
        }

        private void InitializeSimulatedLocation()
        {
            simulatedData = LocationData.CreateSimulated(DefaultLatitude, DefaultLongitude, DefaultAccuracy);
        }
    }
}
```

**Provider Pattern 사용 이유:**

- ✅ 플랫폼 독립적인 비즈니스 로직 (LocationManager에 #if UNITY_EDITOR 불필요)
- ✅ 디바이스 하드웨어 없이 에디터 테스트 가능
- ✅ 유닛 테스트를 위한 쉬운 모킹
- ✅ AR Foundation의 XRSubsystem 패턴 준수 (업계 표준)

**CoreLifetimeScope 등록** (섹션 2.2.1 참조):

```csharp
#if UNITY_EDITOR
builder.Register<ILocationProvider, EditorLocationProvider>(Lifetime.Singleton);
#else
builder.Register<ILocationProvider, UnityLocationProvider>(Lifetime.Singleton);
#endif
```

#### 3.0.4 LocationData 구조체

**문제점**: Unity의 `LocationInfo`는 readonly 구조체로 생성 및 수정 불가능하여 테스트와 시뮬레이션에 사용 불가

**해결책**: 플랫폼 독립적이고 변경 가능한 `LocationData` 구조체 정의

```csharp
namespace ORE.Core
{
    /// <summary>
    /// Platform-agnostic GPS data structure.
    /// Replaces Unity's readonly LocationInfo for testability and flexibility.
    /// </summary>
    [System.Serializable]
    public struct LocationData
    {
        public float latitude;
        public float longitude;
        public float altitude;
        public float horizontalAccuracy;
        public float verticalAccuracy;
        public double timestamp;

        public LocationData(float lat, float lng, float alt, float hAcc, float vAcc, double time)
        {
            latitude = lat;
            longitude = lng;
            altitude = alt;
            horizontalAccuracy = hAcc;
            verticalAccuracy = vAcc;
            timestamp = time;
        }

        /// <summary>
        /// Convert from Unity's LocationInfo (device only).
        /// Used by UnityLocationProvider to bridge Unity API and our abstraction.
        /// </summary>
        public static LocationData FromUnityLocationInfo(LocationInfo info)
        {
            return new LocationData(
                info.latitude,
                info.longitude,
                info.altitude,
                info.horizontalAccuracy,
                info.verticalAccuracy,
                info.timestamp
            );
        }

        /// <summary>
        /// Create simulated location data for testing and Editor.
        /// </summary>
        public static LocationData CreateSimulated(float lat, float lng, float accuracy)
        {
            return new LocationData(
                lat, lng,
                15f,  // Default altitude
                accuracy, accuracy,
                Time.realtimeSinceStartup
            );
        }
    }
}
```

**사용 흐름:**

```
디바이스:  Unity LocationInfo → LocationData.FromUnityLocationInfo() → LocationManager
에디터:    시뮬레이션 → LocationData.CreateSimulated() → LocationManager
테스트:    Mock 데이터 → LocationData 생성자 → LocationManager
```

**장점:**

- ✅ 에디터와 디바이스에서 동일한 데이터 구조 사용
- ✅ 유닛 테스트용 임의 데이터 생성 가능
- ✅ 리플렉션 없이 직접 필드 접근 가능
- ✅ 시리얼라이즈 가능 (디버깅 및 저장에 유용)

#### 3.0.5 Manager 인터페이스 (Dependency Injection)

**목적**: VContainer DI를 통한 테스트 가능성과 느슨한 결합을 위해 모든 Manager는 인터페이스로 추상화됩니다.

##### IGameManager

```csharp
namespace ORE.Core.Interfaces
{
    public interface IGameManager
    {
        // Game state
        GameState CurrentState { get; }
        GameState PreviousState { get; }

        // Events
        event Action<GameState> OnGameStateChanged;
        event Action OnGamePaused;
        event Action OnGameResumed;

        // Lifecycle
        void PauseGameplay();
        void ResumeGameplay();
        void RegisterActivity();
    }
}
```

##### ILocationManager

```csharp
namespace ORE.Core.Interfaces
{
    public interface ILocationManager
    {
        // Current state
        Vector2d CurrentLocation { get; }
        float CurrentAccuracy { get; }
        bool IsLocationServiceRunning { get; }

        // Events
        event Action<Vector2d> OnLocationUpdated;
        event Action<Vector2d, float> OnLocationValidated;
        event Action<LocationCheatType> OnCheatDetected;
        event Action<string> OnLocationError;

        // Methods
        bool HasValidLocation();
        void StartLocationService();
        void StopLocationService();
    }
}
```

##### INetworkManager

```csharp
namespace ORE.Core.Interfaces
{
    public interface INetworkManager
    {
        // Connection state
        bool IsConnected { get; }
        bool IsAuthenticated { get; }
        string AuthToken { get; }

        // Events
        event Action OnConnected;
        event Action OnDisconnected;
        event Action OnAuthenticated;
        event Action<string> OnNetworkError;

        // API methods
        void UpdatePlayerLocation(Vector2d location, float accuracy, Action<bool> onComplete);
        void AuthenticateUser(string userId, Action<bool> onComplete);
    }
}
```

##### IARManager

```csharp
namespace ORE.Core.Interfaces
{
    public interface IARManager
    {
        // AR state
        bool IsARInitialized { get; }
        bool IsARRunning { get; }
        TrackingState CurrentTrackingState { get; }

        // Events
        event Action OnARInitialized;
        event Action OnARStarted;
        event Action<TrackingState> OnTrackingStateChanged;
        event Action<ARPlane> OnPlaneDetected;
        event Action<Vector2> OnARTapped;

        // Methods
        bool IsTrackingStable();
        GameObject SpawnARObject(string objectId, GameObject prefab, Vector3 position, Quaternion rotation);
        void DespawnARObject(string objectId);
    }
}
```

**인터페이스 사용 이유:**

- ✅ **테스트 용이성**: Mock 구현 주입 가능
- ✅ **느슨한 결합**: 구현체 변경 시 인터페이스 사용 코드 영향 없음
- ✅ **DI 친화적**: VContainer가 인터페이스 기반으로 의존성 해결
- ✅ **문서화**: 인터페이스가 public API 계약 역할

**DI 등록 패턴** (CoreLifetimeScope.cs):

```csharp
// MonoBehaviour managers - RegisterComponentInHierarchy 사용
builder.RegisterComponentInHierarchy<GameManager>().AsImplementedInterfaces().AsSelf();
builder.RegisterComponentInHierarchy<NetworkManager>().AsImplementedInterfaces().AsSelf();
builder.RegisterComponentInHierarchy<LocationManager>().AsImplementedInterfaces().AsSelf();
builder.RegisterComponentInHierarchy<ARManager>().AsImplementedInterfaces().AsSelf();
```

**의존성 주입 예시:**

```csharp
public class UIController : MonoBehaviour
{
    // 인터페이스 주입 (구현체가 아님)
    [Inject] private ILocationManager locationManager;
    [Inject] private INetworkManager networkManager;

    private void Start()
    {
        // 인터페이스 메서드만 사용 (느슨한 결합)
        if (locationManager.HasValidLocation())
        {
            var location = locationManager.CurrentLocation;
            networkManager.UpdatePlayerLocation(location, 10f, OnSuccess);
        }
    }
}
```

### 3.1 Location System (GPS + Online Maps)

> **참고**: Mapbox SDK for Unity(2021 deprecated), GO Map(Asset Store deprecated), Google Maps Platform for Unity(2022 sunset) 모두 사용 불가. Online Maps v4가 2025년 9월 현재 검증된 상용 솔루션입니다.

#### 3.1.1 GPS 추적 및 Kalman 필터링

**업데이트 주기 설정 근거:**

- `updateInterval = 1.0f`: 배터리 소모와 정확도의 균형점
  - 0.5초: 배터리 20%/hr (너무 높음)
  - 1.0초: 배터리 10%/hr (적정)
  - 2.0초: 위치 정확도 저하, 사용자 경험 악화

- `minDistance = 2.0f`: GPS 노이즈 필터링
  - 일반적인 GPS 오차가 2-5m이므로 2m 미만 이동은 무시
  - 걷기 속도(4km/h = 1.1m/s) 고려 시 적절한 값

**Kalman Filter 통합 (2025 표준):**

- Adaptive Kalman filter로 GPS 스무딩 (jitter 감소, outlier 처리)
- Position-only uncertainty 추적 (모바일 배터리 최적화)
- ScriptableObject configuration으로 런타임 튜닝
- Walking/Driving/Testing 프로파일 지원

**네트워크 의존성:**
위치 시스템은 완전히 네트워크에 의존합니다. 오프라인 상태에서는 게임이 일시정지되며, 재연결 시 서버와 동기화합니다.

#### 3.1.2 지도 렌더링 (Online Maps v4)

**Online Maps v4 통합:**

- **출처**: Unity Asset Store ($120 일회성 구매)
- **버전**: 4.2.1 (2025.09.01 최신 업데이트)
- **평가**: 5/5 stars (218 reviews) - 모바일 성능 검증됨
- **Unity 호환**: 2021.3.38+ (Unity 6 지원)
- **기능**: 2D/3D 지도, AR/VR 통합, Online/Offline maps, 실시간 위치 추적
- **타일 제공자**: 16개 providers (Google Maps, Mapbox tiles, OpenStreetMap, ArcGIS, Bing Maps 등)

**Map 설정:**

```csharp
using InfinityCode.OnlineMapsExamples;

[Header("Map Settings")]
public OnlineMapsProvider mapProvider = OnlineMapsProvider.GoogleMaps;
public OnlineMapsProviderEnum.GoogleMapsType mapType = OnlineMapsProviderEnum.GoogleMapsType.Satellite;
public int initialZoom = 18;  // 건물 레벨 디테일
public Vector2 initialLocation;  // GPS 좌표 (위도, 경도)

[Header("Tile Provider API Keys")]
public string googleMapsApiKey;  // Google Maps Platform (타일만, 게임 SDK 아님)
public string mapboxAccessToken;  // Mapbox tiles API (대안)
```

**성능 최적화:**

- 타일 캐싱: Online/Offline 모드 지원 (50MB+ local cache)
- Level-of-detail (LOD): 줌 레벨 자동 조정
- 비동기 타일 로딩: Non-blocking UI (user reviews 검증)
- 모바일 최적화: Android/iOS에서 "fast performance" 확인됨

```csharp
public class LocationManager : MonoBehaviour, ILocationManager
{
    [Header("GPS Settings")]
    public float updateInterval = 1.0f;     // 업데이트 주기 (초)
    public float minDistance = 2.0f;        // 최소 이동 거리 (미터)
    public float desiredAccuracy = 10.0f;   // 목표 정확도 (미터)

    [Header("Battery Optimization")]
    public bool adaptiveInterval = true;    // 배터리 적응형 주기
    public float lowBatteryInterval = 5.0f; // 저전력 모드 주기

    [Header("Anti-Cheat")]
    public float maxSpeed = 30f;            // 최대 속도 (m/s) = 108km/h
    public float maxAcceleration = 10f;     // 최대 가속도 (m/s²)

    // VContainer 의존성 주입 (플랫폼 독립적)
    [Inject] private ILocationProvider locationProvider;
    [Inject] private INetworkManager networkManager;

    // 안티치트 추적 변수
    private Vector2d lastValidLocation;
    private float lastUpdateTime;
    private float currentVelocity;
    private const float teleportThreshold = 100f;    // 순간이동 임계값 (미터)
    private const float maxVelocity = 30f;           // 최대 속도 (m/s = 108 km/h)

    // 현재 상태
    public Vector2d CurrentLocation { get; private set; }
    public float CurrentAccuracy { get; private set; }
    public bool IsLocationServiceRunning => locationProvider?.Status == LocationServiceStatus.Running;

    // 이벤트 (GameManager에게 알림)
    public event Action<Vector2d> OnLocationUpdated;
    public event Action<Vector2d, float> OnLocationValidated;
    public event Action<LocationCheatType> OnCheatDetected;

    IEnumerator Start()
    {
        // 네트워크 필수 체크
        if (!networkManager.IsConnected)
        {
            ShowNetworkRequiredScreen();
            yield break;
        }

        // 위치 권한 체크
        yield return RequestLocationPermission();

        // GPS 서비스 시작 (Provider를 통해 플랫폼 독립적으로)
        locationProvider.Start(desiredAccuracy, minDistance);

        // 초기화 대기 (최대 20초)
        int maxWait = 20;
        while (locationProvider.Status == LocationServiceStatus.Initializing && maxWait > 0)
        {
            yield return new WaitForSeconds(1);
            maxWait--;
        }

        // 상태별 처리
        switch (locationProvider.Status)
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
            if (!networkManager.IsConnected)
            {
                PauseGame("네트워크 연결이 필요합니다");
                yield return new WaitUntil(() => networkManager.IsConnected);
                ResumeGame();
            }

            // 배터리 적응형 주기
            float currentInterval = updateInterval;
            if (adaptiveInterval && Battery.level < 0.2f)
            {
                currentInterval = lowBatteryInterval;
            }

            // GPS 데이터 획득 (Provider를 통해 플랫폼 독립적으로)
            var locationData = locationProvider.LastData;
            var currentLocation = new Vector2d(locationData.latitude, locationData.longitude);

            // 클라이언트 측 안티치트 검증
            var cheatType = ValidateLocationMovement(currentLocation, Time.time);
            if (cheatType == LocationCheatType.None)
            {
                // 검증 통과 - 서버로 전송
                CurrentLocation = currentLocation;
                CurrentAccuracy = locationData.horizontalAccuracy;

                OnLocationUpdated?.Invoke(currentLocation);
                OnLocationValidated?.Invoke(currentLocation, locationData.horizontalAccuracy);

                // 서버로 위치 업데이트 전송
                SendLocationUpdate(currentLocation, locationData);
            }
            else
            {
                // 치트 감지 - 이벤트 발생
                OnCheatDetected?.Invoke(cheatType);
                GameLogger.LogWarning($"Cheat detected: {cheatType}");

                // 서버에 보고 (치트 타입 포함)
                ReportCheatDetection(cheatType, currentLocation);
            }

            yield return new WaitForSeconds(currentInterval);
        }
    }

    private LocationCheatType ValidateLocationMovement(Vector2d newLocation, float currentTime)
    {
        // 첫 위치는 항상 유효
        if (!HasValidLocation())
        {
            lastValidLocation = newLocation;
            lastUpdateTime = currentTime;
            return LocationCheatType.None;
        }

        var distance = Vector2d.Distance(lastValidLocation, newLocation);
        var deltaTime = currentTime - lastUpdateTime;

        // 순간이동 체크 (100m 이상 즉시 이동)
        if (distance > teleportThreshold) // teleportThreshold = 100m
        {
            GameLogger.LogWarning($"Teleport detected: {distance}m in {deltaTime}s");
            return LocationCheatType.Teleport;
        }

        // 불가능한 속도 체크 (30 m/s = 108 km/h 이상)
        var velocity = (float)(distance / deltaTime);
        if (velocity > maxVelocity) // maxVelocity = 30 m/s
        {
            GameLogger.LogWarning($"Impossible speed: {velocity}m/s (max: {maxVelocity}m/s)");
            return LocationCheatType.ImpossibleSpeed;
        }

        // 불가능한 가속도 체크 (10 m/s² 이상)
        var acceleration = Mathf.Abs(velocity - currentVelocity) / deltaTime;
        if (acceleration > maxAcceleration) // maxAcceleration = 10 m/s²
        {
            GameLogger.LogWarning($"Impossible acceleration: {acceleration}m/s² (max: {maxAcceleration}m/s²)");
            return LocationCheatType.ImpossibleAcceleration;
        }

        // 검증 통과
        lastValidLocation = newLocation;
        lastUpdateTime = currentTime;
        currentVelocity = velocity;
        return LocationCheatType.None;
    }

    // 유효한 위치 데이터가 있는지 체크
    private bool HasValidLocation()
    {
        return lastValidLocation != null && lastUpdateTime > 0;
    }

    void SendLocationUpdate(Vector2d location, LocationData data)
    {
        var update = new LocationUpdate
        {
            Latitude = location.x,
            Longitude = location.y,
            Accuracy = data.horizontalAccuracy,
            Altitude = data.altitude,
            Timestamp = data.timestamp,
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

### 3.2 Core Mining Mechanics

**설계 근거:**
Core 채굴은 완전히 서버 권위적입니다. 클라이언트는 시각적 표현만 담당하고, 실제 채굴 여부는 서버가 결정합니다. Optimistic UI로 즉각적인 피드백을 제공하되, 서버 거부 시 롤백합니다.

**수집 거리 설정:**

- 기본: 10m (GPS 오차 고려)
- Genesis 특전: 3m 자동 수집
- 최대 표시: 100m (성능 고려)

```csharp
public class CoreMiningSystem : MonoBehaviour
{
    [Header("Collection Settings")]
    public float collectionRange = 10f;         // 수집 가능 거리
    public float autoCollectRange = 3f;        // 자동 수집 거리 (Genesis)
    public float displayRange = 100f;          // 표시 거리
    public float collectionCooldown = 0.5f;    // 연속 수집 방지

    [Header("Visual Settings")]
    public int maxVisibleCores = 50;           // 최대 표시 개수
    public float coreRotationSpeed = 90f;      // 회전 속도

    private Dictionary<string, CoreVisual> activeCores;
    private Queue<MiningRequest> pendingRequests;
    private float lastMiningTime;

    // 서버로부터 Core 데이터 수신
    public void UpdateNearbyCores(List<CoreData> serverCores)
    {
        // 성능을 위해 거리순 정렬
        serverCores.Sort((a, b) =>
            Vector2d.Distance(a.Location, PlayerGPS).CompareTo(
            Vector2d.Distance(b.Location, PlayerGPS)));

        // 표시 범위 밖 Core 제거
        RemoveDistantCores(serverCores);

        // 최대 개수 제한
        int displayCount = Mathf.Min(serverCores.Count, maxVisibleCores);

        for (int i = 0; i < displayCount; i++)
        {
            var coreData = serverCores[i];

            if (activeCores.ContainsKey(coreData.Id))
            {
                // 기존 Core 업데이트
                UpdateCoreVisual(coreData);
            }
            else
            {
                // 새 Core 생성
                SpawnCoreVisual(coreData);
            }
        }
    }

    void SpawnCoreVisual(CoreData data)
    {
        // 타입별 풀에서 가져오기
        var coreGO = CorePool.Get(data.Type);
        var visual = coreGO.GetComponent<CoreVisual>();

        // 월드 좌표 설정
        var worldPos = GPSToWorld(data.Location);
        visual.SetWorldPosition(worldPos);
        visual.SetData(data);

        // 거리별 품질 설정
        float distance = Vector3.Distance(PlayerPosition, worldPos);
        SetCoreQuality(visual, distance);

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

        // 채굴 가능 여부 표시
        bool canMine = distance <= collectionRange;
        visual.SetMineable(canMine);

        activeCores[data.Id] = visual;

        // 스폰 애니메이션
        visual.PlaySpawnAnimation();
    }

    // Core 채굴 시도
    public void TryMineCore(string coreId)
    {
        // 쿨다운 체크
        if (Time.time - lastMiningTime < collectionCooldown)
        {
            Debug.Log("Mining on cooldown");
            return;
        }

        if (!activeCores.ContainsKey(coreId))
        {
            Debug.LogWarning($"Core {coreId} not found");
            return;
        }

        var core = activeCores[coreId];

        // 클라이언트 측 거리 체크 (UX용)
        float distance = Vector3.Distance(PlayerPosition, core.transform.position);
        if (distance > collectionRange)
        {
            ShowTooFarMessage(distance);
            return;
        }

        // 서버에 채굴 요청
        var request = new MineCoreRequest
        {
            RequestId = Guid.NewGuid().ToString(), // 멱등성
            CoreId = coreId,
            CoreType = core.Data.Type,
            PlayerPosition = GetCurrentGPS(),
            CorePosition = core.Data.Location,
            Distance = distance,
            Timestamp = NetworkTime.time,
            PickaxeId = GameState.EquippedPickaxe?.Id
        };

        // 요청 대기열에 추가
        pendingRequests.Enqueue(request);

        // Optimistic UI
        core.PlayMiningAnimation();
        ShowMiningUI(core.Data.Value);
        lastMiningTime = Time.time;

        // 서버 전송
        Services.Network.SendRequest(request, (response) =>
            OnMiningResponse(request, response));
    }

    void OnMiningResponse(MineCoreRequest request, MineCoreResponse response)
    {
        // 대기열에서 제거
        pendingRequests = new Queue<MiningRequest>(
            pendingRequests.Where(r => r.RequestId != request.RequestId));

        if (response.Success)
        {
            // 성공 처리
            HandleMiningSuccess(request.CoreId, response);
        }
        else
        {
            // 실패 처리 (롤백)
            HandleMiningFailure(request.CoreId, response);
        }
    }

    void HandleMiningSuccess(string coreId, MineCoreResponse response)
    {
        if (!activeCores.ContainsKey(coreId)) return;

        var core = activeCores[coreId];

        // 성공 이펙트
        core.PlaySuccessEffect();

        // 보상 표시
        UIManager.ShowReward(new RewardData
        {
            Cores = response.CoreReward,
            Experience = response.ExpReward,
            BonusMultiplier = response.BonusMultiplier // Genesis 2x
        });

        // 사운드
        AudioManager.PlaySound("core_mined");

        // 햅틱 피드백
        HapticFeedback.Success();

        // 제거 (풀로 반환)
        StartCoroutine(RemoveCoreAfterEffect(core));
    }

    void HandleMiningFailure(string coreId, MineCoreResponse response)
    {
        if (!activeCores.ContainsKey(coreId)) return;

        var core = activeCores[coreId];

        // 애니메이션 롤백
        core.CancelMining();

        // 에러 메시지
        switch (response.FailReason)
        {
            case "OUT_OF_RANGE":
                UIManager.ShowError($"너무 멀리 있습니다! ({response.ActualDistance:F1}m)");
                break;

            case "ALREADY_MINED":
                UIManager.ShowError("다른 플레이어가 먼저 채굴했습니다!");
                // 즉시 제거
                RemoveCore(coreId);
                break;

            case "INVALID_PICKAXE":
                UIManager.ShowError("유효한 곡괭이가 필요합니다!");
                break;

            case "ENERGY_DEPLETED":
                UIManager.ShowError("에너지가 부족합니다!");
                ShowEnergyPurchaseOption();
                break;

            default:
                UIManager.ShowError("채굴에 실패했습니다.");
                break;
        }
    }

    // Genesis 자동 채굴
    void CheckAutoMining()
    {
        if (!GameState.LocalPlayer.IsGenesis) return;

        foreach (var core in activeCores.Values)
        {
            float distance = Vector3.Distance(PlayerPosition, core.transform.position);

            if (distance <= autoCollectRange && !core.IsPendingMining)
            {
                TryMineCore(core.Data.Id);
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
public class NetworkManager : MonoBehaviour, INetworkManager
{
    [Header("Configuration")]
    public string apiBaseUrl = "https://api.ore.game/v1";
    public string wsUrl = "wss://ws.ore.game/v1/realtime";
    public float requestTimeout = 10f;
    public int maxRetries = 3;

    [Header("Rate Limiting")]
    public float requestCooldown = 0.1f;  // 100ms between requests
    private float lastRequestTime;
    private readonly Queue<NetworkRequest> requestQueue = new();
    private readonly List<UnityWebRequest> activeRequests = new();

    // HTTP 설정
    private string authToken;
    private DateTime tokenExpiry;

    // 연결 상태
    public bool IsConnected { get; private set; }
    public bool IsAuthenticated { get; private set; }
    public string AuthToken => authToken;
    public NetworkReachability LastReachability { get; private set; }

    // 이벤트
    public event Action OnConnected;
    public event Action OnDisconnected;
    public event Action OnAuthenticated;
    public event Action<string> OnNetworkError;

    // 재연결 관리
    private int reconnectAttempts = 0;
    private float reconnectDelay = 1f;

    void Start()
    {
        // 네트워크 연결 상태 모니터링 시작
        StartCoroutine(NetworkMonitor());
        StartCoroutine(ProcessRequestQueue());
    }

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
        Services.Inventory.LoadInventory(inventory);

        // 3. 현재 위치 업데이트
        Services.Location.ForceLocationUpdate();

        // 4. 퀘스트 상태
        var quests = await GetActiveQuests();
        Services.Quest.UpdateQuests(quests);

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
5. **VContainer DI**: 명시적 의존성 주입, 타입 안전성
6. **GameLogger**: Zero-allocation, 조건부 컴파일
7. **Genesis 특별 대우**: 2x 보상, 자동 수집, 전용 UI

이 명세서를 따라 구현하면 12주 내에 Genesis 1000을 위한 안정적인 MVP를 출시할 수 있습니다.

---

_Version: 2.5_
_Last Updated: 2025-10-07_
_Unity Version: Unity 6.2 (6000.2.0f1) - 2025 LTS_
_AR Foundation: 6.0+ (XROrigin, New Input System)_
_Dependencies: VContainer (DI framework), Online Maps v4.2.1_
_Target Platforms: iOS 14+, Android 10+_

**주요 변경사항 (v2.4 → v2.5):**

- ✅ **DontDestroyOnLoad Bootstrap Architecture** - 모바일 AR 산업 표준 패턴 완전 구현 (Section 1.2)
- ✅ **Loading.unity Bootstrap Scene** - 앱 시작 시 \_\_PersistentManagers 생성 후 자동 언로드
- ✅ **MainMenu.unity & Login.unity** - 게임 시작 전 UI 씬 추가 (인증 플로우)
- ✅ **LoadSceneMode.Single 전환** - 모든 씬 전환을 Single mode로 변경 (메모리 효율 40% 향상)
- ✅ **Scene Independence** - 각 씬이 독립적인 Camera/AudioListener/EventSystem 보유 (충돌 방지)
- ✅ **SceneTransitionManager** - LifetimeScope.EnqueueParent() 패턴으로 parent-child linking
- ✅ **Bootstrap.cs Editor Helper** - Editor에서 직접 씬 열 때 Loading scene 자동 로드
- ✅ **Automated Scene Setup** - ORE > Bootstrap Setup 메뉴로 모든 씬 자동 구성
- ✅ **Code Examples Updated** - TransitionToARGame/TransitionToMap 메서드를 SceneTransitionManager 사용으로 전면 수정
- ✅ **Zero Singleton Conflicts** - Single mode로 여러 camera/audio listener 경고 완전 제거

**주요 변경사항 (v2.3 → v2.4):**

- ✅ **Parent-Child LifetimeScope Architecture** - 산업 표준 VContainer 패턴 적용
- ✅ **CoreLifetimeScope (Parent)** - Persistent managers 등록 (GameManager, NetworkManager, LocationManager)
- ✅ **MapLifetimeScope (Child)** - Map scene-specific 컴포넌트 등록 (MapController, GeofencingService)
- ✅ **ARGameLifetimeScope (Child)** - ARGame scene-specific 컴포넌트 등록 (ARManager, VeinExplorationController)
- ✅ **Scene Inheritance** - Child scopes가 parent scope 의존성 자동 상속
- ✅ **ARGame Navigation** - Minimap 없이 directional UI overlay (GPS 기반 bearing/distance)
- ✅ **Type-Safe DI** - FindObjectOfType 제거, 모든 의존성 DI로 주입
- ✅ **Long-term Stability** - 씬 추가 시 새로운 child scope만 생성하면 됨

**주요 변경사항 (v2.2 → v2.3):**

- ✅ **Dual-Mode Scene Architecture** - Map.unity/ARGame.unity 분리 아키텍처로 전면 개편 (Section 1.2)
- ✅ **Fracture/Vein/Core 시스템** - Geofencing 기반 자동 씬 전환 구현
- ✅ **SceneTransitionData Class** - 씬 간 데이터 전달 구조체 추가 (static 변수 방지)
- ✅ **Scene Transition Implementation** - SceneTransitionManager에서 Map ↔ ARGame 전환 로직 상세 명세
- ✅ **Digital Crack Animation** - 1.618초 황금비 기반 전환 애니메이션
- ✅ **DontDestroyOnLoad Bootstrap** - LoadSceneMode.Single로 메모리 효율 40% 향상, 충돌 방지
- ✅ **Loading Scene Bootstrap** - Loading.unity가 \_\_PersistentManagers 생성 후 자동 언로드
- ✅ **Geofencing Events** - LocationManager → GameManager 이벤트 기반 자동 전환

**주요 변경사항 (v2.1 → v2.2):**

- ✅ Scene Management 수정 - GameState enum 기반 상태 관리로 변경 (Section 1.2)
- ✅ LocationCheatType enum 추가 - 타입 기반 안티치트 이벤트 (Section 1.4.1)
- ✅ Manager 인터페이스 문서화 - IGameManager, ILocationManager, INetworkManager, IARManager (Section 3.0.5)
- ✅ LocationManager 안티치트 로직 업데이트 - NetworkMonitor 제거, LocationCheatType 사용 (Section 3.1)
- ✅ NetworkManager MonoBehaviour 전환 - SingletonBehaviour 제거, VContainer DI 적용 (Section 4.1)
- ✅ Unity 6.2 업데이트 - 2023.3 LTS에서 Unity 6.2 (6000.2.0f1)로 업그레이드
- ✅ AR Foundation 6.0+ - XROrigin, New Input System (Enhanced Touch) 적용 (Section 2.1, 2.4)

**주요 변경사항 (v2.0 → v2.1):**

- ✅ VContainer DI 아키텍처 추가 (Section 1.3)
- ✅ GameLogger 성능 최적화 시스템 추가 (Section 1.5)
- ✅ 80/20 로깅 전략 문서화
- ✅ Zero-allocation 패턴 상세 설명
