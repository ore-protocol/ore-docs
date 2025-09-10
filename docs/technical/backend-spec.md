# Project ORE - Backend System Specification v5.0

_AI-Native 개발로 구현하는 확장 가능한 AR P2E 플랫폼 백엔드_

## Executive Summary

### 프로젝트 컨텍스트

Project ORE는 포켓몬GO의 위치 기반 게임플레이와 블록체인 투명 광고를 결합한 P2E 플랫폼입니다. **AI-Native 개발 방식**으로 1인 CTO가 Claude Code를 활용하여 20인 팀 수준의 생산성을 달성합니다.

### 핵심 설계 철학

```yaml
"처음부터 제대로, 리팩토링 최소화"

원칙:
  1. Proper Architecture: 도메인별 명확한 분리
  2. Right Technology: 각 문제에 최적 기술
  3. No Shortcuts: 나중에 바꿀 것은 처음부터 제대로
  4. AI Leverage: 복잡도는 AI로 극복
```

### 기술 전략

```yaml
개발 방식:
  - 1인 CTO + Claude Code (10-20x 생산성)
  - 12주 개발 (2025년 9-11월)
  - Genesis 1000 베타 (2025년 12월)

아키텍처:
  - 8개 마이크로서비스 (명확한 도메인 경계)
  - Event-Driven (Kafka 처음부터)
  - CQRS 패턴 (읽기/쓰기 분리)

기술 스택:
  - Core Services: Rust (4개)
  - Business Services: Go (4개)
  - Event Bus: Apache Kafka
  - Analytics: TimescaleDB
  - Cache: Redis Cluster
  - Container: ECS Fargate → EKS Ready

확장 목표:
  - MVP: 1,000 동시접속
  - 6개월: 10,000 동시접속
  - 1년: 100,000 동시접속
  - 재작성 없이 1,000,000까지
```

## 1. System Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│                     Clients                         │
│   Unity Mobile    Web Dashboard    Admin Portal     │
└─────────────────────────────────────────────────────┘
                           │
                    ┌──────▼──────┐
                    │ CloudFront  │
                    │    CDN      │
                    └──────┬──────┘
                           │
                ┌──────────▼──────────┐
                │    API Gateway      │
                │  (Kong or AWS APIG) │
                └──────────┬──────────┘
                           │
    ┌──────────────────────┼──────────────────────┐
    │               Service Mesh                  │
    │              (AWS App Mesh)                 │
    └──────────────────────┬──────────────────────┘
                           │
    ┌──────────┬───────────┼───────────┬──────────────┐
    ▼          ▼           ▼           ▼              ▼
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌───────────┐ ┌─────────┐
│Location │ │  Game   │ │Realtime │ │Blockchain │ │   User  │
│Service  │ │Service  │ │ Engine  │ │ Service   │ │ Service │
│ (Rust)  │ │ (Rust)  │ │ (Rust)  │ │  (Rust)   │ │  (Go)   │
└─────────┘ └─────────┘ └─────────┘ └───────────┘ └─────────┘
    │          │           │           │               │
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌───────────┐ ┌─────────┐
│   Ad    │ │Analytics│ │  Auth   │ │   Event   │ │  Social │
│Service  │ │Service  │ │Service  │ │ Processor │ │ Service │
│  (Go)   │ │  (Go)   │ │  (Go)   │ │   (Go)    │ │  (Go)   │
└─────────┘ └─────────┘ └─────────┘ └───────────┘ └─────────┘
                           │
    ┌──────────────────────┼──────────────────────┐
    │                 Kafka Event Bus             │
    │         (Topics: events, commands, logs)    │
    └────────────────────────────┬────────────────┘
                                 │
    ┌──────────────┬─────────────┼───────────┬─────────────┐
    ▼              ▼             ▼           ▼             ▼
┌───────────┐ ┌───────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────┐
│PostgreSQL │ │TimescaleDB│ │  Redis  │ │   S3    │ │ Polygon     │
│(Primary)  │ │(Analytics)│ │ (Cache) │ │(Storage)│ │(Blockchain) │
└───────────┘ └───────────┘ └─────────┘ └─────────┘ └─────────────┘
```

### 1.2 Service Domain Boundaries

```yaml
Location Domain (Rust):
  책임: 모든 위치 관련 처리
  - GPS 데이터 수집/검증
  - S2 Geometry 공간 인덱싱
  - 지오펜싱 및 영역 관리
  - 위치 기반 이벤트 트리거
  독립성: 다른 서비스 의존 없음

Game Domain (Rust):
  책임: 핵심 게임 로직
  - 코인 수집 트랜잭션
  - 곡괭이 시스템
  - 인벤토리 관리
  - 퀘스트 시스템
  독립성: 이벤트 기반 통신만

Realtime Domain (Rust):
  책임: 실시간 통신
  - WebSocket 연결 관리
  - 실시간 위치 브로드캐스팅
  - 라이브 이벤트 전파
  - P2P 메시징
  독립성: 완전 독립 실행

Blockchain Domain (Rust):
  책임: Web3 상호작용
  - 스마트 컨트랙트 호출
  - 토큰 트랜잭션
  - 이벤트 모니터링
  - 가스 최적화
  독립성: 외부 체인 의존

User Domain (Go):
  책임: 사용자 관리
  - 프로필 관리
  - Genesis 1000 관리
  - 계정 설정
  - 프리미엄 기능
  독립성: Auth와 협력

Auth Domain (Go):
  책임: 인증/인가
  - JWT 토큰 관리
  - OAuth2 소셜 로그인
  - 권한 관리
  - 세션 관리
  독립성: 게이트웨이 역할

Ad Domain (Go):
  책임: 광고 시스템
  - 캠페인 관리
  - 타겟팅 로직
  - 예산 관리
  - 성과 추적
  독립성: 이벤트 구독

Analytics Domain (Go):
  책임: 데이터 분석
  - 메트릭 수집
  - 리포트 생성
  - A/B 테스트
  - 비즈니스 인텔리전스
  독립성: 이벤트 소비만
```

### 1.3 Event-Driven Architecture

```yaml
Kafka Topics:
  # Commands (요청)
  commands.location.update
  commands.coin.collect
  commands.quest.complete

  # Events (발생한 일)
  events.location.updated
  events.coin.collected
  events.quest.completed
  events.user.registered
  events.ad.viewed

  # Analytics (분석용)
  analytics.raw.events
  analytics.processed.metrics

  # Dead Letter Queue
  dlq.failed.events

Event Flow Example:
  1. User collects coin →
  2. Game Service validates →
  3. Publishes "coin.collected" event →
  4. Consumed by:
     - Analytics (통계 업데이트)
     - Ad Service (광고 코인 체크)
     - Blockchain (토큰 보상)
     - Realtime (다른 유저 알림)
```

## 2. 주요 기능별 End-to-End 데이터 흐름

### 2.1 코인 수집 플로우

```yaml
사용자 코인 수집 요청 처리:
  1. Unity Client: → 코인 감지 및 수집 애니메이션
    → POST /game/coins/{id}/collect API 호출

  2. API Gateway: → JWT 토큰 검증
    → Rate limiting 체크
    → Game Service로 라우팅

  3. Game Service (Rust): → 요청 멱등성 체크 (중복 방지)
    → 위치 유효성 검증 (Location Service 호출)
    → 코인 소유권 확인
    → DB 트랜잭션 시작
    → 플레이어 상태 업데이트
    → 코인 수집 기록
    → 트랜잭션 커밋
    → Kafka에 이벤트 발행

  4. Event Bus (Kafka): → events.coin.collected 토픽에 발행
    → 여러 서비스가 동시 구독

  5. Event Consumers:
    a) Analytics Service: → 통계 데이터 업데이트
      → TimescaleDB에 저장
      → 리더보드 갱신

    b) Ad Service: → 광고 코인인지 확인
      → 광고 성과 추적
      → 광고주 대시보드 업데이트

    c) Realtime Engine: → 주변 플레이어에게 알림
      → 코인 사라짐 브로드캐스트
      → 미니맵 업데이트

    d) Blockchain Service: → 보상 계산
      → 토큰 지급 큐에 추가
      → 일괄 처리 대기

  6. Response to Client: → 성공 응답 + 업데이트된 잔액
    → WebSocket으로 실시간 업데이트
```

### 2.2 위치 업데이트 플로우

```yaml
사용자 위치 업데이트 처리:
  1. Unity Client: → GPS 데이터 수집 (30초마다)
    → 정확도 필터링
    → POST /location/update

  2. Location Service (Rust): → Anti-cheat 검증
    - 속도 체크 (150km/h 이하)
    - 가속도 체크
    - 패턴 분석
    → S2 Cell 계산
    → 공간 인덱스 업데이트
    → 지오펜스 트리거 체크

  3. Event Publishing: → events.location.updated 발행

  4. Triggered Actions:
    a) Game Service: → 근처 코인 스폰
      → 퀘스트 진행 체크

    b) Ad Service: → 광고 존 진입 체크
      → 광고 코인 생성

    c) Realtime Engine: → 플레이어 위치 브로드캐스트
      → 근접 플레이어 알림

    d) Analytics: → 히트맵 업데이트
      → 활동 패턴 분석
```

### 2.3 광고 시청 및 보상 플로우

```yaml
광고 시청 완료 처리:
  1. Unity Client: → 광고 SDK 호출
    → 시청 완료 콜백
    → POST /ads/{id}/complete

  2. Ad Service (Go): → 시청 시간 검증
    → 광고주 예산 차감
    → 완료 이벤트 발행

  3. Event Processing: → events.ad.viewed 발행

  4. Reward Distribution:
    a) Game Service: → 보너스 코인 지급
      → 곡괭이 효율 부스트

    b) Blockchain Service: → ORE 토큰 계산
      → 스마트 컨트랙트 호출
      → 온체인 기록

    c) Analytics: → 광고 성과 추적
      → CTR/CVR 계산
      → ROI 리포트
```

### 2.4 Genesis 1000 특별 플로우

```yaml
Genesis 멤버 특별 처리:
  1. 가입 시: → Discord/Twitter 검증
    → 고유 번호 할당 (1-1000)
    → NFT 뱃지 민팅 준비

  2. 게임 플레이: → 2x 보상 배수
    → 특별 코인 스폰율
    → 프리미엄 퀘스트

  3. 토큰 에어드롭: → 총 공급량 3% 예약
    → 베스팅 스케줄 적용
    → 거버넌스 권한 부여
```

## 3. 프론트엔드-백엔드 컴포넌트 매핑

### 3.1 Unity 컴포넌트 ↔ 백엔드 서비스 매핑

| Unity 컴포넌트           | 주요 백엔드 서비스              | API 엔드포인트                                      | 실시간 채널                              |
| ------------------------ | ------------------------------- | --------------------------------------------------- | ---------------------------------------- |
| **ARInteractionManager** | Game Service, Location Service  | `/game/coins/nearby`<br>`/game/coins/{id}/collect`  | `coins.spawn`<br>`coins.collected`       |
| **LocationManager**      | Location Service                | `/location/update`<br>`/location/nearby`            | `location.updates`                       |
| **CoinCollectionSystem** | Game Service, Realtime Engine   | `/game/coins/{id}/collect`<br>`/game/inventory`     | `coins.collected`<br>`inventory.updated` |
| **InventorySystem**      | Game Service                    | `/game/inventory`<br>`/game/items/{id}/use`         | `inventory.changed`                      |
| **QuestSystem**          | Game Service, Analytics Service | `/game/quests`<br>`/game/quests/{id}/complete`      | `quest.progress`<br>`quest.completed`    |
| **AdManager**            | Ad Service                      | `/ads/available`<br>`/ads/{id}/view`                | `ad.triggered`                           |
| **SocialManager**        | User Service, Realtime Engine   | `/social/friends`<br>`/social/chat/{channel}`       | `chat.messages`<br>`friends.online`      |
| **ProfileManager**       | User Service, Auth Service      | `/users/me`<br>`/users/avatar`                      | `profile.updated`                        |
| **LeaderboardUI**        | Analytics Service               | `/analytics/leaderboard`<br>`/analytics/rankings`   | `leaderboard.updated`                    |
| **AuthenticationFlow**   | Auth Service                    | `/auth/login`<br>`/auth/refresh`                    | -                                        |
| **BlockchainBridge**     | Blockchain Service              | `/blockchain/balance`<br>`/blockchain/transactions` | `transaction.confirmed`                  |
| **OfflineManager**       | - (로컬 처리)                   | -                                                   | -                                        |

### 3.2 데이터 동기화 매트릭스

```yaml
실시간 동기화 (WebSocket):
  위치 데이터:
    Unity → Location Service: 30초마다
    Location → Realtime Engine: 즉시
    Realtime → 주변 플레이어: < 100ms

  게임 상태:
    코인 수집: 즉시 동기화
    인벤토리: 변경 시 즉시
    퀘스트: 진행률 5분마다

  소셜 기능:
    채팅: 실시간
    친구 상태: 1분마다
    리더보드: 5분마다

배치 동기화:
  블록체인:
    토큰 전송: 10분 배치
    NFT 민팅: 1시간 배치

  분석 데이터:
    이벤트 로그: 1분 배치
    통계 집계: 5분 배치
```

### 3.3 오프라인 모드 처리

```yaml
Unity 로컬 캐싱:
  캐시 데이터:
    - 플레이어 프로필
    - 인벤토리 상태
    - 수집한 코인 (임시)
    - 퀘스트 진행률

  재연결 시 동기화: 1. 오프라인 액션 큐 전송
    2. 서버 검증 및 조정
    3. 최종 상태 동기화
    4. 보상 지급

백엔드 충돌 해결:
  우선순위: 1. 블록체인 기록 (최우선)
    2. 서버 타임스탬프
    3. 클라이언트 데이터

  검증 규칙:
    - 코인 중복 수집 방지
    - 위치 이동 합리성 체크
    - 시간 조작 탐지
```

## 4. Core Services (Rust) - 성능 크리티컬

### 4.0 Rust Coding Standards

**Module Organization:**

```rust
// ✅ Explicit module declarations only
pub mod game;
pub mod rules;

// ❌ No re-exports in mod.rs files
// pub use game::GameService;  // DON'T DO THIS

// ✅ Use explicit imports in consuming code instead
use crate::services::game::GameService;
use crate::models::player::PlayerState;
```

**Import Conventions:**

- Always use explicit imports: `use crate::services::game::GameService`
- No re-exports in `mod.rs` files to maintain clear dependency tracking
- Prefer explicit over implicit to improve AI code analysis and maintainability

**Error Handling:**

```rust
// ✅ Use thiserror for domain-specific errors
#[derive(Debug, thiserror::Error)]
pub enum GameError {
    #[error("Database operation failed: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Invalid coin collection: {reason}")]
    InvalidCollection { reason: String },
}
```

**Async Patterns:**

```rust
// ✅ Use Result<T> for all fallible operations
pub async fn collect_coin(&self, cmd: CollectCoinCommand) -> Result<CollectResult, GameError> {
    // Implementation
}

// ✅ Use Arc<T> for shared state across async tasks
player_states: Arc<DashMap<PlayerId, PlayerState>>,
```

### 4.1 Location Service

```rust
// Service Architecture
pub struct LocationService {
    // Spatial indexing
    spatial_index: Arc<RwLock<S2CellIndex>>,
    rtree: Arc<RwLock<RTree<CoinLocation>>>,

    // Caching
    cache: Arc<RedisCluster>,

    // Event publishing
    kafka_producer: Arc<KafkaProducer>,

    // Anti-cheat
    anti_cheat: Arc<AntiCheatEngine>,

    // Metrics
    metrics: Arc<PrometheusMetrics>,
}

// Core APIs
impl LocationService {
    // 위치 업데이트 (Command)
    pub async fn update_location(&self, cmd: UpdateLocationCommand) -> Result<()> {
        // 1. Validate movement
        if !self.anti_cheat.validate_movement(&cmd).await? {
            return Err(LocationError::InvalidMovement);
        }

        // 2. Update spatial index
        self.spatial_index.update(cmd.user_id, cmd.location).await?;

        // 3. Check triggers (지오펜싱, 이벤트 영역)
        let triggers = self.check_location_triggers(&cmd.location).await?;

        // 4. Publish event
        self.kafka_producer.send(&Event::LocationUpdated {
            user_id: cmd.user_id,
            location: cmd.location,
            triggers,
            timestamp: Utc::now(),
        }).await?;

        // 5. Update cache
        self.cache.set_location(cmd.user_id, cmd.location).await?;

        Ok(())
    }

    // 근접 검색 (Query)
    pub async fn find_nearby(&self, query: NearbyQuery) -> Result<NearbyResponse> {
        // S2 Cell 기반 효율적 검색
        let cell = S2CellId::from_point(&query.location);
        let neighbors = cell.get_neighbors(query.radius);

        // R-tree로 정확한 거리 필터링
        let items = self.rtree.find_within_radius(
            query.location,
            query.radius,
        ).await?;

        Ok(NearbyResponse {
            coins: items.coins,
            players: items.players,
            ads: items.ads,
        })
    }
}

// Performance Characteristics
// - Throughput: 100,000 updates/sec
// - Query latency: P95 < 5ms
// - Memory: O(n) where n = active users
// - CPU: Lock-free operations
```

### 4.2 Game Service

```rust
// Domain Model
pub struct GameService {
    // State management
    player_states: Arc<DashMap<PlayerId, PlayerState>>,

    // Game rules
    rules_engine: Arc<RulesEngine>,
    drop_tables: Arc<DropTables>,

    // Persistence
    db_pool: Arc<PgPool>,

    // Events
    kafka_producer: Arc<KafkaProducer>,
}

// Transaction Processing
impl GameService {
    // 코인 수집 (멱등성 보장)
    pub async fn collect_coin(&self, cmd: CollectCoinCommand) -> Result<CollectResult> {
        // 1. Idempotency check
        if self.is_duplicate_request(&cmd.request_id).await? {
            return self.get_cached_result(&cmd.request_id).await;
        }

        // 2. Start transaction
        let mut tx = self.db_pool.begin().await?;

        // 3. Validate collection
        let validation = self.validate_collection(&cmd, &mut tx).await?;
        if !validation.is_valid {
            tx.rollback().await?;
            return Err(GameError::InvalidCollection(validation.reason));
        }

        // 4. Update player state
        let player = self.player_states.get_mut(&cmd.player_id).unwrap();
        player.add_coin(cmd.coin_id, cmd.coin_value);

        // 5. Save to database
        sqlx::query!(
            "INSERT INTO coin_collections (player_id, coin_id, collected_at)
             VALUES ($1, $2, $3)",
            cmd.player_id, cmd.coin_id, Utc::now()
        ).execute(&mut tx).await?;

        // 6. Commit transaction
        tx.commit().await?;

        // 7. Publish event
        self.kafka_producer.send(&Event::CoinCollected {
            player_id: cmd.player_id,
            coin_id: cmd.coin_id,
            value: cmd.coin_value,
            location: cmd.location,
            timestamp: Utc::now(),
        }).await?;

        // 8. Cache result (for idempotency)
        self.cache_result(&cmd.request_id, &result).await?;

        Ok(result)
    }

    // 곡괭이 강화 시스템
    pub async fn upgrade_pickaxe(&self, cmd: UpgradePickaxeCommand) -> Result<UpgradeResult> {
        // Similar transaction pattern
        // Ensures no item duplication or currency exploits
    }
}

// Safety Guarantees
// - No data races (Rust compiler enforced)
// - No memory leaks (RAII)
// - No null pointer exceptions
// - Transaction safety (ACID)
```

### 4.3 Realtime Engine

```rust
pub struct RealtimeEngine {
    // Connection management
    connections: Arc<DashMap<ConnectionId, WebSocketConnection>>,

    // Room management (S2 Cell based)
    rooms: Arc<DashMap<S2CellId, HashSet<ConnectionId>>>,

    // Message routing
    router: Arc<MessageRouter>,

    // Kafka integration
    kafka_consumer: Arc<KafkaConsumer>,
    kafka_producer: Arc<KafkaProducer>,
}

impl RealtimeEngine {
    // WebSocket handler
    pub async fn handle_connection(&self, ws: WebSocket) -> Result<()> {
        let conn_id = Uuid::new_v4();
        let (tx, rx) = ws.split();

        // Register connection
        self.connections.insert(conn_id, Connection { tx, metadata });

        // Subscribe to relevant Kafka topics
        self.subscribe_to_events(conn_id).await?;

        // Message loop
        while let Some(msg) = rx.next().await {
            match msg {
                Message::Binary(data) => {
                    // Use MessagePack for efficiency
                    let decoded: ClientMessage = rmp_serde::decode(&data)?;
                    self.handle_client_message(conn_id, decoded).await?;
                }
                Message::Close(_) => break,
                _ => {}
            }
        }

        // Cleanup
        self.connections.remove(&conn_id);
        Ok(())
    }

    // Broadcast to S2 Cell (location-based room)
    pub async fn broadcast_to_cell(&self, cell_id: S2CellId, message: &[u8]) -> Result<()> {
        if let Some(connections) = self.rooms.get(&cell_id) {
            // Parallel broadcast with zero-copy
            let futures: Vec<_> = connections.iter()
                .filter_map(|conn_id| {
                    self.connections.get(conn_id).map(|conn| {
                        conn.tx.send(Message::Binary(message.into()))
                    })
                })
                .collect();

            futures::future::join_all(futures).await;
        }
        Ok(())
    }
}

// Scalability
// - 10,000+ concurrent connections
// - < 50ms P99 latency
// - 100KB memory per connection
// - Horizontal scaling ready
```

### 4.4 Blockchain Service

```rust
use ethers::prelude::*;

pub struct BlockchainService {
    // Web3 providers (multiple for redundancy)
    providers: Vec<Arc<Provider<Http>>>,

    // Wallet management
    wallet: LocalWallet,

    // Smart contracts
    ore_token: Arc<OreTokenContract>,
    ad_campaign: Arc<AdCampaignContract>,

    // Transaction queue
    tx_queue: Arc<TransactionQueue>,

    // Gas optimization
    gas_optimizer: Arc<GasOptimizer>,
}

impl BlockchainService {
    // Send rewards with retry logic
    pub async fn send_rewards(&self, cmd: SendRewardsCommand) -> Result<TxHash> {
        // 1. Optimize gas
        let gas_price = self.gas_optimizer.get_optimal_gas_price().await?;

        // 2. Build transaction
        let tx = self.ore_token
            .transfer(cmd.recipient, cmd.amount)
            .gas_price(gas_price)
            .gas(150_000);

        // 3. Sign and send with retry
        let pending = tx.send().await?;

        // 4. Wait for confirmation
        let receipt = pending.confirmations(3).await?;

        // 5. Publish event
        self.kafka_producer.send(&Event::RewardsSent {
            recipient: cmd.recipient,
            amount: cmd.amount,
            tx_hash: receipt.transaction_hash,
            block: receipt.block_number,
        }).await?;

        Ok(receipt.transaction_hash)
    }

    // Monitor blockchain events
    pub async fn monitor_events(&self) -> Result<()> {
        let events = self.ore_token.events();
        let mut stream = events.stream().await?;

        while let Some(event) = stream.next().await {
            match event {
                OreTokenEvent::Transfer(transfer) => {
                    self.handle_transfer_event(transfer).await?;
                }
                OreTokenEvent::Approval(approval) => {
                    self.handle_approval_event(approval).await?;
                }
            }
        }
        Ok(())
    }
}
```

## 5. Business Services (Go) - 비즈니스 로직

### 5.1 User Service

```go
type UserService struct {
    db          *sqlx.DB
    cache       *redis.ClusterClient
    kafka       *kafka.Producer
    authClient  AuthServiceClient
}

// Genesis 1000 관리
type GenesisManager struct {
    service *UserService
}

func (gm *GenesisManager) RegisterGenesisMember(req RegisterGenesisRequest) (*GenesisMember, error) {
    // 1. Verify Discord/Twitter
    if !gm.verifyDiscord(req.DiscordID) {
        return nil, ErrInvalidDiscord
    }

    // 2. Check availability (1-1000)
    number := gm.getNextAvailableNumber()
    if number > 1000 {
        return nil, ErrGenesisFulll
    }

    // 3. Create Genesis member
    member := &GenesisMember{
        UserID:         req.UserID,
        GenesisNumber:  number,
        DiscordID:      req.DiscordID,
        TwitterHandle:  req.TwitterHandle,
        BonusMultiplier: 2.0,
        AirdropEligible: true,
    }

    // 4. Save to database
    _, err := gm.service.db.Exec(`
        INSERT INTO genesis_members
        (user_id, genesis_number, discord_id, twitter_handle, bonus_multiplier)
        VALUES ($1, $2, $3, $4, $5)`,
        member.UserID, member.GenesisNumber, member.DiscordID,
        member.TwitterHandle, member.BonusMultiplier,
    )

    // 5. Publish event
    gm.service.kafka.Produce("events.genesis.registered", member)

    // 6. Cache Genesis status
    gm.service.cache.Set(fmt.Sprintf("genesis:%s", req.UserID), "true", 0)

    return member, nil
}

// Profile management with Genesis benefits
func (s *UserService) GetProfile(userID string) (*UserProfile, error) {
    // Check cache first
    cached, _ := s.cache.Get(fmt.Sprintf("profile:%s", userID)).Result()
    if cached != "" {
        return unmarshalProfile(cached), nil
    }

    // Query with Genesis join
    var profile UserProfile
    err := s.db.Get(&profile, `
        SELECT u.*,
               gm.genesis_number,
               gm.bonus_multiplier,
               gm.airdrop_eligible
        FROM users u
        LEFT JOIN genesis_members gm ON u.id = gm.user_id
        WHERE u.id = $1`, userID)

    // Apply Genesis benefits
    if profile.GenesisNumber > 0 {
        profile.IsGenesis = true
        profile.Benefits = getGenesisBenefits()
    }

    // Cache and return
    s.cache.Set(fmt.Sprintf("profile:%s", userID), marshalProfile(profile), 5*time.Minute)

    return &profile, nil
}
```

### 5.2 Auth Service

```go
type AuthService struct {
    db           *sqlx.DB
    redis        *redis.ClusterClient
    jwtSecret    []byte
    kafka        *kafka.Producer
}

// Enhanced JWT with Genesis status
func (s *AuthService) IssueToken(userID string) (*TokenPair, error) {
    // Get user with Genesis status
    var user struct {
        ID        string
        IsGenesis bool
        Role      string
    }

    err := s.db.Get(&user, `
        SELECT u.id,
               CASE WHEN gm.user_id IS NOT NULL THEN true ELSE false END as is_genesis,
               u.role
        FROM users u
        LEFT JOIN genesis_members gm ON u.id = gm.user_id
        WHERE u.id = $1`, userID)

    // Create claims with Genesis benefits
    claims := jwt.MapClaims{
        "sub": userID,
        "role": user.Role,
        "is_genesis": user.IsGenesis,
        "iat": time.Now().Unix(),
        "exp": time.Now().Add(s.getTokenDuration(user.IsGenesis)).Unix(),
    }

    // Genesis members get longer token life
    accessToken := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    accessString, _ := accessToken.SignedString(s.jwtSecret)

    // Refresh token
    refreshToken := generateRefreshToken()
    s.redis.Set(fmt.Sprintf("refresh:%s", refreshToken), userID, 7*24*time.Hour)

    // Publish login event
    s.kafka.Produce("events.auth.login", map[string]interface{}{
        "user_id": userID,
        "is_genesis": user.IsGenesis,
        "timestamp": time.Now(),
    })

    return &TokenPair{
        AccessToken:  accessString,
        RefreshToken: refreshToken,
    }, nil
}

func (s *AuthService) getTokenDuration(isGenesis bool) time.Duration {
    if isGenesis {
        return 30 * time.Minute // Genesis: 30 min
    }
    return 15 * time.Minute // Regular: 15 min
}
```

### 5.3 Ad Service

```go
type AdService struct {
    db            *sqlx.DB
    cache         *redis.ClusterClient
    kafka         *kafka.Producer
    kafkaConsumer *kafka.Consumer
    locationClient LocationServiceClient
}

// Location-based ad matching
func (s *AdService) GetRelevantAds(location Location) ([]*Ad, error) {
    // 1. Get S2 cell for location
    cellID := s2.CellIDFromLatLng(s2.LatLngFromDegrees(location.Lat, location.Lng))

    // 2. Query ads for this cell and parent cells
    cells := getCellHierarchy(cellID)

    var ads []*Ad
    query := `
        SELECT * FROM ad_campaigns
        WHERE s2_cell_id = ANY($1)
          AND status = 'active'
          AND budget_remaining > 0
          AND NOW() BETWEEN starts_at AND ends_at
        ORDER BY bid_amount DESC, ctr DESC
        LIMIT 10`

    err := s.db.Select(&ads, query, pq.Array(cells))

    // 3. Filter by targeting criteria
    filtered := s.applyTargeting(ads, location)

    // 4. Record impressions
    for _, ad := range filtered {
        s.kafka.Produce("events.ad.impression", map[string]interface{}{
            "ad_id": ad.ID,
            "location": location,
            "timestamp": time.Now(),
        })
    }

    return filtered, nil
}

// Consume location events for ad triggers
func (s *AdService) ConsumeLocationEvents() {
    s.kafkaConsumer.Subscribe("events.location.updated")

    for message := range s.kafkaConsumer.Messages() {
        var event LocationUpdatedEvent
        json.Unmarshal(message.Value, &event)

        // Check if user entered ad zone
        if s.isInAdZone(event.Location) {
            // Trigger ad coin spawn
            s.spawnAdCoin(event.UserID, event.Location)
        }
    }
}
```

### 5.4 Analytics Service

```go
type AnalyticsService struct {
    timescale    *sqlx.DB  // TimescaleDB
    redis        *redis.ClusterClient
    kafka        *kafka.Consumer
    metrics      *prometheus.Registry
}

// Consume all events for analytics
func (s *AnalyticsService) StartEventConsumer() {
    topics := []string{
        "events.location.updated",
        "events.coin.collected",
        "events.quest.completed",
        "events.ad.viewed",
        "events.user.registered",
    }

    s.kafka.Subscribe(topics...)

    for message := range s.kafka.Messages() {
        go s.processEvent(message)
    }
}

// Process and store in TimescaleDB
func (s *AnalyticsService) processEvent(message *kafka.Message) {
    // 1. Parse event
    var event map[string]interface{}
    json.Unmarshal(message.Value, &event)

    // 2. Insert into TimescaleDB
    _, err := s.timescale.Exec(`
        INSERT INTO events (time, topic, user_id, event_data)
        VALUES ($1, $2, $3, $4)`,
        time.Now(), message.Topic, event["user_id"], message.Value,
    )

    // 3. Update real-time metrics
    s.updateMetrics(message.Topic, event)

    // 4. Update aggregations
    s.updateAggregations(message.Topic, event)
}

// Real-time metrics API
func (s *AnalyticsService) GetRealTimeMetrics() (*Metrics, error) {
    metrics := &Metrics{}

    // Get from Redis (updated by event processor)
    metrics.ActiveUsers, _ = s.redis.Get("metrics:active_users").Int()
    metrics.CoinsCollected, _ = s.redis.Get("metrics:coins_collected").Int()
    metrics.QuestsCompleted, _ = s.redis.Get("metrics:quests_completed").Int()

    // Get from TimescaleDB (aggregated)
    s.timescale.Get(metrics, `
        SELECT
            COUNT(DISTINCT user_id) as dau,
            COUNT(*) as total_events,
            AVG(session_length) as avg_session
        FROM events
        WHERE time > NOW() - INTERVAL '24 hours'`)

    return metrics, nil
}
```

## 6. Data Architecture

### 6.1 PostgreSQL (Primary Database)

#### 6.1.1 Database Schema Management Strategy

The ORE platform uses an **ORM-first approach** for database schema management:

```yaml
ORM-First Database Strategy:
  Philosophy: "Let the services own their schemas"

  Implementation:
    - No static table definitions in infrastructure/docker/postgres/init.sql
    - Each service manages its own database schema via ORM auto-migration
    - Database initialization only creates extensions and basic setup

  Service-Specific Schema Management:
    Rust Services (SQLx):
      - Database migrations in service/migrations/ directory
      - Versioned migration files: 001_initial.up.sql, 001_initial.down.sql
      - Applied via SQLx migrate during service startup

    Go Services (GORM):
      - Auto-migration via GORM during service startup
      - Struct-based schema definitions
      - GORM handles CREATE TABLE, ALTER TABLE automatically

  Benefits:
    - No schema conflicts between services
    - Independent service deployments
    - Schema evolution tied to service releases
    - Simplified database setup for development

  Trade-offs (and Modern Solutions):
    ❌ Coordination for shared tables:
      ✅ Solution: Event-driven architecture eliminates most sharing needs

    ❌ Initial setup takes longer:
      ✅ Solution: 30 seconds per service vs hours of schema conflicts

    ❌ Distributed schema documentation:
      ✅ Solution: Each service documents its own schema (better ownership)

  Industry Context:
    - ORM-first is the 2025 standard for microservices (Netflix, Spotify, Uber, Stripe)
    - Eliminates deployment coupling between services
    - Enables true "database per service" microservices principle
    - Static SQL schemas are considered legacy anti-pattern in modern microservices
```

#### 6.1.2 Database Schema Structure

**Note:** The following schema represents the logical structure. Actual tables are created by each service's ORM auto-migration during startup.

```sql
-- Core domain tables (managed by respective services)
-- users table: Created by auth-service via GORM auto-migration
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    wallet_address VARCHAR(42) UNIQUE,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    level INTEGER DEFAULT 1,
    experience BIGINT DEFAULT 0,
    points BIGINT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Location domain: Created by location-service via SQLx migrations
CREATE TABLE locations (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    point GEOGRAPHY(POINT, 4326) NOT NULL,
    s2_cell_id BIGINT NOT NULL,
    accuracy FLOAT,
    recorded_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (recorded_at);

-- Game domain: Created by game-service via SQLx migrations
CREATE TABLE coins (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location GEOGRAPHY(POINT, 4326) NOT NULL,
    s2_cell_id BIGINT NOT NULL,
    type VARCHAR(20) NOT NULL,
    value INTEGER NOT NULL,
    spawned_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ,
    collected_by UUID REFERENCES users(id),
    collected_at TIMESTAMPTZ
);

-- Game domain: Created by game-service via SQLx migrations
CREATE TABLE pickaxes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    type VARCHAR(20) NOT NULL,
    level INTEGER DEFAULT 1,
    efficiency DECIMAL(3,2) DEFAULT 1.0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- User domain: Created by user-service via GORM auto-migration
CREATE TABLE genesis_members (
    user_id UUID PRIMARY KEY REFERENCES users(id),
    genesis_number INTEGER UNIQUE CHECK (genesis_number BETWEEN 1 AND 1000),
    discord_id VARCHAR(100) UNIQUE NOT NULL,
    twitter_handle VARCHAR(100),
    bonus_multiplier DECIMAL(2,1) DEFAULT 2.0,
    airdrop_eligible BOOLEAN DEFAULT TRUE,
    joined_at TIMESTAMPTZ DEFAULT NOW()
);

-- Ad domain: Created by ad-service via GORM auto-migration
CREATE TABLE ad_campaigns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    advertiser_id UUID REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    budget_remaining DECIMAL(10,2),
    bid_amount DECIMAL(10,4),
    s2_cell_id BIGINT,
    targeting_criteria JSONB,
    status VARCHAR(20) DEFAULT 'draft',
    starts_at TIMESTAMPTZ,
    ends_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Optimized indexes
CREATE INDEX idx_locations_s2cell ON locations(s2_cell_id);
CREATE INDEX idx_locations_user_time ON locations(user_id, recorded_at DESC);
CREATE INDEX idx_coins_s2cell_available ON coins(s2_cell_id) WHERE collected_by IS NULL;
CREATE INDEX idx_coins_location ON coins USING GIST(location);
CREATE INDEX idx_ad_campaigns_active ON ad_campaigns(s2_cell_id, status) WHERE status = 'active';
CREATE INDEX idx_genesis_number ON genesis_members(genesis_number);
```

### 6.2 TimescaleDB (Analytics)

```sql
-- Create hypertable for time-series data
CREATE TABLE events (
    time TIMESTAMPTZ NOT NULL,
    topic VARCHAR(100) NOT NULL,
    user_id UUID,
    event_data JSONB,
    location GEOGRAPHY(POINT, 4326),
    session_id UUID
);

SELECT create_hypertable('events', 'time');

-- Continuous aggregates for real-time analytics
CREATE MATERIALIZED VIEW hourly_stats
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour,
    topic,
    COUNT(*) as event_count,
    COUNT(DISTINCT user_id) as unique_users,
    AVG((event_data->>'value')::numeric) as avg_value
FROM events
GROUP BY hour, topic;

-- Retention policy (keep raw data for 30 days)
SELECT add_retention_policy('events', INTERVAL '30 days');

-- Compression policy (compress after 7 days)
ALTER TABLE events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'topic'
);

SELECT add_compression_policy('events', INTERVAL '7 days');
```

### 6.3 Redis Cluster (Cache & Real-time)

```yaml
Cluster Configuration:
  - 3 master nodes
  - 3 replica nodes
  - Automatic failover
  - Consistent hashing

Data Structures:
  # User sessions
  STRING session:{token} → user_data
  TTL: 24 hours

  # Location tracking
  GEO locations:active → user_positions
  ZSET locations:recent:{cell_id} → users by time

  # Game state
  HASH user:state:{user_id} → game_state
  SET coins:cell:{cell_id} → available_coins

  # Leaderboards
  ZSET leaderboard:global → scores
  ZSET leaderboard:daily → today_scores
  ZSET leaderboard:genesis → genesis_only

  # Rate limiting
  STRING rate:{user_id}:{minute} → count
  TTL: 60 seconds

  # Real-time metrics
  HASH metrics:realtime → {
    active_users: count,
    coins_collected: count,
    quests_completed: count
  }

  # Pub/Sub for real-time updates
  PUBSUB channels:
    - location:updates
    - game:events
    - chat:messages
```

### 6.4 Kafka Topics & Schema

```yaml
Topic Configuration:
  # Commands (requests)
  commands.location.update:
    partitions: 10
    replication: 3
    retention: 24 hours

  commands.coin.collect:
    partitions: 10
    replication: 3
    retention: 24 hours

  # Events (what happened)
  events.location.updated:
    partitions: 20
    replication: 3
    retention: 7 days

  events.coin.collected:
    partitions: 20
    replication: 3
    retention: 7 days

  # Analytics
  analytics.raw:
    partitions: 50
    replication: 2
    retention: 30 days
    compression: snappy

  # Dead letter queue
  dlq.failed:
    partitions: 5
    replication: 3
    retention: 30 days

Schema Registry:
  Format: Avro
  Compatibility: BACKWARD

  Schemas:
    - LocationUpdatedEvent
    - CoinCollectedEvent
    - QuestCompletedEvent
    - UserRegisteredEvent
    - AdViewedEvent
```

## 7. API Design

### 7.1 REST API Structure

```yaml
Base URL: https://api.ore.game/v1

Authentication: POST   /auth/register
  POST   /auth/login
  POST   /auth/refresh
  POST   /auth/logout
  POST   /auth/social/{provider} # Google, Discord, Twitter

User: GET    /users/me
  PUT    /users/me
  GET    /users/{id}
  GET    /users/search?q={query}
  POST   /users/avatar

Genesis: POST   /genesis/apply
  GET    /genesis/verify
  GET    /genesis/members
  GET    /genesis/benefits
  GET    /genesis/leaderboard

Location: POST   /location/update
  GET    /location/nearby?radius={m}
  GET    /location/history
  POST   /location/share

Game: GET    /game/coins/nearby
  POST   /game/coins/{id}/collect
  GET    /game/inventory
  POST   /game/pickaxe/{id}/equip
  POST   /game/pickaxe/{id}/upgrade
  GET    /game/quests
  POST   /game/quests/{id}/complete
  GET    /game/leaderboard

Ads: GET    /ads/campaigns
  POST   /ads/campaigns
  PUT    /ads/campaigns/{id}
  DELETE /ads/campaigns/{id}
  GET    /ads/campaigns/{id}/analytics
  POST   /ads/campaigns/{id}/fund
  GET    /ads/available

Social: GET    /social/friends
  POST   /social/friends/add
  DELETE /social/friends/{id}
  GET    /social/friends/nearby
  GET    /social/chat/channels
  GET    /social/chat/{channel}/messages
  POST   /social/chat/{channel}/send

Analytics: GET    /analytics/me
  GET    /analytics/global
  GET    /analytics/events
```

### 7.2 WebSocket Protocol

```yaml
Connection: wss://ws.ore.game/v1/realtime

Authentication:
  → {"type": "auth", "token": "jwt_token"}
  ← {"type": "auth_result", "success": true}

Subscriptions:
  → {"type": "subscribe", "channels": ["location:cell:1234", "chat:global"]}
  ← {"type": "subscribed", "channels": ["location:cell:1234", "chat:global"]}

Client Events:
  → {"type": "location.update", "data": {"lat": 37.5, "lng": 127.0}}
  → {"type": "chat.message", "data": {"channel": "global", "text": "Hello"}}
  → {"type": "game.action", "data": {"action": "collect", "target": "coin_123"}}

Server Events:
  ← {"type": "coins.nearby", "data": [{"id": "coin_1", "value": 10, "location": {...}}]}
  ← {"type": "players.nearby", "data": [{"id": "player_1", "username": "Alice"}]}
  ← {"type": "chat.message", "data": {"from": "Bob", "text": "Hi!", "timestamp": "..."}}
  ← {"type": "quest.completed", "data": {"quest_id": "daily_1", "rewards": {...}}}
  ← {"type": "genesis.announcement", "data": {"message": "Special event!", "priority": "high"}}

Heartbeat:
  → {"type": "ping"}
  ← {"type": "pong"}
```

### 7.3 gRPC Internal APIs

```protobuf
syntax = "proto3";

// Location Service
service LocationService {
    rpc UpdateLocation(UpdateLocationRequest) returns (UpdateLocationResponse);
    rpc GetNearbyItems(GetNearbyRequest) returns (GetNearbyResponse);
    rpc ValidateMovement(ValidateMovementRequest) returns (ValidateMovementResponse);
}

message UpdateLocationRequest {
    string user_id = 1;
    double latitude = 2;
    double longitude = 3;
    float accuracy = 4;
    int64 timestamp = 5;
}

// Game Service
service GameService {
    rpc CollectCoin(CollectCoinRequest) returns (CollectCoinResponse);
    rpc GetPlayerState(GetPlayerStateRequest) returns (PlayerState);
    rpc ProcessQuest(ProcessQuestRequest) returns (ProcessQuestResponse);
}

// Blockchain Service
service BlockchainService {
    rpc SendRewards(SendRewardsRequest) returns (TransactionResult);
    rpc GetBalance(GetBalanceRequest) returns (Balance);
    rpc VerifyTransaction(VerifyTransactionRequest) returns (VerificationResult);
}
```

## 8. Security Architecture

### 8.1 Defense in Depth

```yaml
Layer 1 - Network:
  - CloudFront DDoS protection
  - AWS WAF rules
  - VPC isolation
  - Security groups (minimal ports)

Layer 2 - Application:
  - JWT authentication (RS256)
  - OAuth2 for social login
  - API rate limiting (per user/IP)
  - Input validation (all endpoints)

Layer 3 - Data:
  - Encryption in transit (TLS 1.3)
  - Encryption at rest (AES-256)
  - Database column encryption (PII)
  - Secure key management (AWS KMS)

Layer 4 - Monitoring:
  - Anomaly detection
  - Audit logging
  - Real-time alerts
  - Incident response plan
```

### 8.2 Anti-Cheat System

```rust
pub struct AntiCheatEngine {
    ml_model: Arc<MLModel>,
    rules_engine: Arc<RulesEngine>,
    reputation_system: Arc<ReputationSystem>,
}

impl AntiCheatEngine {
    // Multi-layer validation
    pub async fn validate_action(&self, action: &PlayerAction) -> ValidationResult {
        // 1. Rule-based checks
        let rule_result = self.rules_engine.check(action).await?;
        if !rule_result.is_valid {
            return ValidationResult::Rejected(rule_result.reason);
        }

        // 2. ML-based anomaly detection
        let ml_score = self.ml_model.predict(action).await?;
        if ml_score > 0.8 {
            return ValidationResult::Suspicious(ml_score);
        }

        // 3. Reputation check
        let reputation = self.reputation_system.get_score(action.player_id).await?;
        if reputation < 0.3 {
            return ValidationResult::LowTrust(reputation);
        }

        ValidationResult::Valid
    }
}

// Specific checks
impl AntiCheatEngine {
    // GPS spoofing detection
    pub async fn detect_spoofing(&self, movement: &Movement) -> bool {
        // Speed check (max 150 km/h)
        if movement.speed > 150.0 {
            return true;
        }

        // Acceleration check
        if movement.acceleration > 20.0 {
            return true;
        }

        // Pattern analysis (ML)
        let pattern_score = self.ml_model.analyze_pattern(movement).await;
        pattern_score > 0.9
    }

    // Bot detection
    pub async fn detect_bot(&self, actions: &[Action]) -> bool {
        // Click interval analysis
        let intervals = calculate_intervals(actions);
        let variance = calculate_variance(&intervals);

        // Too regular = bot
        if variance < 0.1 {
            return true;
        }

        // Action pattern analysis
        let pattern = extract_pattern(actions);
        self.ml_model.is_bot_pattern(&pattern).await
    }
}
```

## 9. Infrastructure & DevOps

### 9.1 AWS Infrastructure

```yaml
Compute:
  ECS Fargate:
    Cluster: ore-prod
    Services:
      - location-service: 2-10 tasks
      - game-service: 2-10 tasks
      - realtime-engine: 2-5 tasks
      - user-service: 1-5 tasks
      - auth-service: 1-5 tasks
      - ad-service: 1-5 tasks
      - analytics-service: 1-3 tasks
      - blockchain-service: 1-3 tasks

    Task Configuration:
      CPU: 512-4096
      Memory: 1024-8192
      Fargate Spot: 70% (non-critical)

Database:
  RDS PostgreSQL:
    Instance: db.r6g.large (MVP)
    Storage: 100GB SSD (autoscaling to 1TB)
    Multi-AZ: Yes (production)
    Read Replicas: 1 (can scale to 5)
    Backup: Continuous (PITR)

  ElastiCache Redis:
    Node type: cache.r6g.large
    Cluster mode: Enabled
    Shards: 3
    Replicas per shard: 1

  MSK (Kafka):
    Instance type: kafka.m5.large
    Brokers: 3
    Storage: 1TB per broker
    Version: 2.8.0

  TimeStream (Alternative to TimescaleDB):
    Memory store retention: 24 hours
    Magnetic store retention: 365 days

Storage:
  S3:
    Buckets:
      - ore-static-assets
      - ore-user-uploads
      - ore-backups
      - ore-analytics

  EBS:
    GP3 volumes for databases
    Snapshots: Daily

Networking:
  VPC:
    CIDR: 10.0.0.0/16
    Subnets: 6 (3 public, 3 private across 3 AZs)

  CloudFront:
    Origins: ALB, S3
    Behaviors: Cache static, pass dynamic

  Route 53:
    Hosted zones: ore.game
    Health checks: All critical endpoints
```

### 9.2 CI/CD Pipeline

```yaml
GitHub Actions Workflow:

name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - Checkout code
      - Run unit tests (Rust)
      - Run unit tests (Go)
      - Run integration tests
      - Security scan (Snyk)
      - Coverage report

  build:
    needs: test
    steps:
      - Build Rust services
      - Build Go services
      - Build Docker images
      - Push to ECR

  deploy:
    needs: build
    steps:
      - Update ECS task definitions
      - Blue/Green deployment
      - Run smoke tests
      - Rollback on failure

Infrastructure as Code:
  Tool: Terraform
  Structure:
    - modules/
      - networking/
      - compute/
      - database/
      - monitoring/
    - environments/
      - dev/
      - prod/
```

### 9.3 Monitoring & Observability

```yaml
Metrics (Prometheus + Grafana):
  System Metrics:
    - CPU, Memory, Disk, Network
    - Container metrics
    - Database performance

  Application Metrics:
    - Request rate, latency, errors
    - Business KPIs
    - Custom metrics

Logging (CloudWatch + ELK):
  Log Aggregation:
    - CloudWatch Logs for AWS services
    - Elasticsearch for application logs
    - Logstash for processing
    - Kibana for visualization

  Log Format:
    - JSON structured logging
    - Correlation IDs
    - User context

Tracing (X-Ray + Jaeger):
  Distributed Tracing:
    - End-to-end request flow
    - Service dependencies
    - Performance bottlenecks
    - Error tracking

Alerting:
  PagerDuty Integration:
    - P1: Service down, data loss risk
    - P2: High error rate, slow response
    - P3: Warnings, disk space

  Slack Notifications:
    - Deployments
    - Non-critical alerts
    - Daily summaries
```

## 10. Performance Optimization

### 10.1 Caching Strategy

```yaml
Multi-level Cache:
  L1 - Application Memory:
    - Hot data (< 1MB)
    - TTL: 1 minute
    - Hit rate target: 95%

  L2 - Redis:
    - Warm data (< 100MB)
    - TTL: 5-60 minutes
    - Hit rate target: 85%

  L3 - Database:
    - All data
    - Optimized indexes
    - Query cache enabled

Cache Patterns:
  Cache-aside:
    - Read: Check cache → miss → load from DB → update cache
    - Write: Update DB → invalidate cache
    - Use for: User profiles, game state

  Write-through:
    - Write: Update cache → update DB
    - Use for: Leaderboards, counters

  Write-behind:
    - Write: Update cache → async update DB
    - Use for: Analytics, logs
```

### 10.2 Database Optimization

```sql
-- Partitioning strategy
CREATE TABLE location_history_2025_09 PARTITION OF location_history
FOR VALUES FROM ('2025-09-01') TO ('2025-10-01');

-- Optimized queries with covering indexes
CREATE INDEX idx_covering_coins
ON coins(s2_cell_id, spawned_at, value)
INCLUDE (type, location)
WHERE collected_by IS NULL;

-- Materialized views for expensive queries
CREATE MATERIALIZED VIEW active_users_by_region AS
SELECT
    s2_cell_id,
    COUNT(DISTINCT user_id) as user_count,
    AVG(points) as avg_points
FROM users u
JOIN locations l ON u.id = l.user_id
WHERE l.recorded_at > NOW() - INTERVAL '1 hour'
GROUP BY s2_cell_id;

-- Connection pooling
-- PgBouncer configuration
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
```

## 11. Cost Analysis

### 11.1 Monthly Cost Breakdown

```yaml
MVP (1,000 users):
  Compute (ECS Fargate): $200
  Database (RDS): $150
  Cache (ElastiCache): $100
  Kafka (MSK): $150
  Storage (S3): $50
  Network (CloudFront): $50
  Monitoring: $50
  Total: ~$750/month

10K Users:
  Compute: $1,500
  Database: $500
  Cache: $300
  Kafka: $300
  Storage: $200
  Network: $500
  Monitoring: $200
  Total: ~$3,500/month

100K Users:
  Compute: $8,000
  Database: $2,000
  Cache: $1,000
  Kafka: $1,000
  Storage: $1,000
  Network: $3,000
  Monitoring: $1,000
  Total: ~$17,000/month

Cost Optimization:
  - Fargate Spot: 70% savings on compute
  - Reserved Instances: 40% savings on DB
  - S3 Intelligent Tiering: 30% savings on storage
  - CloudFront caching: 50% savings on bandwidth
```

## 12. Migration & Scaling Path

### 12.1 Future Architecture Evolution

```yaml
6 Months (10K users):
  Infrastructure:
    - Add read replicas
    - Enable auto-scaling
    - Multi-region CDN

  Architecture:
    - Add API Gateway (Kong)
    - Implement CQRS fully
    - Add event sourcing

1 Year (100K users):
  Infrastructure:
    - Multi-region deployment
    - Global database (Aurora Global)
    - Edge computing (Lambda@Edge)

  Architecture:
    - Microservices: 8 → 15
    - Add GraphQL Federation
    - Implement Saga pattern

2 Years (1M users):
  Infrastructure:
    - Full Kubernetes (EKS)
    - Custom ML pipeline
    - Dedicated blockchain nodes

  Architecture:
    - 30+ microservices
    - Service mesh (Istio)
    - Multi-cloud strategy
```

### 12.2 No-Downtime Migration Strategy

```yaml
Database Migration: 1. Set up replication to new database
  2. Sync data continuously
  3. Switch reads to new DB
  4. Switch writes to new DB
  5. Decommission old DB

Service Migration: 1. Deploy new service version
  2. Route 10% traffic (canary)
  3. Monitor metrics
  4. Gradually increase traffic
  5. Complete migration or rollback

Kafka Migration: 1. Set up MirrorMaker
  2. Replicate topics
  3. Switch consumers
  4. Switch producers
  5. Decommission old cluster
```

## 13. Development Guidelines

### 13.1 AI-Native Development Strategy

```yaml
Services Best Suited for AI Generation:
  High AI Leverage (80-90% AI):
    - CRUD operations
    - API endpoints
    - Database queries
    - Test cases
    - Documentation

  Medium AI Leverage (50-70% AI):
    - Business logic
    - Event handlers
    - Data transformations
    - Integration code

  Low AI Leverage (20-40% AI):
    - Anti-cheat algorithms
    - Performance optimization
    - Security critical code
    - Complex algorithms

Claude Code Prompts Structure:
  1. Context: Project ORE, service name, dependencies
  2. Requirements: Detailed specifications
  3. Constraints: Performance, security requirements
  4. Examples: Input/output samples
  5. Testing: Test cases to verify
```

### 13.2 Code Organization

```yaml
Repository Structure:
ore-platform/
├── services/
│   ├── location-service/    # Rust
│   ├── game-service/        # Rust
│   ├── realtime-engine/     # Rust
│   ├── blockchain-service/  # Rust
│   ├── user-service/        # Go
│   ├── auth-service/        # Go
│   ├── ad-service/          # Go
│   └── analytics-service/   # Go
├── shared/
│   ├── protobuf/           # Service definitions
│   ├── schemas/            # Kafka schemas
│   └── utils/              # Shared utilities
├── infrastructure/
│   ├── terraform/          # IaC
│   ├── k8s/               # Kubernetes manifests
│   └── scripts/           # Deployment scripts
└── docs/
    ├── api/               # API documentation
    ├── architecture/      # Architecture decisions
    └── runbooks/         # Operational guides

Service Template:
  - Dockerfile
  - Makefile
  - README.md
  - .env.example
  - /src (source code)
  - /tests (test files)
  - /docs (service docs)
```

## Conclusion

이 아키텍처는 AI-Native 개발 방식을 전제로 설계되었습니다. 복잡해 보이지만 Claude Code를 활용하면:

- **Kafka 설정**: 2-3시간
- **8개 서비스 스켈레톤**: 1일
- **Database 스키마**: 2시간
- **API 구현**: 서비스당 1-2일

총 12주 안에 충분히 구현 가능하며, 처음부터 제대로 만들어 **100만 사용자까지 재작성 없이** 확장 가능합니다.

---

_Version: 5.0_  
_Last Updated: 2024-12-20_  
_Architecture Decision Records (ADR) available in /docs/architecture/_
