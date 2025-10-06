# Project ORE - API Documentation Strategy v1.0

_AI-Native 개발로 구현하는 마이크로서비스 API 문서화 전략_

## Executive Summary

ORE Platform의 8개 마이크로서비스에 대한 **통합 API 문서화 전략**을 정의합니다. 하이브리드 접근 방식(분산 생성 + 중앙 표시)을 채택하여 개발자 경험과 유지보수성을 극대화합니다.

### 서비스 구성 (Service Architecture)

| 분류         | 서비스             | 언어 | 문서화 도구         | 엔드포인트                         |
| ------------ | ------------------ | ---- | ------------------- | ---------------------------------- |
| **Core**     | Location Service   | Rust | utoipa 5.4.0        | `/api-docs/openapi.json`           |
| **Core**     | Game Service       | Rust | utoipa 5.4.0        | `/api-docs/openapi.json`           |
| **Core**     | Realtime Service   | Rust | utoipa 5.4.0        | `/api-docs/openapi.json`           |
| **Core**     | Blockchain Service | Rust | utoipa 5.4.0        | `/api-docs/openapi.json`           |
| **Business** | Gateway Service    | Go   | swaggo/swag v1.16.4 | `/swagger/doc.json` (+ Aggregator) |
| **Business** | Auth Service       | Go   | swaggo/swag v1.16.4 | `/swagger/doc.json`                |
| **Business** | Ad Service         | Go   | swaggo/swag v1.16.4 | `/swagger/doc.json`                |
| **Business** | Analytics Service  | Go   | swaggo/swag v1.16.4 | `/swagger/doc.json`                |

### 핵심 전략

**접근 방식**: 각 서비스가 코드에서 자동으로 OpenAPI 스펙 생성 → Gateway가 통합하여 단일 Swagger UI 제공

**개발자 경험**: `http://localhost:8080/swagger` 하나의 URL로 모든 서비스 API 접근

**표준**: OpenAPI 3.1.0, JSON Schema 2020-12, JWT Bearer 인증

## 1. 개요 (Overview)

### 1.1 목적 및 목표 (Purpose & Goals)

**주요 목적:**

1. **통합 API 문서**: 8개 마이크로서비스의 API를 단일 인터페이스로 제공
2. **자동화**: 코드에서 문서 자동 생성으로 동기화 문제 제거
3. **개발자 생산성**: "Try it out" 기능으로 즉시 API 테스트 가능
4. **유지보수성**: 각 서비스가 자체 문서를 관리하여 결합도 최소화

**성공 지표:**

- 모든 공개 API 엔드포인트 95% 이상 문서화
- API 문서와 실제 구현 100% 일치 (자동 생성)
- 개발자 온보딩 시간 50% 단축
- API 관련 지원 요청 70% 감소

### 1.2 핵심 원칙 (Core Principles)

```yaml
원칙:
  1. 단일 진실의 원천 (Single Source of Truth):
    - 각 서비스는 자신의 OpenAPI 스펙을 소유
    - 코드에서 자동 생성 (code-first approach)

  2. 중앙 집중식 접근 (Centralized Access):
    - Gateway에서 모든 API 문서 통합
    - 개발자는 단일 URL만 기억: http://localhost:8080/swagger

  3. 자동화 우선 (Automation First):
    - 수동 YAML 작성 금지
    - 코드 어노테이션에서 자동 생성

  4. 최신 표준 준수 (Modern Standards):
    - OpenAPI 3.1.0 스펙 (2025년 표준)
    - JSON Schema 2020-12 지원
```

### 1.3 아키텍처 개요 (Architecture Overview)

```
┌─────────────────────────────────────────────────────────┐
│                    Developer Browser                    │
│            http://localhost:8080/swagger                │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────┐
        │      Gateway Service (Go/Fiber)      │
        │   Swagger Aggregator + Single UI     │
        └──────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Location     │   │ Auth Service │   │ Ad Service   │
│ Service      │   │ (Go/swaggo)  │   │ (Go/swaggo)  │
│ (Rust/utoipa)│   │              │   │              │
│              │   │ /docs.json   │   │ /docs.json   │
│ /api-docs    │   └──────────────┘   └──────────────┘
└──────────────┘
     │
     └─── 각 서비스는 자신의 OpenAPI 스펙 제공
          (Each service provides its own OpenAPI spec)
```

## 2. 문서화 아키텍처 (Documentation Architecture)

### 2.1 하이브리드 접근 방식 (Hybrid Approach)

**전략: 분산 생성 + 중앙 집중식 표시 (Distributed Generation + Centralized Display)**

#### 단계 1: 각 서비스에서 OpenAPI 스펙 생성

```yaml
서비스별 책임:
  Rust Services (4개):
    - location-service
    - game-service
    - realtime-service
    - blockchain-service
    도구: utoipa 5.4.0
    엔드포인트: /api-docs/openapi.json

  Go Services (4개):
    - gateway-service (특별: aggregator 역할)
    - auth-service
    - ad-service
    - analytics-service
    도구: swaggo/swag v1.16.4 (stable)
    엔드포인트: /swagger/doc.json
```

#### 단계 2: Gateway에서 통합 및 표시

```yaml
Gateway 역할:
  1. Swagger Aggregator:
    - 모든 서비스의 OpenAPI 스펙 수집
    - 단일 Swagger UI에서 통합 표시

  2. Service Discovery:
    - 동적으로 서비스 목록 관리
    - 새 서비스 추가 시 자동 감지

  3. Single Access Point:
    - URL: http://localhost:8080/swagger
    - 모든 서비스 API를 드롭다운에서 선택
```

### 2.2 왜 이 방식인가? (Why This Approach?)

```yaml
장점:
  서비스 자율성:
    - 각 팀이 독립적으로 API 문서 관리
    - 서비스 배포 시 문서 자동 업데이트

  개발자 경험:
    - 단일 URL에서 모든 API 탐색
    - "Try it out" 기능으로 즉시 테스트
    - 서비스 간 관계 파악 용이

  확장성:
    - 새 서비스 추가 시 aggregator 설정만 추가
    - 서비스별 문서는 독립적으로 확장

  최신 상태 유지:
    - 코드 변경 → 자동 문서 업데이트
    - 수동 동기화 불필요

업계 사례:
  - Uber: 마이크로서비스 300+ 개에 동일 패턴 사용
  - Netflix: API Gateway에서 통합 문서 제공
  - Spotify: 각 Squad가 자신의 스펙 관리, 중앙 포털에서 통합
```

## 3. 도구 및 구현 (Tools & Implementation)

### 3.1 Rust Services: utoipa

#### 버전 정보 (Version Information)

```toml
[dependencies]
utoipa = "5.4.0"              # Latest stable (2025-09)
utoipa-axum = "0.1.0"         # Axum integration
utoipa-swagger-ui = "8.0.3"   # Swagger UI 5.x
utoipa-scalar = "0.2.0"       # Modern alternative UI (optional)

# Optional: For advanced features
utoipa-redoc = "4.0.1"        # ReDoc UI
utoipa-rapidoc = "4.0.1"      # RapiDoc UI
```

#### 구현 예시: Location Service

```rust
// location-service/src/main.rs
use utoipa::{OpenApi, ToSchema};
use utoipa_swagger_ui::SwaggerUi;
use axum::{Router, routing::get};

// Step 1: Define OpenAPI structure with metadata
#[derive(OpenApi)]
#[openapi(
    info(
        title = "ORE Location Service API",
        version = "1.0.0",
        description = "GPS tracking, S2 geometry, anti-cheat validation",
        contact(
            name = "ORE Platform Team",
            email = "dev@oreprotocol.com"
        ),
        license(
            name = "MIT",
            url = "https://opensource.org/licenses/MIT"
        )
    ),
    servers(
        (url = "http://localhost:8081", description = "Local development"),
        (url = "https://api.ore.game/v1", description = "Production")
    ),
    paths(
        update_location,
        find_nearby,
        get_location_history,
    ),
    components(
        schemas(Location, UpdateLocationRequest, NearbyQuery, LocationResponse)
    ),
    tags(
        (name = "location", description = "Location tracking operations"),
        (name = "spatial", description = "Spatial search operations"),
        (name = "history", description = "Location history operations")
    )
)]
struct ApiDoc;

// Step 2: Define schema for request/response types
#[derive(Serialize, Deserialize, ToSchema)]
#[schema(example = json!({
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "latitude": 37.5665,
    "longitude": 126.9780,
    "accuracy": 10.5,
    "speed": 5.2
}))]
struct UpdateLocationRequest {
    /// User unique identifier (UUID v4)
    #[schema(format = "uuid")]
    user_id: uuid::Uuid,

    /// Latitude in decimal degrees (-90 to 90)
    #[schema(minimum = -90.0, maximum = 90.0)]
    latitude: f64,

    /// Longitude in decimal degrees (-180 to 180)
    #[schema(minimum = -180.0, maximum = 180.0)]
    longitude: f64,

    /// GPS accuracy in meters
    #[schema(minimum = 0.0)]
    accuracy: f64,

    /// Speed in meters per second (optional)
    #[schema(minimum = 0.0)]
    speed: Option<f64>,
}

#[derive(Serialize, Deserialize, ToSchema)]
struct Location {
    id: i64,
    user_id: uuid::Uuid,
    latitude: f64,
    longitude: f64,
    accuracy: f64,
    recorded_at: chrono::DateTime<chrono::Utc>,
}

// Step 3: Annotate handler functions with OpenAPI metadata
/// Update user location with anti-cheat validation
///
/// Validates GPS data, checks for speed hacking, and updates spatial index.
/// Rate limited to 2 requests per minute per user.
#[utoipa::path(
    post,
    path = "/location/update",
    request_body = UpdateLocationRequest,
    responses(
        (status = 200, description = "Location updated successfully", body = Location),
        (status = 400, description = "Invalid location data", body = ErrorResponse),
        (status = 429, description = "Rate limit exceeded", body = ErrorResponse),
        (status = 401, description = "Unauthorized", body = ErrorResponse)
    ),
    tag = "location",
    security(
        ("bearer_auth" = [])
    )
)]
async fn update_location(
    Json(request): Json<UpdateLocationRequest>,
) -> Result<Json<Location>, AppError> {
    // Implementation...
}

/// Find nearby locations using S2 spatial index
///
/// Uses hybrid spatial indexing (R-tree + S2 cells) for fast proximity search.
/// Maximum radius: 5000 meters. Maximum results: 100.
#[utoipa::path(
    get,
    path = "/location/nearby",
    params(
        ("latitude" = f64, Query, description = "Center latitude"),
        ("longitude" = f64, Query, description = "Center longitude"),
        ("radius" = f64, Query, description = "Search radius in meters (max 5000)"),
        ("limit" = Option<i32>, Query, description = "Max results (default 20, max 100)")
    ),
    responses(
        (status = 200, description = "Nearby locations found", body = Vec<Location>),
        (status = 400, description = "Invalid query parameters", body = ErrorResponse)
    ),
    tag = "spatial"
)]
async fn find_nearby(
    Query(query): Query<NearbyQuery>,
) -> Result<Json<Vec<Location>>, AppError> {
    // Implementation...
}

// Step 4: Setup Swagger UI and API documentation endpoints
#[tokio::main]
async fn main() {
    let app = Router::new()
        // API routes (version prefix in server URL)
        .route("/location/update", post(update_location))
        .route("/location/nearby", get(find_nearby))

        // Swagger UI at /swagger-ui/
        .merge(SwaggerUi::new("/swagger-ui")
            .url("/api-docs/openapi.json", ApiDoc::openapi()))

        // OpenAPI spec JSON endpoint (for Gateway aggregation)
        .route("/api-docs/openapi.json",
            get(|| async { Json(ApiDoc::openapi()) }));

    println!("🚀 Location Service running on http://localhost:8081");
    println!("📖 Swagger UI: http://localhost:8081/swagger-ui/");
    println!("📄 OpenAPI Spec: http://localhost:8081/api-docs/openapi.json");

    axum::Server::bind(&"0.0.0.0:8081".parse().unwrap())
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

#### 고급 기능 (Advanced Features)

```rust
// Security schemes definition
use utoipa::openapi::security::{Http, HttpAuthScheme, SecurityScheme};

impl ApiDoc {
    fn openapi_with_security() -> utoipa::openapi::OpenApi {
        let mut openapi = Self::openapi();

        // Add JWT Bearer authentication
        openapi.components.as_mut().unwrap().add_security_scheme(
            "bearer_auth",
            SecurityScheme::Http(Http::new(HttpAuthScheme::Bearer))
        );

        openapi
    }
}

// Custom error responses
#[derive(Serialize, ToSchema)]
struct ErrorResponse {
    #[schema(example = "VALIDATION_ERROR")]
    code: String,

    #[schema(example = "Invalid latitude value")]
    message: String,

    #[schema(example = json!({"field": "latitude", "value": 91.0}))]
    details: Option<serde_json::Value>,
}

// Enum documentation
#[derive(Serialize, Deserialize, ToSchema)]
#[schema(example = "active")]
enum UserStatus {
    #[serde(rename = "active")]
    Active,
    #[serde(rename = "suspended")]
    Suspended,
    #[serde(rename = "banned")]
    Banned,
}
```

### 3.2 Go Services: swaggo/swag

#### 버전 정보 (Version Information)

```bash
# Install Swag CLI
go install github.com/swaggo/swag/cmd/swag@v1.16.4

# For OpenAPI 3.x (experimental, use with caution)
# go install github.com/swaggo/swag/v2/cmd/swag@v2.0.0-rc4
```

```go
// go.mod
module github.com/ore-protocol/auth-service

go 1.23

require (
    github.com/gofiber/fiber/v2 v2.52.5
    github.com/gofiber/swagger v1.1.0       // Fiber Swagger middleware
    github.com/swaggo/swag v1.16.4          // Swagger generator
    github.com/swaggo/files v1.0.1          // Swagger UI assets
)
```

#### 구현 예시: Auth Service

```go
// auth-service/main.go
package main

import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/swagger"

    _ "github.com/ore-protocol/auth-service/docs" // Swagger generated docs
)

// @title           ORE Auth Service API
// @version         1.0
// @description     JWT authentication, OAuth2 social login, Genesis 1000 benefits
// @termsOfService  https://oreprotocol.com/terms/

// @contact.name   ORE Platform Team
// @contact.url    https://oreprotocol.com/support
// @contact.email  dev@oreprotocol.com

// @license.name  MIT
// @license.url   https://opensource.org/licenses/MIT

// @host      localhost:8084
// @BasePath  /api/v1

// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
// @description Type "Bearer" followed by a space and JWT token.

// @tag.name auth
// @tag.description Authentication operations (login, register, refresh)

// @tag.name oauth
// @tag.description OAuth2 social login (Discord, Twitter, Google)

// @tag.name genesis
// @tag.description Genesis 1000 member management

func main() {
    app := fiber.New()

    // Swagger UI endpoint
    app.Get("/swagger/*", swagger.HandlerDefault)

    // API routes
    app.Post("/api/v1/auth/login", loginHandler)
    app.Post("/api/v1/auth/register", registerHandler)
    app.Post("/api/v1/auth/refresh", refreshHandler)

    app.Listen(":8084")
}

// LoginRequest represents user login credentials
type LoginRequest struct {
    Email    string `json:"email" example:"user@example.com" validate:"required,email"`
    Password string `json:"password" example:"SecureP@ssw0rd" validate:"required,min=8"`
}

// TokenResponse represents JWT token pair
type TokenResponse struct {
    AccessToken  string `json:"access_token" example:"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."`
    RefreshToken string `json:"refresh_token" example:"550e8400-e29b-41d4-a716-446655440000"`
    ExpiresIn    int    `json:"expires_in" example:"86400"`
    TokenType    string `json:"token_type" example:"Bearer"`
    IsGenesis    bool   `json:"is_genesis" example:"true"`
}

// ErrorResponse represents API error response
type ErrorResponse struct {
    Error   string                 `json:"error" example:"Invalid credentials"`
    Code    string                 `json:"code" example:"AUTH_INVALID_CREDENTIALS"`
    Details map[string]interface{} `json:"details,omitempty"`
}

// Login authenticates user and returns JWT tokens
// @Summary      User login
// @Description  Authenticate with email/password and receive JWT tokens
// @Description  Genesis members receive 72-hour tokens (regular users: 24 hours)
// @Tags         auth
// @Accept       json
// @Produce      json
// @Param        request  body      LoginRequest  true  "Login credentials"
// @Success      200      {object}  TokenResponse
// @Failure      400      {object}  ErrorResponse  "Invalid request"
// @Failure      401      {object}  ErrorResponse  "Invalid credentials"
// @Failure      429      {object}  ErrorResponse  "Rate limit exceeded"
// @Router       /auth/login [post]
func loginHandler(c *fiber.Ctx) error {
    var req LoginRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(400).JSON(ErrorResponse{
            Error: "Invalid request body",
            Code:  "AUTH_INVALID_REQUEST",
        })
    }

    // Implementation...
    return c.JSON(TokenResponse{
        AccessToken:  "jwt_token_here",
        RefreshToken: "refresh_token_here",
        ExpiresIn:    86400,
        TokenType:    "Bearer",
        IsGenesis:    true,
    })
}

// RegisterRequest represents new user registration
type RegisterRequest struct {
    Email           string `json:"email" example:"user@example.com" validate:"required,email"`
    Password        string `json:"password" example:"SecureP@ssw0rd" validate:"required,min=8"`
    Username        string `json:"username" example:"ore_miner_42" validate:"required,min=3,max=20"`
    DiscordID       string `json:"discord_id,omitempty" example:"123456789012345678"`
    TwitterHandle   string `json:"twitter_handle,omitempty" example:"@ore_user"`
    ApplyForGenesis bool   `json:"apply_for_genesis" example:"true"`
}

// Register creates new user account
// @Summary      User registration
// @Description  Create new account with optional Genesis 1000 application
// @Description  If applying for Genesis, Discord verification required
// @Tags         auth
// @Accept       json
// @Produce      json
// @Param        request  body      RegisterRequest  true  "Registration data"
// @Success      201      {object}  TokenResponse
// @Failure      400      {object}  ErrorResponse  "Invalid request"
// @Failure      409      {object}  ErrorResponse  "Email already exists"
// @Failure      422      {object}  ErrorResponse  "Validation failed"
// @Router       /auth/register [post]
func registerHandler(c *fiber.Ctx) error {
    // Implementation...
    return c.Status(201).JSON(TokenResponse{})
}

// RefreshToken refreshes access token using refresh token
// @Summary      Refresh access token
// @Description  Get new access token using refresh token
// @Tags         auth
// @Accept       json
// @Produce      json
// @Param        refresh_token  body      object{refresh_token=string}  true  "Refresh token"
// @Success      200            {object}  TokenResponse
// @Failure      401            {object}  ErrorResponse  "Invalid refresh token"
// @Router       /auth/refresh [post]
func refreshHandler(c *fiber.Ctx) error {
    // Implementation...
    return c.JSON(TokenResponse{})
}

// OAuth2Callback handles OAuth2 callback from social providers
// @Summary      OAuth2 callback
// @Description  Handle OAuth2 callback from Discord/Twitter/Google
// @Tags         oauth
// @Accept       json
// @Produce      json
// @Param        provider  path      string  true  "OAuth provider (discord, twitter, google)"
// @Param        code      query     string  true  "Authorization code"
// @Param        state     query     string  true  "CSRF state token"
// @Success      200       {object}  TokenResponse
// @Failure      400       {object}  ErrorResponse  "Invalid OAuth callback"
// @Router       /auth/social/{provider}/callback [get]
func oauthCallbackHandler(c *fiber.Ctx) error {
    // Implementation...
    return c.JSON(TokenResponse{})
}

// GenesisStatusResponse represents Genesis 1000 member status
type GenesisStatusResponse struct {
    IsGenesis        bool    `json:"is_genesis" example:"true"`
    GenesisNumber    int     `json:"genesis_number" example:"42"`
    BonusMultiplier  float64 `json:"bonus_multiplier" example:"2.0"`
    AutoCollectRange int     `json:"auto_collect_range" example:"3"`
    TokenDuration    int     `json:"token_duration_hours" example:"72"`
    AirdropEligible  bool    `json:"airdrop_eligible" example:"true"`
    JoinedAt         string  `json:"joined_at" example:"2025-12-01T00:00:00Z"`
}

// GetGenesisStatus retrieves Genesis 1000 member status
// @Summary      Get Genesis status
// @Description  Retrieve Genesis 1000 membership status and benefits
// @Description  Genesis members receive: 2x rewards, 3m auto-collect range, 72-hour tokens
// @Tags         genesis
// @Accept       json
// @Produce      json
// @Security     BearerAuth
// @Success      200  {object}  GenesisStatusResponse
// @Failure      401  {object}  ErrorResponse  "Unauthorized"
// @Failure      404  {object}  ErrorResponse  "Not a Genesis member"
// @Router       /genesis/status [get]
func getGenesisStatusHandler(c *fiber.Ctx) error {
    // Extract user_id from JWT token
    userID := c.Locals("user_id").(string)

    // Implementation...
    return c.JSON(GenesisStatusResponse{
        IsGenesis:        true,
        GenesisNumber:    42,
        BonusMultiplier:  2.0,
        AutoCollectRange: 3,
        TokenDuration:    72,
        AirdropEligible:  true,
        JoinedAt:         "2025-12-01T00:00:00Z",
    })
}

// GenesisBenefitsResponse represents all Genesis benefits
type GenesisBenefitsResponse struct {
    RewardMultiplier float64  `json:"reward_multiplier" example:"2.0"`
    AutoCollectRange int      `json:"auto_collect_range_meters" example:"3"`
    TokenDuration    int      `json:"token_duration_hours" example:"72"`
    AirdropPercent   float64  `json:"airdrop_percent" example:"3.0"`
    ExclusiveItems   []string `json:"exclusive_items" example:"golden_pickaxe,genesis_badge"`
    PrioritySupport  bool     `json:"priority_support" example:"true"`
}

// GetGenesisBenefits retrieves all Genesis 1000 benefits
// @Summary      Get Genesis benefits
// @Description  List all benefits available to Genesis 1000 members
// @Tags         genesis
// @Accept       json
// @Produce      json
// @Success      200  {object}  GenesisBenefitsResponse
// @Router       /genesis/benefits [get]
func getGenesisBenefitsHandler(c *fiber.Ctx) error {
    return c.JSON(GenesisBenefitsResponse{
        RewardMultiplier: 2.0,
        AutoCollectRange: 3,
        TokenDuration:    72,
        AirdropPercent:   3.0,
        ExclusiveItems:   []string{"golden_pickaxe", "genesis_badge", "special_skin"},
        PrioritySupport:  true,
    })
}
```

#### Swagger 생성 및 통합 (Generate and Integrate)

```bash
# Step 1: Generate Swagger docs from code comments
cd auth-service
swag init

# This creates:
# docs/
#   ├── docs.go          # Generated Go code
#   ├── swagger.json     # OpenAPI spec (JSON)
#   └── swagger.yaml     # OpenAPI spec (YAML)

# Step 2: Build and run service
go build
./auth-service

# Step 3: Access Swagger UI
# http://localhost:8084/swagger/index.html
```

#### 고급 어노테이션 (Advanced Annotations)

```go
// Complex nested object documentation
type GenesisProfile struct {
    // Genesis member number (1-1000)
    // @example 42
    GenesisNumber int `json:"genesis_number"`

    // Bonus multiplier for rewards
    // @example 2.0
    BonusMultiplier float64 `json:"bonus_multiplier"`

    // Auto-collect range in meters
    // @example 3
    AutoCollectRange int `json:"auto_collect_range"`
}

// Array response documentation
// @Success 200 {array} User
// @Success 200 {object} PaginatedResponse{data=[]User}

// Multiple response content types
// @Produce json
// @Produce xml
// @Success 200 {object} User

// File upload
// @Param file formData file true "Avatar image (max 5MB)"
// @Accept multipart/form-data

// Query parameters with enums
// @Param status query string false "User status" Enums(active, suspended, banned)

// Header parameters
// @Param X-Request-ID header string false "Request tracking ID"
```

### 3.3 Gateway Service: Swagger Aggregator

**Gateway Service의 이중 역할 (Dual Role):**

Gateway Service는 ORE Platform에서 **두 가지 핵심 역할**을 수행합니다:

```yaml
역할 1: API Gateway (Routing & Security)
  기능:
    - 모든 클라이언트 요청의 진입점 (Entry Point)
    - 인증/인가 검증 (JWT validation)
    - Rate limiting 적용
    - 요청 라우팅 (Request routing to backend services)
    - CORS 정책 관리

  예시:
    Client → Gateway (Port 8080) → Backend Services (8081-8087)
    POST /api/v1/location/update → location-service:8081
    POST /api/v1/auth/login → auth-service:8084

역할 2: Documentation Aggregator (Unified Swagger UI)
  기능:
    - 모든 서비스의 OpenAPI 스펙 수집 및 통합
    - 단일 Swagger UI 제공 (http://localhost:8080/swagger)
    - 서비스별 스펙을 드롭다운에서 선택 가능
    - 개발자 경험 통합 (Single point of API exploration)

  예시:
    Developer → http://localhost:8080/swagger
    → Dropdown: "Location Service", "Auth Service", etc.
    → "Try it out" 기능으로 실제 API 테스트

통합 아키텍처:
  ┌─────────────────────────────────────────┐
  │         Gateway Service (8080)          │
  │                                         │
  │  ┌─────────────┐    ┌───────────────┐  │
  │  │ API Gateway │    │ Swagger       │  │
  │  │ (Routing)   │    │ Aggregator    │  │
  │  └─────────────┘    └───────────────┘  │
  │         │                    │          │
  └─────────┼────────────────────┼──────────┘
            │                    │
            ▼                    ▼
  ┌──────────────────┐  ┌──────────────────┐
  │ Backend Services │  │ OpenAPI Specs    │
  │ (8081-8087)     │  │ (/api-docs/...)  │
  └──────────────────┘  └──────────────────┘

왜 Gateway에서 통합하는가?:
  ✅ 개발자가 이미 Gateway URL을 알고 있음 (8080)
  ✅ API 호출과 문서 탐색을 동일한 위치에서 가능
  ✅ "Try it out" 기능이 실제 Gateway를 통해 동작
  ✅ 프로덕션 환경에서도 동일한 URL 구조 유지
  ✅ 인프라 단순화 (별도 문서 서버 불필요)
```

#### 구현: 통합 Swagger UI

```go
// gateway-service/main.go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"

    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/swagger"
)

// ServiceSpec represents OpenAPI spec from a service
type ServiceSpec struct {
    Name      string                 `json:"name"`
    URL       string                 `json:"url"`
    Spec      map[string]interface{} `json:"spec"`
    UpdatedAt time.Time              `json:"updated_at"`
}

// SwaggerAggregator collects OpenAPI specs from all services
type SwaggerAggregator struct {
    services map[string]*ServiceConfig
    specs    map[string]*ServiceSpec
    mu       sync.RWMutex
}

type ServiceConfig struct {
    Name    string
    BaseURL string
    DocsURL string
}

func NewSwaggerAggregator() *SwaggerAggregator {
    return &SwaggerAggregator{
        services: map[string]*ServiceConfig{
            "location": {
                Name:    "Location Service",
                BaseURL: getEnv("LOCATION_SERVICE_URL", "http://location-service:8081"),
                DocsURL: "/api-docs/openapi.json",
            },
            "game": {
                Name:    "Game Service",
                BaseURL: getEnv("GAME_SERVICE_URL", "http://game-service:8082"),
                DocsURL: "/api-docs/openapi.json",
            },
            "realtime": {
                Name:    "Realtime Service",
                BaseURL: getEnv("REALTIME_SERVICE_URL", "http://realtime-service:8083"),
                DocsURL: "/api-docs/openapi.json",
            },
            "blockchain": {
                Name:    "Blockchain Service",
                BaseURL: getEnv("BLOCKCHAIN_SERVICE_URL", "http://blockchain-service:8087"),
                DocsURL: "/api-docs/openapi.json",
            },
            "auth": {
                Name:    "Auth Service",
                BaseURL: getEnv("AUTH_SERVICE_URL", "http://auth-service:8084"),
                DocsURL: "/swagger/doc.json",
            },
            "ad": {
                Name:    "Ad Service",
                BaseURL: getEnv("AD_SERVICE_URL", "http://ad-service:8085"),
                DocsURL: "/swagger/doc.json",
            },
            "analytics": {
                Name:    "Analytics Service",
                BaseURL: getEnv("ANALYTICS_SERVICE_URL", "http://analytics-service:8086"),
                DocsURL: "/swagger/doc.json",
            },
        },
        specs: make(map[string]*ServiceSpec),
    }
}

// FetchSpecs fetches OpenAPI specs from all services concurrently
func (sa *SwaggerAggregator) FetchSpecs() error {
    var wg sync.WaitGroup

    for key, service := range sa.services {
        wg.Add(1)
        go func(k string, svc *ServiceConfig) {
            defer wg.Done()

            url := svc.BaseURL + svc.DocsURL
            resp, err := http.Get(url)
            if err != nil {
                return // Skip failed services
            }
            defer resp.Body.Close()

            var spec map[string]interface{}
            json.NewDecoder(resp.Body).Decode(&spec)

            sa.mu.Lock()
            sa.specs[k] = &ServiceSpec{
                Name:      svc.Name,
                URL:       url,
                Spec:      spec,
                UpdatedAt: time.Now(),
            }
            sa.mu.Unlock()
        }(key, service)
    }

    wg.Wait()
    return nil
}

// GetServiceList returns list of available services for Swagger UI dropdown
func (sa *SwaggerAggregator) GetServiceList(c *fiber.Ctx) error {
    sa.mu.RLock()
    defer sa.mu.RUnlock()

    services := make([]map[string]interface{}, 0, len(sa.specs))
    for key, spec := range sa.specs {
        services = append(services, map[string]interface{}{
            "name": spec.Name,
            "url":  fmt.Sprintf("/swagger/docs/%s", key),
        })
    }

    return c.JSON(services)
}

// GetServiceSpec returns OpenAPI spec for specific service
func (sa *SwaggerAggregator) GetServiceSpec(c *fiber.Ctx) error {
    service := c.Params("service")

    sa.mu.RLock()
    spec, exists := sa.specs[service]
    sa.mu.RUnlock()

    if !exists {
        return c.Status(404).JSON(fiber.Map{
            "error": fmt.Sprintf("Service '%s' not found", service),
            "code":  "SERVICE_NOT_FOUND",
        })
    }

    return c.JSON(spec.Spec)
}

// StartAutoRefresh refreshes specs every 5 minutes
func (sa *SwaggerAggregator) StartAutoRefresh() {
    ticker := time.NewTicker(5 * time.Minute)
    go func() {
        for range ticker.C {
            fmt.Println("🔄 Refreshing OpenAPI specs from all services...")
            if err := sa.FetchSpecs(); err != nil {
                fmt.Printf("❌ Failed to refresh specs: %v\n", err)
            } else {
                fmt.Println("✅ Successfully refreshed all OpenAPI specs")
            }
        }
    }()
}

func main() {
    app := fiber.New()
    aggregator := NewSwaggerAggregator()

    // Fetch initial specs and start auto-refresh
    aggregator.FetchSpecs()
    aggregator.StartAutoRefresh()

    // Swagger routes
    app.Get("/swagger/services", aggregator.GetServiceList)
    app.Get("/swagger/docs/:service", aggregator.GetServiceSpec)
    app.Get("/swagger/*", swagger.HandlerDefault)

    app.Listen(":8080")
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

#### Swagger UI 커스터마이징 (Customization)

```html
<!-- gateway-service/static/swagger-custom.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>ORE Platform API Documentation</title>
    <link
      rel="stylesheet"
      href="https://unpkg.com/swagger-ui-dist@5.10.0/swagger-ui.css"
    />
    <style>
      .swagger-ui .topbar {
        background-color: #1a1a2e;
        border-bottom: 3px solid #16213e;
      }
      .swagger-ui .topbar .download-url-wrapper {
        display: none;
      }
      .swagger-ui .info .title {
        color: #0f3460;
      }

      /* Service selector dropdown */
      #service-selector {
        position: fixed;
        top: 10px;
        right: 20px;
        z-index: 1000;
        padding: 10px;
        background: white;
        border-radius: 5px;
        box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
      }
    </style>
  </head>
  <body>
    <!-- Service selector dropdown -->
    <div id="service-selector">
      <label for="service-select">Select Service:</label>
      <select id="service-select" onchange="changeService(this.value)">
        <option value="">-- Choose Service --</option>
      </select>
    </div>

    <div id="swagger-ui"></div>

    <script src="https://unpkg.com/swagger-ui-dist@5.10.0/swagger-ui-bundle.js"></script>
    <script src="https://unpkg.com/swagger-ui-dist@5.10.0/swagger-ui-standalone-preset.js"></script>
    <script>
      let ui;

      // Load service list
      fetch("/swagger/services")
        .then((res) => res.json())
        .then((services) => {
          const select = document.getElementById("service-select");
          services.forEach((service) => {
            const option = document.createElement("option");
            option.value = service.url;
            option.textContent = service.name;
            select.appendChild(option);
          });

          // Load first service by default
          if (services.length > 0) {
            changeService(services[0].url);
          }
        });

      function changeService(url) {
        if (!url) return;

        ui = SwaggerUIBundle({
          url: url,
          dom_id: "#swagger-ui",
          deepLinking: true,
          presets: [SwaggerUIBundle.presets.apis, SwaggerUIStandalonePreset],
          plugins: [SwaggerUIBundle.plugins.DownloadUrl],
          layout: "StandaloneLayout",
          docExpansion: "list",
          filter: true,
          tryItOutEnabled: true,
        });
      }
    </script>
  </body>
</html>
```

## 4. 구현 가이드 (Implementation Guide)

### 4.1 개발 워크플로우 (Development Workflow)

```yaml
일일 개발 플로우:
  1. API 엔드포인트 개발:
    - Rust: 핸들러 함수 작성 + #[utoipa::path] 어노테이션
    - Go: 핸들러 함수 작성 + // @Summary 주석

  2. 문서 확인:
    - Rust: cargo run → http://localhost:808X/swagger-ui/
    - Go: swag init → go run main.go → http://localhost:808X/swagger/

  3. Gateway에서 통합 확인:
    - just dev (모든 서비스 시작)
    - http://localhost:8080/swagger/
    - 드롭다운에서 서비스 선택하여 API 테스트

  4. Commit:
    - Rust: 코드만 커밋 (docs/ 자동 생성)
    - Go: 코드 + docs/ 폴더 함께 커밋
```

### 4.2 코드 리뷰 체크리스트 (Code Review Checklist)

**모든 엔드포인트에 대해:**

- [ ] Summary와 Description이 명확한가?
- [ ] 모든 Request 파라미터가 문서화되었는가?
- [ ] 모든 Response 코드가 정의되었는가? (200, 400, 401, 404, 500)
- [ ] Request/Response 예시가 있는가?
- [ ] 인증이 필요한 경우 security 정의가 있는가?

**Rust (utoipa) 체크리스트:**

- [ ] #[derive(ToSchema)]가 모든 DTO에 적용되었는가?
- [ ] #[schema(example = ...)]가 복잡한 타입에 적용되었는가?
- [ ] OpenApi derive의 paths()에 모든 핸들러가 등록되었는가?
- [ ] components(schemas(...))에 모든 DTO가 등록되었는가?

**Go (swaggo) 체크리스트:**

- [ ] Package-level annotations (@title, @version, etc.)이 있는가?
- [ ] 모든 핸들러에 @Summary와 @Description이 있는가?
- [ ] @Param 어노테이션이 올바른 형식인가? (name, type, dataType, required, description)
- [ ] @Success/@Failure body 타입이 정확한가?
- [ ] swag init 실행 후 docs/ 폴더가 커밋되었는가?

**Gateway 통합 체크리스트:**

- [ ] 서비스가 SwaggerAggregator의 services 맵에 등록되었는가?
- [ ] /swagger/에서 새 서비스가 드롭다운에 나타나는가?
- [ ] 모든 엔드포인트가 정상적으로 테스트되는가?

## 5. 표준 및 규칙 (Standards & Conventions)

### 5.1 OpenAPI 스펙 규칙 (OpenAPI Spec Conventions)

```yaml
버전:
  openapi: "3.1.0"  # OpenAPI 3.1.0 사용 (2025년 표준)

메타데이터:
  info:
    title: "ORE [Service Name] API"
    version: "1.0.0"
    description: "[간단한 설명] + [주요 기능 3-5개]"
    contact:
      name: "ORE Platform Team"
      email: "dev@oreprotocol.com"
    license:
      name: "MIT"
      url: "https://opensource.org/licenses/MIT"

서버:
  servers:
    - url: "http://localhost:808X"
      description: "Local development"
    - url: "https://api.oreprotocol.com/{service}"
      description: "Production"
      variables:
        service:
          default: "location"
          description: "Service name"

태그:
  - 도메인별로 그룹화
  - 동사 대신 명사 사용 (예: "location", "auth", "fractures", "veins", "cores")
  - 각 태그에 description 필수

보안:
  - JWT Bearer 인증: "bearer_auth"
  - 모든 protected 엔드포인트에 security 적용
```

### 5.2 네이밍 규칙 (Naming Conventions)

```yaml
엔드포인트 경로 (2025 Industry Standard):
  - 버전은 서버 URL에 포함: https://api.ore.game/v1
  - 경로는 버전 없이 리소스만: /location/nearby, /auth/login
  - RESTful 패턴 준수
  - 복수형 명사 사용: /users, /fractures, /veins, /cores, /quests
  - 계층 구조: /users/{id}/inventory/items, /fractures/{id}/veins
  - 동작: POST /cores/{id}/mine (동사는 마지막에만)

  OpenAPI 구성:
    servers:
      - url: https://api.ore.game/v1
    paths:
      /location/nearby:        # Full URL: https://api.ore.game/v1/location/nearby
      /location/update:        # Full URL: https://api.ore.game/v1/location/update

  실제 엔드포인트 예시:
    GET    https://api.ore.game/v1/location/nearby-fractures  # 근처 Fracture 조회
    POST   https://api.ore.game/v1/location/update            # 위치 업데이트
    GET    https://api.ore.game/v1/fractures                  # Fracture 목록
    POST   https://api.ore.game/v1/fractures/{id}/enter       # Fracture 진입
    GET    https://api.ore.game/v1/veins/search               # Vein 탐색
    POST   https://api.ore.game/v1/cores/{id}/mine            # Core 채굴
    GET    https://api.ore.game/v1/users/me                   # 내 프로필

파라미터 이름:
  - snake_case 사용: user_id, genesis_number, fracture_id, vein_id, core_id
  - Boolean: is_*, has_*, can_* (예: is_genesis, has_premium)
  - 시간: *_at (예: created_at, discovered_at, collected_at)
  - 카운트: *_count (예: core_count, fracture_count, view_count)

Response 필드:
  - 성공: data, items, result
  - 에러: error, code, message, details
  - 페이지네이션: page, limit, total, has_next

Status Codes:
  200: Success (GET, PUT, PATCH)
  201: Created (POST)
  204: No Content (DELETE)
  400: Bad Request (validation error)
  401: Unauthorized (missing/invalid token)
  403: Forbidden (valid token, insufficient permissions)
  404: Not Found
  409: Conflict (duplicate resource)
  422: Unprocessable Entity (semantic error)
  429: Too Many Requests (rate limit)
  500: Internal Server Error
  503: Service Unavailable (maintenance, circuit breaker)
```

### 5.3 문서화 스타일 가이드 (Documentation Style Guide)

```yaml
Summary:
  - 40자 이내
  - 동사로 시작 (Get, Create, Update, Delete, List)
  - 예: "Get user profile", "Update location data"

Description:
  - 2-4 문장
  - 첫 문장: 엔드포인트 목적
  - 두 번째: 중요한 비즈니스 로직
  - 세 번째: 제약사항 또는 주의사항

  예시:
    "Updates user's GPS location with anti-cheat validation.
    Validates movement speed (max 150km/h) and acceleration patterns.
    Rate limited to 2 requests per minute per user."

Examples:
  - 실제 사용 가능한 예시 제공
  - UUID: 유효한 UUID v4 형식
  - Timestamp: ISO 8601 형식
  - 숫자: 현실적인 범위

  좋은 예시:
    {
      "user_id": "550e8400-e29b-41d4-a716-446655440000",
      "latitude": 37.5665,
      "longitude": 126.9780,
      "accuracy": 10.5,
      "recorded_at": "2025-09-15T14:30:00Z"
    }

에러 메시지:
  - code: SCREAMING_SNAKE_CASE (예: AUTH_INVALID_TOKEN)
  - message: 사용자 친화적 메시지 (한글 OK)
  - details: 추가 컨텍스트 (개발자용)

  예시:
    {
      "error": "Invalid location data",
      "code": "LOCATION_INVALID_DATA",
      "message": "위도 값이 유효한 범위를 벗어났습니다",
      "details": {
        "field": "latitude",
        "value": 91.0,
        "valid_range": "-90.0 to 90.0"
      }
    }
```

### 5.4 공통 에러 코드 레퍼런스 (Common Error Codes Reference)

모든 서비스에서 일관되게 사용하는 표준 에러 코드:

```yaml
인증/인가 에러 (Authentication/Authorization):
  AUTH_MISSING: "Authorization header missing"
  AUTH_INVALID_FORMAT: "Invalid authorization format (use 'Bearer <token>')"
  AUTH_INVALID_TOKEN: "Invalid or expired token"
  AUTH_INSUFFICIENT_PERMISSIONS: "Insufficient permissions for this action"
  AUTH_SERVICE_ERROR: "Authentication service unavailable"

검증 에러 (Validation):
  VALIDATION_ERROR: "Request validation failed"
  VALIDATION_REQUIRED_FIELD: "Required field missing"
  VALIDATION_INVALID_FORMAT: "Invalid field format"
  VALIDATION_OUT_OF_RANGE: "Value out of valid range"
  VALIDATION_DUPLICATE: "Duplicate value not allowed"

위치 서비스 에러 (Location Service):
  LOCATION_INVALID_DATA: "Invalid GPS coordinates"
  LOCATION_RATE_LIMIT: "Location update rate limit exceeded (max 2/min)"
  LOCATION_SPEED_VIOLATION: "Unrealistic movement speed detected (max 150km/h)"
  LOCATION_SPOOFING_DETECTED: "GPS spoofing detected"

게임 서비스 에러 (Game Service):
  GAME_FRACTURE_NOT_FOUND: "Fracture not found or depleted"
  GAME_FRACTURE_BOUNDARY: "Outside Fracture boundary (150m)"
  GAME_VEIN_NOT_FOUND: "Vein not discovered yet"
  GAME_CORE_NOT_FOUND: "Core not found or already mined"
  GAME_OUT_OF_RANGE: "Core mining out of range (max 10m)"
  GAME_INSUFFICIENT_ENERGY: "Insufficient energy"
  GAME_INVENTORY_FULL: "Inventory full"
  GAME_COOLDOWN_ACTIVE: "Fracture entry cooldown (5s)"
  GAME_SCENE_TRANSITION_BLOCKED: "Scene transition already in progress"

광고 서비스 에러 (Ad Service):
  AD_CAMPAIGN_NOT_FOUND: "Campaign not found"
  AD_BUDGET_EXHAUSTED: "Campaign budget exhausted"
  AD_INVALID_TARGETING: "Invalid targeting parameters"

Genesis 1000 에러:
  GENESIS_NOT_MEMBER: "Not a Genesis 1000 member"
  GENESIS_FULL: "Genesis 1000 slots full"
  GENESIS_VERIFICATION_FAILED: "Discord/Twitter verification failed"

시스템 에러 (System):
  RATE_LIMIT_EXCEEDED: "Too many requests (retry after X seconds)"
  SERVICE_UNAVAILABLE: "Service temporarily unavailable"
  SERVICE_CIRCUIT_OPEN: "Service circuit breaker open"
  GATEWAY_TIMEOUT: "Request timeout"
  BAD_GATEWAY: "Upstream service error"
  INTERNAL_ERROR: "Internal server error"
```

**에러 응답 표준 포맷:**

```json
{
  "error": "User-friendly error message",
  "code": "ERROR_CODE_CONSTANT",
  "message": "한글 설명 (선택적)",
  "details": {
    "field": "field_name",
    "value": "invalid_value",
    "additional_context": "..."
  },
  "request_id": "req_12345_67890",
  "timestamp": 1633024800
}
```

### 5.5 보안 문서화 (Security Documentation)

```yaml
JWT 인증:
  Rust (utoipa):
    security(("bearer_auth" = []))

  Go (swaggo):
    // @Security BearerAuth

인증 헤더 형식:
  Authorization: Bearer <JWT_TOKEN>

Genesis 멤버 특별 처리:
  - 문서에 명시: "Genesis members receive extended benefits"
  - Response 예시에 is_genesis 필드 포함
  - Rate limit 차이 명시: "Genesis: 120 req/min, Regular: 60 req/min"

Rate Limiting:
  - 헤더 문서화:
    X-RateLimit-Limit: 60
    X-RateLimit-Remaining: 45
    X-RateLimit-Reset: 1694876400

  - 429 Response 예시:
    {
      "error": "Rate limit exceeded",
      "code": "RATE_LIMIT_EXCEEDED",
      "retry_after": 60
    }
```

## 6. 프로덕션 고려사항 (Production Considerations)

### 6.1 환경별 URL 구성 (Environment-Specific URLs)

```yaml
개발 환경 (Development):
  Base URL: http://localhost:8080
  Swagger UI: http://localhost:8080/swagger/index.html
  특징:
    - 로컬 Docker Compose 사용
    - Hot reload 활성화
    - 상세한 에러 메시지
    - CORS 완전 개방

프로덕션 환경 (Production):
  Base URL: https://api.ore-protocol.com
  Swagger UI: https://api.ore-protocol.com/swagger/index.html
  특징:
    - AWS ECS Fargate 배포
    - Rate limiting 강화
    - 에러 메시지 일반화 (보안상)
    - CORS 허용 도메인 제한
    - HTTPS 전용 (HSTS 활성화)

환경 변수 설정:
  # .env.local (개발)
  API_BASE_URL=http://localhost:8080
  ENVIRONMENT=development
  LOG_LEVEL=debug
  RATE_LIMIT_ENABLED=false

  # .env.production (프로덕션)
  API_BASE_URL=https://api.ore-protocol.com
  ENVIRONMENT=production
  LOG_LEVEL=info
  RATE_LIMIT_ENABLED=true
  RATE_LIMIT_REQUESTS=60
  RATE_LIMIT_WINDOW=60s
```

### 6.2 Rate Limiting 문서화 (Rate Limiting Documentation)

```yaml
기본 Rate Limit 정책:
  Regular Users:
    Global: 60 requests/min
    Location Update: 2 requests/min
    Fracture Entry: 12 requests/min (5s cooldown)
    Core Mining: 10 requests/min
    Vein Discovery: 20 requests/min

  Genesis 1000 Members:
    Global: 120 requests/min (2x)
    Location Update: 4 requests/min (2x)
    Fracture Entry: 24 requests/min (5s cooldown)
    Core Mining: 20 requests/min (2x)
    Vein Discovery: 40 requests/min (2x)

Response Headers:
  X-RateLimit-Limit: 60 # 시간창 내 최대 요청 수
  X-RateLimit-Remaining: 45 # 남은 요청 수
  X-RateLimit-Reset: 1694876400 # 리셋 시간 (Unix timestamp)
  X-Genesis-Multiplier: 2.0 # Genesis 멤버만 포함

429 Rate Limit Response:
  {
    "error": "Rate limit exceeded",
    "code": "RATE_LIMIT_EXCEEDED",
    "retry_after": 60,
    "limit":
      {
        "max_requests": 60,
        "window_seconds": 60,
        "reset_at": "2025-10-01T12:00:00Z",
      },
  }

API 문서에 명시 방법:
  Rust (utoipa):
    #[utoipa::path(
    responses(
    (status = 429, description = "Rate limit exceeded (60 req/min, 120 for Genesis)")
    )
    )]

  Go (swaggo): // @Failure 429 {object} ErrorResponse "Rate limit exceeded (60 req/min, 120 for Genesis)"
```

### 6.3 API Versioning 전략 (API Versioning Strategy)

```yaml
버전 네이밍 (2025 Industry Standard):
  서버 URL: https://api.ore.game/v1
  엔드포인트: /location/update
  전체 URL: https://api.ore.game/v1/location/update

  - Major version은 서버 URL에 포함 (v1, v2, v3)
  - 엔드포인트 경로는 버전 없이 작성
  - Minor/Patch 변경은 URL 유지

Breaking Changes 정의:
  - 필드 제거 또는 이름 변경
  - Response 구조 변경
  - 필수 파라미터 추가
  - 데이터 타입 변경 (string → int)

Non-Breaking Changes:
  - 새 optional 필드 추가
  - 새 엔드포인트 추가
  - 에러 메시지 개선
  - 성능 최적화

버전 수명 주기:
  1. 새 버전 출시: v2 릴리스
  2. 병행 운영: v1과 v2 6개월 공존
  3. Deprecation 공지: v1에 deprecation 헤더 추가
  4. 마이그레이션 기간: 3개월 추가 운영
  5. Sunset: v1 완전 제거 (410 Gone 응답)

Deprecation Headers:
  Deprecation: true
  Sunset: Mon, 1 Mar 2026 00:00:00 GMT
  Link: <https://api.ore.game/v2/location>; rel="successor-version"

OpenAPI Specification:
  info:
    version: "1.2.3"
    description: |
      **Version History:**
      - v1.0.0 (2025-12-31): Initial MVP release
      - v1.1.0 (2026-03-15): S2 spatial index added
      - v1.2.0 (2026-06-01): Genesis 1000 benefits added

      **Deprecation Notice:**
      Some v1 endpoints will be deprecated after v2 release (planned 2026-12-31).
```

**URL Path Versioning의 강점 (Strengths of URL Path Versioning):**

이 방식을 선택한 이유와 장점:

```yaml
1. 명시적이고 투명함 (Explicit & Transparent):
   ✅ URL만 보고 API 버전 즉시 확인 가능
   ✅ 브라우저 개발자 도구, 로그에서 버전 명확
   ✅ 디버깅 시 혼란 없음

   예시: https://api.ore.game/v1/location/update
   → 누구나 v1 API임을 즉시 인식

2. 캐싱 최적화 (Caching Optimization):
   ✅ CDN/프록시에서 버전별 캐싱 가능
   ✅ v1과 v2가 독립적인 캐시 키 보유
   ✅ Cache-Control 헤더를 버전별로 설정 가능

   CloudFront 규칙:
   - /v1/* → TTL 1시간 (안정적인 버전)
   - /v2/* → TTL 5분 (새 버전, 자주 업데이트)

3. Gateway 라우팅 단순화 (Simplified Gateway Routing):
   ✅ URL prefix로 간단한 라우팅 규칙
   ✅ 버전별로 다른 백엔드 서비스 지정 가능
   ✅ 카나리 배포 시 트래픽 분할 용이

   Kong/AWS API Gateway 설정:
   routes:
     - path: /v1/*
       service: backend-v1-stable
     - path: /v2/*
       service: backend-v2-canary
       weight: 10%  # 10% 트래픽만 v2로

4. 클라이언트 SDK 생성 용이 (Easy Client SDK Generation):
   ✅ OpenAPI Generator가 버전별 SDK 자동 생성
   ✅ 클라이언트가 baseURL만 설정하면 됨
   ✅ 여러 버전 동시 사용 가능 (마이그레이션 중)

   Unity C# SDK:
   var v1Client = new OreApiClient("https://api.ore.game/v1");
   var v2Client = new OreApiClient("https://api.ore.game/v2");

5. 병행 운영 간편함 (Parallel Operation):
   ✅ v1과 v2를 동시에 운영하면서 점진적 마이그레이션
   ✅ 사용자별로 다른 버전 사용 가능
   ✅ Rollback 시 v1으로 즉시 복귀

   Blue-Green Deployment:
   - Genesis 1000: v2 먼저 테스트
   - Regular Users: v1 유지
   - 문제 발견 시 v1으로 rollback

6. 모니터링 및 분석 (Monitoring & Analytics):
   ✅ 버전별 트래픽 통계 쉽게 수집
   ✅ 버전별 에러율, 레이턴시 분리 측정
   ✅ 사용률 추적으로 Sunset 시기 결정

   Grafana 대시보드:
   - /v1/* requests/sec: 10K
   - /v2/* requests/sec: 500
   → v1 사용률 95%이므로 Sunset 연기 필요

7. 업계 표준 준수 (Industry Standard Compliance):
   ✅ 2025년 기준 70%+ 기업이 이 방식 채택
   ✅ 게임 업계 주요 플랫폼 표준 (Riot, Epic, Unity, Steam)
   ✅ 개발자들에게 익숙한 패턴
   ✅ 신규 팀원 온보딩 시 별도 설명 불필요

   Gaming Industry Adoption:
   - Riot Games (League of Legends): /lol/v4/*
   - Epic Games (Fortnite): /v1/games/fortnite/*
   - Unity Gaming Services: /v1/players/*
   - Steam Web API: /ISteamUser/v1/*

8. SEO 및 문서화 (SEO & Documentation):
   ✅ API 문서 URL이 명확하고 검색 가능
   ✅ 각 버전별로 별도 문서 페이지 생성
   ✅ 구글 검색 시 버전별로 구분되어 노출

   Documentation Structure:
   - docs.ore.game/v1/api-reference
   - docs.ore.game/v2/api-reference
   - docs.ore.game/migration-guide-v1-to-v2

대안 방식과의 비교:

헤더 기반 버전 관리 (Header-Based Versioning):
  ❌ URL에 버전 정보 없음 (디버깅 어려움)
  ❌ CDN 캐싱 복잡 (헤더 기반 캐시 키 필요)
  ❌ 브라우저에서 테스트 불편 (헤더 수동 추가)
  ❌ Swagger UI "Try it out" 사용 시 혼란

  Accept: application/vnd.ore.v1+json
  → 개발자가 헤더를 매번 명시해야 함

쿼리 파라미터 버전 관리 (Query Parameter Versioning):
  ❌ URL이 지저분함: /api/location?version=1
  ❌ 캐싱 효율 저하 (쿼리 파라미터 변동)
  ❌ 버전 누락 시 기본값 처리 복잡
  ❌ OpenAPI 표준에서 권장하지 않음

리소스별 버전 관리 (Per-Resource Versioning):
  ❌ 일관성 없음: /location/v1/update, /auth/v2/login
  ❌ Gateway 라우팅 규칙 복잡
  ❌ 전체 API 버전 관리 어려움
  ❌ 사용자가 어떤 리소스가 어느 버전인지 추적 힘듦

결론:
URL Path Versioning은 단순함, 명시성, 확장성의 완벽한 균형을 제공하며,
특히 마이크로서비스 아키텍처와 API Gateway 패턴에 최적화되어 있습니다.
```

### 6.4 프로덕션 배포 체크리스트 (Production Deployment Checklist)

```yaml
코드 준비: ✅ API 문서 생성 완료 (just docs-generate)
  ✅ OpenAPI 스펙 검증 (just docs-validate)
  ✅ Rate limiting 설정 확인
  ✅ CORS 허용 도메인 제한
  ✅ 환경 변수 프로덕션 설정

보안 검증: ✅ JWT 서명 키 교체 (개발용 키 사용 금지)
  ✅ HTTPS 전용 설정 (HSTS 활성화)
  ✅ 민감 정보 에러 메시지 제거
  ✅ SQL Injection 방어 확인
  ✅ Rate limiting 활성화

성능 테스트: ✅ Load test 100K req/sec (Location Service)
  ✅ Latency P99 < 50ms
  ✅ Redis cache hit rate > 85%
  ✅ Database connection pool 설정

모니터링 설정: ✅ CloudWatch metrics 활성화
  ✅ API Gateway access logs
  ✅ Error rate 알람 (> 1%)
  ✅ Latency 알람 (P99 > 100ms)

문서 배포: ✅ Swagger UI HTTPS 배포
  ✅ API 문서 버전 표시
  ✅ Rate limit 정보 명시
  ✅ 지원 채널 정보 추가
```

### 6.5 장애 대응 가이드 (Incident Response)

```yaml
API 문서 불일치 발견 시:
  1. GitHub Issue 생성: "API-DOC-MISMATCH: [Endpoint] actual vs documented"
  2. Hotfix 브랜치 생성
  3. 우선순위 결정:
     - Critical: 보안 이슈, 데이터 손실 가능성 → 즉시 배포
     - High: 핵심 기능 불일치 → 당일 배포
     - Medium: 부가 기능 불일치 → 다음 릴리스
  4. 문서 또는 코드 수정 (일관성 유지)
  5. 변경 로그 업데이트

Gateway Aggregator 장애 시:
  1. 개별 서비스 Swagger UI로 폴백:
     - Location: http://location-service:8081/api-docs/swagger-ui/
     - Auth: http://auth-service:8084/swagger/index.html
  2. CloudWatch 알람 확인
  3. 서비스별 health check: GET /health
  4. Redis 연결 상태 확인
  5. 로그 분석: just logs-gateway

버전 마이그레이션 실패 시:
  1. v1 엔드포인트 즉시 복구 (Rollback)
  2. v2 deprecation 일정 연기 공지
  3. 마이그레이션 가이드 개선
  4. 주요 클라이언트 개별 지원
```

## 7. 유지보수 및 자동화 (Maintenance & Automation)

### 7.1 CI/CD 통합 (CI/CD Integration)

```yaml
# justfile에 추가
docs-generate:
    #!/usr/bin/env bash
    echo "📖 Generating API documentation..."

    # Rust services (auto-generated at runtime)
    echo "✅ Rust services: docs generated at runtime"

    # Go services
    for service in auth-service ad-service analytics-service; do
        echo "Generating docs for $service..."
        cd backend/$service
        swag init
        cd ../..
    done

    echo "✅ All API docs generated"

docs-validate:
    #!/usr/bin/env bash
    echo "🔍 Validating OpenAPI specifications..."

    # Install validator if not exists
    if ! command -v openapi-validator &> /dev/null; then
        npm install -g ibm-openapi-validator
    fi

    # Validate all specs
    for spec in backend/*/docs/swagger.json; do
        echo "Validating $spec..."
        lint-openapi "$spec"
    done

    echo "✅ All specs validated"

docs-serve:
    @echo "📖 Starting unified API documentation server..."
    cd backend/gateway-service && go run main.go

# Pre-commit hook
pre-commit: docs-generate docs-validate
    @echo "✅ API docs generated and validated"
```

### 7.2 문서 버전 관리 (Documentation Versioning)

```yaml
전략:
  - API 버전: /api/v1, /api/v2 (URL 경로에 포함)
  - 문서 버전: OpenAPI info.version (SemVer)
  - 하위 호환성: v1은 최소 6개월 유지

버전 관리 규칙:
  Major (1.0.0 → 2.0.0):
    - Breaking changes
    - 필드 제거 또는 타입 변경
    - 새 URL 버전 필요 (/api/v2)

  Minor (1.0.0 → 1.1.0):
    - 새 엔드포인트 추가
    - 새 optional 필드 추가
    - 하위 호환성 유지

  Patch (1.0.0 → 1.0.1):
    - 버그 수정
    - 문서 개선
    - 성능 최적화

예시:
  Location Service v1.2.3:
    1: 초기 릴리스 (MVP)
    2: S2 spatial index 추가
    3: Anti-cheat 성능 개선
```

### 7.3 문서 품질 모니터링 (Documentation Quality Monitoring)

```yaml
자동화된 품질 체크:
  1. OpenAPI Spec Validation:
     - 도구: openapi-validator, spectral
     - 체크: 스펙 유효성, 네이밍 규칙, 보안 정의

  2. Coverage Check:
     - 모든 엔드포인트가 문서화되었는가?
     - 모든 Response 코드가 정의되었는가?
     - 모든 DTO에 example이 있는가?

  3. Example Validation:
     - Request/Response 예시가 스키마와 일치하는가?
     - UUID, timestamp 형식이 올바른가?

  4. Security Audit:
     - 인증이 필요한 엔드포인트에 security가 있는가?
     - Rate limit 정보가 문서화되었는가?

품질 메트릭스 및 KPI (Quality Metrics & KPIs):
  핵심 메트릭 (Core Metrics):
    1. Documentation Coverage:
       - 목표: > 95%
       - 측정: (Documented Endpoints / Total Endpoints) × 100
       - 계산 예시: Location Service - 47/50 endpoints = 94%
       - 액션: 미문서화 엔드포인트 우선순위화 및 문서화

    2. Example Completeness:
       - 목표: 100%
       - 측정: 모든 DTO에 @example 또는 example 필드 존재
       - 체크 방법: spectral 룰로 자동 검증
       - 중요도: High (개발자 이해도 직접 영향)

    3. Validation Errors:
       - 목표: 0 errors
       - 도구: openapi-validator, spectral
       - CI/CD에서 자동 체크 (blocking)
       - 허용: Warnings만 허용, Errors는 머지 불가

    4. Security Definitions:
       - 목표: 100%
       - 측정: 인증 필요 엔드포인트 중 security 정의 비율
       - 예외: Public 엔드포인트 (/health, /metrics)

  성능 메트릭 (Performance Metrics):
    1. Swagger UI Load Time:
       - 목표: < 2초 (초기 로드)
       - 측정: Time to Interactive (TTI)
       - 개선: Spec caching, CDN, Lazy loading

    2. Aggregator Fetch Time:
       - 목표: < 500ms (전체 서비스 스펙 수집)
       - 측정: Gateway aggregator parallel fetch
       - 개선: 병렬 요청, Redis 캐싱 (TTL: 5분)

    3. Spec File Size:
       - 목표: < 500KB per service (uncompressed)
       - 측정: openapi.json 파일 크기
       - 개선: 중복 스키마 제거, 압축 (gzip/brotli)

  동기화 메트릭 (Synchronization Metrics):
    1. Code-Doc Sync Rate:
       - 목표: 100% (코드와 문서 완전 일치)
       - 측정: 자동 생성 (utoipa/swag) → 항상 100%
       - 수동 문서 금지 정책

    2. Deployment Lag:
       - 목표: < 5분 (코드 배포 → 문서 업데이트)
       - 측정: Git commit timestamp vs Swagger UI update
       - 개선: CI/CD 자동화, Hot reload

  사용자 경험 메트릭 (User Experience Metrics):
    1. Developer Satisfaction:
       - 목표: > 4.0/5.0
       - 측정: 분기별 팀 설문조사
       - 질문: "API 문서가 개발에 도움이 되었나요?"

    2. API Documentation Usage:
       - 목표: 80% of developers use Swagger UI weekly
       - 측정: Gateway /swagger 엔드포인트 접근 로그
       - 개선: Onboarding 가이드, 사용 교육

    3. Issue Resolution Time:
       - 목표: < 24시간 (문서 버그 리포트 → 수정)
       - 측정: GitHub Issue 라벨 "api-docs" 평균 close 시간
       - 개선: 자동 검증 강화로 버그 사전 방지

  모니터링 대시보드 (Monitoring Dashboard):
    도구: Grafana + Prometheus
    패널:
      1. Real-time Metrics:
         - Swagger UI 접근 수 (시간당)
         - Aggregator fetch 성공률
         - Spec validation 에러 수

      2. Historical Trends:
         - Documentation coverage (주별)
         - API endpoint 증가 추이
         - Example completeness 추이

      3. Service Health:
         - 각 서비스 OpenAPI 엔드포인트 uptime
         - Gateway aggregator 응답 시간
         - Spec file size 추이

    알람 설정:
      - Coverage < 90%: Slack #api-docs (Warning)
      - Validation errors > 0: Block CI/CD (Critical)
      - Aggregator fetch time > 1s: Slack #devops (Warning)
      - Swagger UI downtime > 5min: PagerDuty (Critical)

  주간 리뷰 체크리스트 (Weekly Review):
    ✅ 모든 서비스 documentation coverage > 95%
    ✅ OpenAPI validation 에러 0개
    ✅ 새로 추가된 엔드포인트 문서화 완료
    ✅ Example completeness 100%
    ✅ Security definitions 100%
    ✅ Breaking changes 문서화 및 버전 업데이트
    ✅ Swagger UI 접근 로그 분석 (사용 패턴)
    ✅ 문서 관련 GitHub Issues 0건 (또는 진행 중)
```

### 7.4 팀 협업 가이드 (Team Collaboration Guide)

```yaml
역할별 책임:
  Backend Developer:
    - API 개발 시 문서 어노테이션 작성
    - PR 전 Swagger UI에서 테스트
    - Breaking changes 시 버전 업데이트

  Frontend Developer:
    - Swagger UI에서 API 스펙 확인
    - "Try it out"으로 엔드포인트 테스트
    - 불명확한 부분 Issue 생성

  QA Engineer:
    - API 문서와 실제 동작 일치 확인
    - 모든 Response 코드 시나리오 테스트
    - 문서 버그 리포트

  DevOps:
    - CI/CD 파이프라인에 문서 생성 통합
    - Gateway aggregator 모니터링
    - 문서 사이트 배포 (S3/CloudFront)

커뮤니케이션 채널:
  - Slack #api-docs: 문서 관련 질문
  - GitHub Issues: 문서 버그 리포트
  - Weekly API Review: 새 엔드포인트 리뷰
```

## 8. 고급 주제 (Advanced Topics)

### 8.1 API 버전 전환 전략 (API Version Migration)

```yaml
시나리오: Location Service v1 → v2 (Breaking Change)

1단계: v2 병행 운영 (Parallel Deployment)
   - https://api.ore.game/v1/location/* (기존)
   - https://api.ore.game/v2/location/* (신규)
   - Gateway에서 두 버전 동시 라우팅
   - 6개월 병행 운영

2단계: 사용자 마이그레이션 (User Migration)
   - v1 사용량 모니터링
   - 주요 클라이언트 개별 연락
   - v1 → v2 마이그레이션 가이드 제공

3단계: v1 Deprecation
   - v1 응답에 헤더 추가:
     Deprecation: true
     Sunset: Mon, 1 Mar 2026 00:00:00 GMT
     Link: <https://api.ore.game/v2/location/update>; rel="successor-version"

   - Swagger UI에 경고 표시:
     "⚠️ This endpoint is deprecated. Use v2 instead."

4단계: v1 제거 (Sunset)
   - v1 엔드포인트 제거
   - 410 Gone 응답:
     {
       "error": "This API version has been removed",
       "code": "API_VERSION_SUNSET",
       "successor": "https://api.ore.game/v2/location/update"
     }
```

### 8.2 Webhook 및 이벤트 문서화 (Webhook & Event Documentation)

```yaml
Webhook Callbacks:
  # utoipa에서 callback 문서화
  #[utoipa::path(
  callbacks(
  ("coreMined" = {
  "{$request.body.callback_url}" = {
  post(
  request_body = CoreMinedEvent,
  responses(
  (status = 200, description = "Webhook received")
  )
  )
  }
  }),
  ("fractureEntered" = {
  "{$request.body.callback_url}" = {
  post(
  request_body = FractureEnteredEvent,
  responses(
  (status = 200, description = "Webhook received")
  )
  )
  }
  })
  )
  )]

Kafka Events:
  # 별도 섹션에 이벤트 스키마 문서화
  - Topic: events.core.mined
  - Schema:
      {
        "event_id": "uuid",
        "player_id": "uuid",
        "fracture_id": "uuid",
        "vein_id": "uuid",
        "core_id": "uuid",
        "grade": "string",
        "value": "number",
        "mining_duration_ms": "number",
        "location": { "latitude": "number", "longitude": "number" },
        "timestamp": "iso8601",
      }

  - Topic: events.fracture.entered
  - Schema: {
        "event_id": "uuid",
        "player_id": "uuid",
        "fracture_id": "uuid",
        "entry_method": "string", # "geofencing" | "manual"
        "location": { "latitude": "number", "longitude": "number" },
        "timestamp": "iso8601",
      }

  - Topic: events.vein.discovered
  - Schema:
      {
        "event_id": "uuid",
        "player_id": "uuid",
        "fracture_id": "uuid",
        "vein_id": "uuid",
        "discovery_distance": "number",
        "timestamp": "iso8601",
      }
```

### 8.3 성능 최적화 (Performance Optimization)

```yaml
Gateway Aggregator 최적화:
  1. Spec Caching:
    - Redis에 각 서비스 스펙 캐시 (TTL: 5분)
    - 서비스 재시작 시 무효화

  2. Lazy Loading:
    - 초기 로드: 서비스 목록만
    - 사용자가 선택 시 스펙 로드

  3. CDN:
    - Swagger UI 정적 파일 → CloudFront
    - OpenAPI JSON → S3 + CloudFront

  4. 압축:
    - gzip 압축으로 스펙 크기 70% 감소
    - Brotli 압축으로 추가 15% 절감
```

## 9. 트러블슈팅 (Troubleshooting)

### 9.1 일반적인 문제 (Common Issues)

```yaml
문제 1: Swagger UI에서 "Failed to load API definition"
원인:
  - 서비스가 실행되지 않음
  - OpenAPI spec에 JSON 오류
  - CORS 설정 문제

해결:
  1. 서비스 health check: curl http://localhost:808X/health
  2. Spec 유효성 검사: curl http://localhost:808X/api-docs/openapi.json | jq
  3. 브라우저 콘솔에서 CORS 에러 확인

문제 2: swag init 실행 시 "cannot find package"
원인:
  - go.mod에 swag 의존성 없음
  - 어노테이션 형식 오류

해결:
  1. go mod tidy 실행
  2. 어노테이션 검증: swag init --parseDependency --parseInternal
  3. 에러 메시지 확인 후 수정

문제 3: utoipa 매크로 컴파일 에러
원인:
  - ToSchema trait 구현 누락
  - OpenApi derive의 paths에 핸들러 미등록

해결:
  1. 모든 DTO에 #[derive(ToSchema)] 추가
  2. ApiDoc의 paths()에 핸들러 함수 추가
  3. cargo expand로 매크로 확장 코드 확인

문제 4: Gateway에서 특정 서비스 스펙 로드 실패
원인:
  - 서비스 URL 잘못 설정
  - 서비스가 스펙 엔드포인트 제공 안 함

해결:
  1. SwaggerAggregator 로그 확인
  2. 서비스 URL 직접 테스트: curl http://service:8081/api-docs/openapi.json
  3. Gateway 환경변수 확인
```

### 8.2 디버깅 팁 (Debugging Tips)

```bash
# Rust: OpenAPI spec 출력
cargo run --bin print-openapi-spec

# Go: 생성된 swagger.json 검증
swag init --parseDependency
cat docs/swagger.json | jq . > /dev/null && echo "Valid JSON"

# Gateway: Aggregator 상태 확인
curl http://localhost:8080/swagger/services | jq

# OpenAPI spec 검증 (CLI)
npx @stoplight/spectral-cli lint backend/auth-service/docs/swagger.json

# Swagger UI 디버그 모드
# URL에 ?debug=true 추가: http://localhost:8080/swagger/?debug=true
```

## 9. 참고 자료 (References)

### 9.1 공식 문서 (Official Documentation)

```yaml
OpenAPI:
  - OpenAPI Specification 3.1.0: https://spec.openapis.org/oas/v3.1.0
  - JSON Schema 2020-12: https://json-schema.org/draft/2020-12/json-schema-core

Rust:
  - utoipa: https://docs.rs/utoipa/latest/utoipa/
  - utoipa-axum: https://docs.rs/utoipa-axum/latest/utoipa_axum/
  - utoipa examples: https://github.com/juhaku/utoipa/tree/master/examples

Go:
  - swaggo/swag: https://github.com/swaggo/swag
  - Declarative Comments: https://github.com/swaggo/swag#declarative-comments-format
  - gofiber/swagger: https://docs.gofiber.io/contrib/swagger/

Swagger UI:
  - Swagger UI 5.x: https://swagger.io/tools/swagger-ui/
  - Customization: https://swagger.io/docs/open-source-tools/swagger-ui/customization/
```

### 9.2 베스트 프랙티스 가이드 (Best Practices Guide)

```yaml
업계 표준:
  - Microsoft REST API Guidelines: https://github.com/microsoft/api-guidelines
  - Google API Design Guide: https://cloud.google.com/apis/design
  - Zalando RESTful API Guidelines: https://opensource.zalando.com/restful-api-guidelines/

마이크로서비스 문서화:
  - Netflix: https://netflixtechblog.com/how-netflix-uses-open-api-3-0-0-3f2ff73b6ed5
  - Uber: https://eng.uber.com/api-documentation/
  - Spotify: https://engineering.atspotify.com/2015/03/api-design-at-spotify/
```

### 9.3 도구 및 유틸리티 (Tools & Utilities)

```yaml
OpenAPI 검증:
  - Spectral: https://stoplight.io/open-source/spectral
  - openapi-validator: https://www.npmjs.com/package/ibm-openapi-validator
  - openapi-diff: https://github.com/OpenAPITools/openapi-diff

코드 생성:
  - OpenAPI Generator: https://openapi-generator.tech/
  - oapi-codegen (Go): https://github.com/deepmap/oapi-codegen
  - openapi-generator-cli: https://www.npmjs.com/package/@openapitools/openapi-generator-cli

UI 대안:
  - Scalar: https://github.com/scalar/scalar (Modern, fast)
  - ReDoc: https://github.com/Redocly/redoc
  - RapiDoc: https://github.com/rapi-doc/RapiDoc
  - Stoplight Elements: https://stoplight.io/open-source/elements
```

## 10. 체크리스트 및 템플릿 (Checklists & Templates)

### 10.1 새 서비스 추가 체크리스트 (New Service Checklist)

```markdown
## 새 마이크로서비스 API 문서화 체크리스트

### 1. 의존성 설정

- [ ] Rust: Cargo.toml에 utoipa, utoipa-swagger-ui 추가
- [ ] Go: go.mod에 gofiber/swagger, swaggo/swag 추가

### 2. 기본 설정

- [ ] OpenAPI 메타데이터 작성 (title, version, description)
- [ ] 서버 URL 정의 (local, staging, production)
- [ ] 태그 정의 및 설명 작성
- [ ] 보안 스키마 정의 (JWT Bearer)

### 3. 엔드포인트 문서화

- [ ] 모든 핸들러에 어노테이션 추가
- [ ] Request 파라미터 문서화 (query, path, body)
- [ ] Response 정의 (200, 400, 401, 404, 500)
- [ ] 예시 추가 (request, response)

### 4. DTO 문서화

- [ ] 모든 DTO에 schema 정의
- [ ] 필드별 설명 및 제약조건
- [ ] Example 값 추가

### 5. 로컬 테스트

- [ ] Swagger UI 접근 가능 (http://localhost:808X/swagger-ui/)
- [ ] OpenAPI JSON 유효성 검증
- [ ] "Try it out" 기능 테스트

### 6. Gateway 통합

- [ ] SwaggerAggregator에 서비스 등록
- [ ] Gateway Swagger UI에서 서비스 확인
- [ ] 모든 엔드포인트 Gateway를 통해 테스트

### 7. 문서화

- [ ] README에 API 문서 링크 추가
- [ ] CHANGELOG에 API 변경사항 기록

### 8. CI/CD

- [ ] API 문서 생성 스크립트 추가
- [ ] OpenAPI spec 검증 추가
- [ ] Pre-commit hook 설정
```

### 10.2 API 엔드포인트 템플릿

#### Rust (utoipa) 템플릿

```rust
/// [엔드포인트 설명 - 1줄]
///
/// [상세 설명 - 2-3줄]
/// [비즈니스 로직 또는 제약사항]
#[utoipa::path(
    post,  // HTTP method
    path = "/[resource]/[action]",
    request_body = [RequestType],
    responses(
        (status = 200, description = "[성공 시나리오]", body = [ResponseType]),
        (status = 400, description = "[클라이언트 에러]", body = ErrorResponse),
        (status = 401, description = "Unauthorized", body = ErrorResponse),
        (status = 500, description = "Internal server error", body = ErrorResponse)
    ),
    tag = "[tag_name]",
    security(
        ("bearer_auth" = [])
    )
)]
async fn handler_name(
    Json(request): Json<RequestType>,
) -> Result<Json<ResponseType>, AppError> {
    // Implementation
}
```

#### Go (swaggo) 템플릿

```go
// [핸들러 함수명] [간단한 설명]
// @Summary      [한 줄 요약]
// @Description  [상세 설명 - 2-3줄]
// @Description  [비즈니스 로직 또는 제약사항]
// @Tags         [tag_name]
// @Accept       json
// @Produce      json
// @Param        request  body      [RequestType]   true  "[요청 설명]"
// @Success      200      {object}  [ResponseType]  "[성공 시나리오]"
// @Failure      400      {object}  ErrorResponse   "[클라이언트 에러]"
// @Failure      401      {object}  ErrorResponse   "Unauthorized"
// @Failure      500      {object}  ErrorResponse   "Internal server error"
// @Security     BearerAuth
// @Router       /[resource]/[action] [post]
func handlerName(c *fiber.Ctx) error {
    // Implementation
    return c.JSON(response)
}
```

## 11. 결론 (Conclusion)

### 11.1 핵심 요약 (Key Takeaways)

```yaml
전략:
  ✅ 각 서비스가 자신의 OpenAPI 스펙 소유 (분산 생성)
  ✅ Gateway에서 통합 Swagger UI 제공 (중앙 집중식 표시)
  ✅ 코드에서 자동 생성 (code-first approach)
  ✅ 2025년 최신 도구 사용 (utoipa 5.4.0, swaggo 1.16.4)

도구:
  Rust → utoipa 5.4.0 + utoipa-swagger-ui
  Go   → swaggo/swag v1.16.4 + gofiber/swagger
  Gateway → Swagger Aggregator (커스텀 구현)

접근 포인트:
  개발자는 단일 URL만 기억: http://localhost:8080/swagger
  드롭다운에서 서비스 선택하여 API 탐색 및 테스트

유지보수:
  - CI/CD 자동화 (생성, 검증, 배포)
  - 코드 변경 시 문서 자동 업데이트
  - 버전 관리 및 마이그레이션 전략
```

### 11.2 구현 체크리스트 (Implementation Checklist)

**Rust 서비스 (Location, Game, Realtime, Blockchain):**

```markdown
☐ Cargo.toml에 utoipa 의존성 추가 (5.4.0)
☐ 모든 DTO에 #[derive(ToSchema)] 적용
☐ 핸들러에 #[utoipa::path] 어노테이션 추가
☐ ApiDoc struct 생성 및 paths/components 등록
☐ Swagger UI 라우트 추가 (/swagger-ui/)
☐ OpenAPI JSON 엔드포인트 추가 (/api-docs/openapi.json)
☐ 로컬 테스트: http://localhost:808X/swagger-ui/
☐ 보안 스키마 정의 (JWT Bearer)
☐ Request/Response 예시 추가
```

**Go 서비스 (Gateway, Auth, Ad, Analytics):**

```markdown
☐ swag CLI 설치 (v1.16.4)
☐ go.mod에 gofiber/swagger 추가
☐ main.go에 패키지 레벨 어노테이션 추가
☐ 핸들러에 주석 기반 문서 추가
☐ swag init 실행하여 docs/ 생성
☐ Swagger 미들웨어 추가
☐ 로컬 테스트: http://localhost:808X/swagger/
☐ docs/ 폴더 Git 커밋
☐ 모든 Response 코드 정의 (200, 400, 401, 500)
```

**Gateway 통합:**

```markdown
☐ SwaggerAggregator 구조체 구현
☐ 서비스 목록 설정 (services map)
☐ /swagger/services 엔드포인트 (서비스 목록)
☐ /swagger/docs/:service 엔드포인트 (개별 스펙)
☐ 통합 Swagger UI HTML 생성
☐ 자동 새로고침 로직 (5분마다)
☐ 전체 시스템 테스트: http://localhost:8080/swagger/
☐ 드롭다운에서 모든 서비스 확인
```

**자동화 및 품질:**

```markdown
☐ justfile에 docs-generate 명령 추가
☐ justfile에 docs-validate 명령 추가
☐ GitHub Actions 워크플로우 생성
☐ Pre-commit hook 설정 (terraform fmt, swag init)
☐ OpenAPI spec 검증 (spectral/validator)
☐ 비용 추정 (Infracost)
☐ PR에 plan 자동 게시
☐ 문서 사이트 배포 (S3/CloudFront)
```

**품질 검증:**

```markdown
☐ 모든 엔드포인트 문서화 (coverage 95%+)
☐ Request 파라미터 완전성
☐ Response 코드 정의 (최소 200, 400, 401, 500)
☐ 예시 값 추가 (request, response)
☐ 보안 정의 (JWT Bearer)
☐ 태그별 그룹화
☐ "Try it out" 기능 테스트
☐ Genesis 1000 특별 처리 문서화
```

### 11.3 지원 및 피드백 (Support & Feedback)

```yaml
문의:
  - Slack: #api-docs
  - Email: dev@oreprotocol.com
  - GitHub Issues: https://github.com/ore-protocol/ore-platform/issues

기여:
  - API 문서 개선 제안: Pull Request 환영
  - 버그 리포트: GitHub Issues
  - 모범 사례 공유: Slack #api-docs

리소스:
  - 이 문서: ore-docs/docs/technical/api-documentation-strategy.md
  - Backend Spec: ore-docs/docs/technical/backend-spec.md
  - 예제 코드: ore-platform/backend/*/examples/
```

---

**문서 버전**: 1.0.0
**최종 업데이트**: 2025-10-01
**작성자**: ORE Platform Team
**리뷰어**: [TBD]

**변경 이력**:

- 2025-10-01: 초안 작성 (v1.0.0)
  - 8개 마이크로서비스 API 문서화 전략 정의
  - Rust (utoipa 5.4.0) + Go (swaggo 1.16.4) 통합 접근
  - Gateway Swagger Aggregator 아키텍처
  - OpenAPI 3.1.0 표준 준수
  - URL Path Versioning 채택 (2025 industry standard)
  - Production considerations, error codes, metrics/KPIs 포함
  - Backend Spec 및 Backend AI Guide와 크로스 레퍼런스
