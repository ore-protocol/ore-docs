# Project ORE - Infrastructure System Specification v1.1

_AI-Native IaC로 구현하는 하이브리드 클라우드 인프라_

## Executive Summary

### 프로젝트 컨텍스트

Project ORE의 인프라는 **AWS 클라우드(85%)**와 **온프레미스(15%)**의 하이브리드 아키텍처로 구성됩니다. MVP 단계에서는 관리 부담을 최소화하는 매니지드 서비스 중심으로 시작하여, 성장에 따라 점진적으로 복잡도를 증가시키는 전략을 채택합니다.

### 핵심 설계 철학

```yaml
"Simple Start, Scale Smart, Automate Everything"

원칙:
  1. Cloud-Native First: 매니지드 서비스 우선 활용
  2. Progressive Complexity: 필요할 때만 복잡도 추가
  3. Cost Optimization: 스팟 인스턴스와 서버리스로 비용 절감
  4. Infrastructure as Code: 모든 인프라를 Terraform으로 관리
  5. Security by Default: 제로 트러스트 보안 모델
```

### 인프라 전략

Project ORE의 인프라는 **초기 비용과 운영 복잡성을 최소화**하기 위해 AWS의 서버리스 및 매니지드 서비스를 적극 활용합니다. **ECS Fargate**로 빠르게 시작하고, **Aurora/MSK Serverless**로 예측 불가능한 MVP 트래픽에 유연하게 대응합니다.

동시에, **Polygon 풀노드**와 같이 핵심적인 안정성이 요구되는 부분은 **전략적으로 온프레미스**에 배치하여 외부 의존성을 제거하고 장기적인 비용 효율을 확보하는 하이브리드 전략을 채택했습니다.

모든 인프라는 Terraform 코드로 관리되며, 서비스 성장에 따라 **EKS로 마이그레이션**할 수 있는 명확한 로드맵을 가집니다.

```yaml
기술 스택:
  - IaC: Terraform + Ansible
  - Container: ECS Fargate → EKS (Phase 3)
  - Serverless: Lambda, Aurora Serverless, MSK Serverless
  - Monitoring: CloudWatch + Prometheus + Grafana
  - CI/CD: GitHub Actions + ArgoCD

확장 목표:
  - MVP: $500-800/월 (1K users)
  - 6개월: $3,000-5,000/월 (10K users)
  - 1년: $15,000-25,000/월 (100K users)
  - 비용 대비 10x 사용자 처리

2025년 트렌드 반영:
  - Platform Engineering 도입
  - FinOps 자동화
  - AI-Driven Scaling
  - Edge Computing Ready
```

### 핵심 설계 결정 및 Trade-offs

```yaml
주요 결정사항:
  ECS vs EKS:
    선택: ECS Fargate (MVP)
    이유: K8s 복잡성 없이 빠른 시작
    전환: Phase 3에서 EKS로 마이그레이션

  Managed vs Self-hosted:
    선택: 95% Managed Services
    이유: 1인 개발자가 인프라보다 비즈니스에 집중
    예외: 블록체인 노드 (안정성), GPU (비용)

  Single vs Multi-region:
    선택: Single region (N. California) 시작
    이유: 복잡성과 비용 최소화
    확장: 10K users 도달 시 DR 구성
```

## 1. Infrastructure Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                        Users                            │
│                    (Global Traffic)                     │
└─────────────────────────────────────────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   CloudFront CDN  │
                    │  (Edge Locations) │
                    └─────────┬─────────┘
                              │
                    ┌─────────▼─────────┐
                    │    AWS WAF        │
                    │ (DDoS Protection) │
                    └─────────┬─────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │            AWS Cloud Region         │
           │            (us-west-1)              │
           │                                     │
           │  ┌────────────VPC─────────────┐     │
           │  │     10.0.0.0/16            │     │
           │  │                            │     │
           │  │  ┌──Public Subnets───┐     │     │
           │  │  │  10.0.1-3.0/24    │     │     │
           │  │  │  - NAT Gateways   │     │     │
           │  │  │  - Load Balancers │     │     │
           │  │  └───────────────────┘     │     │
           │  │                            │     │
           │  │  ┌──Private Subnets──┐     │     │
           │  │  │  10.0.11-13.0/24  │     │     │
           │  │  │  - ECS Fargate    │     │     │
           │  │  │  - Lambda         │     │     │
           │  │  └───────────────────┘     │     │
           │  │                            │     │
           │  │  ┌──Database Subnets─┐     │     │
           │  │  │  10.0.21-23.0/24  │     │     │
           │  │  │  - Aurora         │     │     │
           │  │  │  - ElastiCache    │     │     │
           │  │  │  - MSK            │     │     │
           │  │  └───────────────────┘     │     │
           │  └────────────────────────────┘     │
           └─────────────────────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  Site-to-Site VPN │
                    │   (IPSec Tunnel)  │
                    └─────────┬─────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │          On-Premise DC              │
           │                                     │
           │  ┌──Blockchain Node─┐  ┌──GPU──┐    │
           │  │ Polygon Fullnode │  │  ML   │    │
           │  └──────────────────┘  └───────┘    │
           └─────────────────────────────────────┘
```

### 1.2 AWS Cloud Infrastructure

#### 1.2.1 Network Architecture

```yaml
VPC Design:
  CIDR Block: 10.0.0.0/16

  Availability Zones: 3 (us-west-1a/1b/1c)

  Subnets:
    Public (10.0.1-3.0/24):
      - NAT Gateways (HA)
      - Application Load Balancers
      - Bastion Hosts (필요시)

    Private (10.0.11-13.0/24):
      - ECS Fargate Tasks
      - Lambda Functions
      - EC2 Instances (필요시)

    Database (10.0.21-23.0/24):
      - RDS/Aurora Instances
      - ElastiCache Clusters
      - MSK Brokers
      - No direct internet access

  Security Groups:
    sg-alb:
      - Inbound: 80, 443 from 0.0.0.0/0
      - Outbound: All to sg-ecs

    sg-ecs:
      - Inbound: Dynamic ports from sg-alb
      - Outbound: All (필요한 포트만 제한)

    sg-database:
      - Inbound: 5432 from sg-ecs
      - Inbound: 6379 from sg-ecs
      - Outbound: None

  Network ACLs:
    - Default로 시작, 필요시 추가 제한

  VPC Endpoints:
    - S3: Gateway Endpoint
    - ECR: Interface Endpoint
    - Secrets Manager: Interface Endpoint
    - CloudWatch: Interface Endpoint
```

**설계 근거:**

**3 AZ 배포 이유:** AWS N. California 리전의 3개 가용 영역을 모두 활용하여 최대한의 가용성을 확보합니다. 2 AZ만 사용할 경우 33% 가용성 손실이 발생하지만, 3 AZ는 단일 AZ 장애 시에도 66%의 용량을 유지할 수 있습니다.

**서브넷 티어링 전략:**

- Public 서브넷을 최소화하여 공격 표면을 줄입니다
- Database 서브넷을 완전히 격리하여 직접 접근을 차단합니다
- Private 서브넷에서만 컴퓨팅 리소스를 실행하여 보안을 강화합니다

**VPC Endpoints 선택:** S3와 ECR로의 데이터 전송 비용을 절감하고 (월 $50-100 절약), 인터넷 게이트웨이를 거치지 않아 보안과 성능이 향상됩니다.

**고려했던 대안:**

- **Transit Gateway vs VPC Peering:** 초기에는 단일 VPC로 충분하므로 Transit Gateway의 추가 비용($72/월)을 피했습니다
- **NAT Instance vs NAT Gateway:** 고가용성과 관리 편의성을 위해 NAT Gateway를 선택했습니다 (Phase 3에서 재검토 예정)

#### 1.2.2 Compute Resources

```yaml
ECS Cluster:
  Name: ore-cluster-prod

  Capacity Providers:
    FARGATE:
      - Weight: 1
      - Base: 0

    FARGATE_SPOT:
      - Weight: 4 # 80% Spot 목표
      - Base: 2 # 최소 2개는 On-Demand

  Services:
    Location Service (Rust):
      Launch Type: FARGATE_SPOT
      CPU: 1024 (1 vCPU)
      Memory: 2048 (2 GB)
      Desired Count: 2
      Auto Scaling:
        Min: 2
        Max: 10
        Target CPU: 70%

    Game Service (Rust):
      Launch Type: FARGATE_SPOT
      CPU: 2048 (2 vCPU)
      Memory: 4096 (4 GB)
      Desired Count: 2
      Auto Scaling:
        Min: 2
        Max: 10
        Target Memory: 80%

    Realtime Engine (Rust):
      Launch Type: FARGATE # 안정성 중요
      CPU: 2048 (2 vCPU)
      Memory: 4096 (4 GB)
      Desired Count: 3
      Auto Scaling:
        Min: 3
        Max: 15
        Target Connections: 1000

    Auth Service (Go):
      Launch Type: FARGATE_SPOT
      CPU: 512 (0.5 vCPU)
      Memory: 1024 (1 GB)
      Desired Count: 2
      Auto Scaling:
        Min: 1
        Max: 5
        Target CPU: 60%

Lambda Functions:
  Image Processing:
    Runtime: Python 3.11
    Memory: 1024 MB
    Timeout: 30s
    Reserved Concurrency: 10

  Webhook Handler:
    Runtime: Node.js 20.x
    Memory: 512 MB
    Timeout: 10s

  Scheduled Jobs:
    Runtime: Go 1.x
    Memory: 256 MB
    Timeout: 5m
```

**설계 근거:**

**ECS Fargate 선택 이유:**
MVP 단계에서는 Kubernetes(EKS)의 복잡성과 운영 오버헤드 없이, 서버리스의 단순함과 빠른 배포, 자동 확장의 이점을 누리기 위해 ECS Fargate를 선택했습니다. 이는 'Progressive Complexity' 원칙에 따라 초기 개발 속도에 집중하기 위함입니다.

**Rust vs Go 서비스 차등 설정:**

- **Rust 서비스 (Location, Game):** 성능이 핵심이므로 더 많은 CPU와 메모리를 할당하고 Fargate Spot을 적극 활용하여 비용 대비 효율을 극대화합니다 (70% 비용 절감)
- **Realtime Engine:** WebSocket 연결의 안정성이 중요하므로 On-Demand Fargate를 사용하여 예기치 않은 중단을 방지합니다
- **Go 서비스 (Auth):** 상대적으로 가벼운 워크로드이므로 최소 리소스로 시작합니다

**80/20 Spot 전략:**
Base 2개의 On-Demand 태스크로 기본 가용성을 보장하면서, 나머지 80%를 Spot으로 운영하여 월 $300-500을 절약합니다. Spot 중단 시에도 서비스가 완전히 중단되지 않습니다.

**Lambda 사용 케이스:**

- **정기 작업:** ECS 태스크보다 90% 저렴
- **이벤트 기반 처리:** 콜드 스타트가 허용되는 비동기 작업
- **버스트 워크로드:** 예측 불가능한 트래픽 처리

**고려했던 대안:**

- **ECS EC2 vs Fargate:** 초기에는 EC2 인스턴스 관리 부담을 피하고자 Fargate 선택. 월 1000 태스크 시간 초과 시 EC2 전환 검토
- **App Runner vs ECS:** 더 세밀한 제어와 비용 효율성을 위해 ECS 선택

#### 1.2.3 Data Layer

```yaml
Aurora PostgreSQL:
  Engine Version: 15.4

  Serverless v2 Configuration:
    Min ACUs: 0.5 # $40/월
    Max ACUs: 4 # $320/월

  High Availability:
    - Multi-AZ deployment
    - 1 Writer + 1 Reader
    - Auto failover < 30s

  Backup:
    - Automated backups: 7 days
    - Point-in-time recovery
    - Snapshot on demand

  Performance Insights: Enabled
  Enhanced Monitoring: 60 second intervals

ElastiCache Redis:
  Engine Version: 7.1

  Serverless Configuration:
    Max ECPUs: 10000
    Max Storage: 100 GB

  Cluster Mode: Enabled
  Automatic Failover: Enabled

  Backup:
    - Daily snapshots
    - Retention: 3 days

Amazon MSK (Kafka):
  Version: 3.5.1

  Serverless Configuration:
    - Auto scaling
    - Pay per throughput

  Topics:
    - Retention: 7 days (default)
    - Replication Factor: 3
    - Min In-Sync Replicas: 2

  Security:
    - TLS encryption
    - IAM authentication

Amazon Timestream:
  Memory Store Retention: 24 hours
  Magnetic Store Retention: 365 days

  Tables:
    - game_events
    - location_tracking
    - ad_metrics

S3 Buckets:
  ore-static-assets:
    - Versioning: Enabled
    - Lifecycle: Intelligent-Tiering
    - Replication: Cross-region (DR)

  ore-user-uploads:
    - Encryption: SSE-S3
    - CORS: Configured
    - Transfer Acceleration: Enabled

  ore-backups:
    - Lifecycle: Glacier after 30 days
    - Object Lock: Enabled
```

**설계 근거:**

**Aurora Serverless v2 선택 이유:**
MVP 단계의 불규칙한 트래픽 패턴에 완벽하게 대응합니다. 0.5 ACU($40/월)로 시작하여 트래픽이 없을 때는 비용을 최소화하고, 피크 시 4 ACU까지 자동 확장하여 성능을 보장합니다. 전통적인 RDS 대비 60-80% 비용 절감이 가능합니다.

**ElastiCache Serverless 채택:**
기존 클러스터 모드는 최소 3개 노드가 필요해 월 $150 이상이지만, Serverless는 사용한 만큼만 과금되어 MVP에 이상적입니다. 자동 샤딩과 리밸런싱으로 운영 부담도 제거됩니다.

**MSK Serverless vs Self-managed Kafka:**

- **MSK Serverless 선택:** 3개 브로커 최소 요구사항과 운영 복잡성을 제거
- **비용:** 초기 트래픽에서는 월 $50 수준으로 시작 가능
- **전환 시점:** 월 100GB 이상 처리 시 전용 클러스터 검토

**Timestream vs TimescaleDB:**

- **Timestream 선택:** 완전 관리형으로 운영 부담 없음
- **자동 티어링:** Hot(메모리) → Cold(마그네틱) 자동 전환
- **비용:** TimescaleDB on RDS 대비 70% 저렴
- **제약:** 복잡한 SQL 쿼리 제한 (향후 재검토 필요)

**S3 스토리지 클래스 전략:**

- **Intelligent-Tiering:** 접근 패턴을 자동 학습하여 비용 최적화 (30% 절감)
- **Transfer Acceleration:** 글로벌 사용자를 위해 업로드 속도 50% 개선
- **Object Lock:** 규제 준수와 랜섬웨어 방어

**고려했던 대안:**

- **DynamoDB vs Aurora:** 복잡한 관계형 쿼리와 트랜잭션 요구사항으로 Aurora 선택
- **Memcached vs Redis:** Pub/Sub, 지속성, 복잡한 데이터 구조 지원으로 Redis 선택
- **Kinesis vs MSK:** Kafka 에코시스템의 풍부한 도구와 커뮤니티 지원으로 MSK 선택

### 1.3 On-Premise Infrastructure

```yaml
Hardware Specifications:
  Blockchain Node Server:
    Model: Dell PowerEdge R750xs
    CPU: AMD EPYC 7763 (64 cores, 128 threads)
    RAM: 256GB DDR4-3200 ECC
    Storage:
      - OS: 2x 480GB SSD RAID 1
      - Data: 4x 2TB NVMe RAID 10
    Network: Dual 10GbE SFP+
    Power: Redundant PSU
    UPS: 3000VA

  GPU Server:
    Model: Custom Build
    CPU: Intel Xeon W-2495X (24 cores)
    GPU: 2x NVIDIA RTX 4090 24GB
    RAM: 128GB DDR5-5600
    Storage:
      - OS: 1TB NVMe
      - Models: 4TB NVMe
    Cooling: Liquid cooled

  Edge Servers (2x):
    Location: Bay Area IX, LA IX
    Model: HPE ProLiant DL360
    CPU: Intel Xeon Gold 6348 (28 cores)
    RAM: 64GB DDR4
    Storage: 2x 960GB SSD
    Network: 10GbE
    Purpose: Game matchmaking, WebRTC relay

Software Stack:
  Hypervisor:
    - Proxmox VE 8.1
    - ZFS for storage
    - Backup to S3

  Container Platform:
    - K3s (Lightweight K8s)
    - Rancher for management
    - Longhorn for persistent storage

  Blockchain:
    - Polygon full node
    - Heimdall + Bor
    - 4TB minimum storage
    - Monitoring: Prometheus

  ML/AI Stack:
    - CUDA 12.2
    - TensorRT
    - Triton Inference Server
    - Model registry on S3

  Networking:
    - WireGuard VPN to AWS
    - pfSense firewall
    - HAProxy load balancer
```

**설계 근거:**

**Polygon 풀노드 온프레미스 운영 이유:**

- **외부 RPC 서비스 제한:** Alchemy는 월 300만 요청 제한, Infura는 일 100K 요청 제한이 있어 P2E 게임에는 부족합니다
- **비용 효과:** 외부 RPC 엔터프라이즈 요금 월 $1,000+ vs 서버 운영비 월 $200
- **레이턴시:** 자체 노드는 <10ms, 외부 RPC는 50-200ms
- **신뢰성:** 서비스 중단 시 게임 전체가 마비되는 것을 방지

**GPU 서버 자체 운영 근거:**

- **클라우드 GPU 비용:** AWS g5.xlarge는 월 $800+, 2x RTX 4090 서버는 초기 투자 $8,000으로 10개월이면 회수
- **추론 성능:** RTX 4090이 T4/V100보다 3-5배 빠른 추론 속도
- **용도:** Anti-cheat ML 모델, 실시간 이미지 생성, 패턴 분석

**K3s 선택 이유:**

- **경량화:** 전체 K8s 대비 메모리 사용량 50% 감소
- **단순성:** 단일 바이너리로 설치, 3대 서버에 최적화
- **호환성:** 표준 K8s API 100% 호환으로 향후 마이그레이션 용이

**엣지 서버 IX 배치 이유:**

- **초저지연:** 게임 매치메이킹 <5ms 응답 필요
- **WebRTC:** TURN/STUN 서버는 물리적 근접성이 품질 결정
- **비용:** IX 코로케이션 월 $300 vs AWS 동일 성능 월 $1,000+

**Proxmox vs VMware:**

- **라이선스 비용:** VMware는 연 $5,000+, Proxmox는 무료 (서브스크립션 선택적)
- **ZFS 통합:** 내장 스냅샷, 압축, 중복 제거로 스토리지 효율 40% 개선
- **API 자동화:** REST API로 Terraform 통합 가능

**고려했던 대안:**

- **베어메탈 vs 가상화:** 유연성과 백업 용이성을 위해 가상화 선택
- **Docker Swarm vs K3s:** Kubernetes 에코시스템의 풍부함으로 K3s 선택
- **자체 데이터센터 vs 코로케이션:** 초기 투자 부담으로 코로케이션 선택 (월 $500)

### 1.4 Hybrid Cloud Integration

```yaml
Connectivity:
  Primary: AWS Site-to-Site VPN
    - 2 tunnels for HA
    - BGP routing
    - 1.25 Gbps per tunnel

  Backup: Direct Connect (Future)
    - 10 Gbps dedicated
    - Virtual interfaces (VIFs)
    - Lower latency

Data Sync:
  Database Replication:
    - PostgreSQL logical replication
    - One-way: Cloud → On-premise
    - For analytics and backup

  Object Storage:
    - AWS DataSync
    - Scheduled transfers
    - Incremental sync

  Blockchain Events:
    - Webhook from on-premise
    - SQS queue in AWS
    - Lambda processor

Service Discovery:
  - AWS Cloud Map
  - Consul (on-premise)
  - Mesh gateway for communication
```

## 2. Container Orchestration

### 2.1 ECS Configuration (MVP Phase)

```yaml
Task Definitions:
  location-service:
    family: ore-location-service
    networkMode: awsvpc
    requiresCompatibilities: [FARGATE]
    cpu: "1024"
    memory: "2048"

    containerDefinitions:
      - name: location-service
        image: ${ECR_REGISTRY}/location-service:${VERSION}

        portMappings:
          - containerPort: 8080
            protocol: tcp

        environment:
          - name: RUST_LOG
            value: info
          - name: SERVICE_NAME
            value: location-service

        secrets:
          - name: DATABASE_URL
            valueFrom: arn:aws:secretsmanager:${REGION}:${ACCOUNT}:secret:ore/database

        logConfiguration:
          logDriver: awslogs
          options:
            awslogs-group: /ecs/location-service
            awslogs-region: ${REGION}
            awslogs-stream-prefix: ecs

        healthCheck:
          command:
            ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
          interval: 30
          timeout: 5
          retries: 3
          startPeriod: 60

Service Discovery:
  Namespace: ore.local

  Services:
    - location.ore.local
    - game.ore.local
    - auth.ore.local
    - realtime.ore.local

  Health Check: Route 53

Load Balancing:
  Application Load Balancer:
    Scheme: internet-facing
    Type: application

    Listeners:
      HTTP (80):
        - Redirect to HTTPS

      HTTPS (443):
        - Certificate: ACM
        - Security Policy: TLS 1.3

    Target Groups:
      - /api/location/* → location-service
      - /api/game/* → game-service
      - /api/auth/* → auth-service
      - /ws/* → realtime-engine (sticky)
```

**설계 근거:**

**awsvpc 네트워크 모드 선택:**
각 태스크가 고유한 ENI를 갖게 되어 세밀한 보안 그룹 제어가 가능하고, 서비스 메시 통합이 용이합니다. 성능상 오버헤드도 bridge 모드 대비 최소화됩니다.

**Health Check 전략:**

- **30초 간격:** 너무 잦으면 부하 증가, 너무 드물면 장애 감지 지연
- **3회 재시도:** 일시적 네트워크 이슈로 인한 false positive 방지
- **60초 시작 대기:** Rust 서비스의 초기화 시간 고려

**Service Discovery 구현:**

- **Cloud Map 사용:** 내부 서비스 간 통신에서 하드코딩된 엔드포인트 제거
- **Route 53 통합:** DNS 기반으로 어플리케이션 수정 없이 서비스 검색
- **비용:** 월 $10 미만으로 서비스 메시 대비 90% 저렴

**ALB vs NLB 선택:**

- **ALB 선택 이유:** L7 라우팅, WebSocket 지원, 상세한 메트릭
- **Sticky Session:** WebSocket 연결 유지를 위해 필수
- **TLS 1.3:** 최신 보안 표준 적용 (2025년 기본)

### 2.2 Migration to EKS (Phase 3)

```yaml
EKS Cluster:
  Version: 1.30

  Node Groups:
    System:
      Instance Types: [t3.medium]
      Min: 2
      Max: 4
      Desired: 2
      Labels:
        workload: system

    Application:
      Instance Types: [c6i.xlarge, c6i.2xlarge]
      Min: 3
      Max: 20
      Desired: 5
      Taints:
        - key: workload
          value: application
          effect: NoSchedule

    GPU (Future):
      Instance Types: [g5.xlarge]
      Min: 0
      Max: 2
      Desired: 0
      Taints:
        - key: nvidia.com/gpu
          effect: NoSchedule

Add-ons:
  - vpc-cni
  - kube-proxy
  - coredns
  - ebs-csi-driver
  - efs-csi-driver

Ingress:
  AWS Load Balancer Controller:
    - ALB for HTTP/HTTPS
    - NLB for TCP/UDP

Service Mesh (Optional):
  AWS App Mesh:
    - Traffic management
    - Observability
    - Security
```

**설계 근거:**

**EKS 전환 시점 (Phase 3):**

- **10K 사용자 도달 시:** ECS의 한계점 도달 (세밀한 스케줄링, 복잡한 배포 전략)
- **마이크로서비스 15개 이상:** Kubernetes의 오케스트레이션 능력 필요
- **비용 임계점:** 월 $5,000 이상 지출 시 EKS 관리 비용($72/월) 정당화

**Node Group 분리 전략:**

- **System 노드:** 작고 안정적인 인스턴스로 컨트롤 플레인 컴포넌트 운영
- **Application 노드:** CPU 최적화 인스턴스로 실제 워크로드 처리
- **Taint/Toleration:** 워크로드 격리로 리소스 경합 방지

**Instance Type 선택:**

- **t3.medium (System):** 버스트 가능한 저비용 옵션
- **c6i 시리즈 (App):** Graviton 대비 x86 호환성 우선
- **g5 (GPU):** 향후 ML 워크로드를 위한 준비

**App Mesh 도입 시기:**

- **초기 (선택적):** 복잡성 대비 이득이 적음
- **100K 사용자 시점:** 서비스 간 통신 복잡도 증가로 필요성 대두
- **대안:** Istio는 학습 곡선이 가파르고, Linkerd는 기능 제한적

**고려했던 대안:**

- **Self-managed K8s vs EKS:** 운영 부담과 보안 업데이트 관리를 AWS에 위임
- **Karpenter vs Cluster Autoscaler:** 초기에는 CA로 충분, 복잡도 증가 시 Karpenter 도입

## 3. Infrastructure as Code

### 3.1 Terraform Structure

```hcl
# Directory Structure
infrastructure/
├── terraform/
│   ├── environments/
│   │   ├── dev/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── terraform.tfvars
│   │   └── prod/
│   ├── modules/
│   │   ├── networking/
│   │   │   ├── vpc.tf
│   │   │   ├── subnets.tf
│   │   │   ├── security_groups.tf
│   │   │   ├── nat_gateway.tf
│   │   │   └── variables.tf
│   │   ├── compute/
│   │   │   ├── ecs_cluster.tf
│   │   │   ├── ecs_services.tf
│   │   │   ├── lambda.tf
│   │   │   └── autoscaling.tf
│   │   ├── database/
│   │   │   ├── aurora.tf
│   │   │   ├── elasticache.tf
│   │   │   ├── msk.tf
│   │   │   └── backups.tf
│   │   ├── storage/
│   │   │   ├── s3.tf
│   │   │   ├── efs.tf
│   │   │   └── lifecycle.tf
│   │   └── monitoring/
│   │       ├── cloudwatch.tf
│   │       ├── alarms.tf
│   │       └── dashboards.tf
│   └── global/
│       ├── iam/
│       ├── route53/
│       └── cloudfront/

# Example Module: VPC
module "vpc" {
  source = "../../modules/networking"

  environment = var.environment
  cidr_block  = "10.0.0.0/16"

  availability_zones = [
    "us-west-1a",
    "us-west-1b",
    "us-west-1c"
  ]

  public_subnet_cidrs = [
    "10.0.1.0/24",
    "10.0.2.0/24",
    "10.0.3.0/24"
  ]

  private_subnet_cidrs = [
    "10.0.11.0/24",
    "10.0.12.0/24",
    "10.0.13.0/24"
  ]

  database_subnet_cidrs = [
    "10.0.21.0/24",
    "10.0.22.0/24",
    "10.0.23.0/24"
  ]

  enable_nat_gateway = true
  single_nat_gateway = false  # HA를 위해 각 AZ에 NAT
  enable_dns_hostnames = true
  enable_dns_support = true

  tags = {
    Project     = "ORE"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

### 3.2 State Management

```yaml
Backend Configuration:
  S3 Backend:
    Bucket: ore-terraform-state-${ACCOUNT_ID}
    Key: ${ENVIRONMENT}/terraform.tfstate
    Region: us-west-1
    Encrypt: true

  State Locking:
    DynamoDB Table: ore-terraform-locks
    Hash Key: LockID

  Workspace Strategy:
    - workspace per environment
    - Separate state files
    - Isolated blast radius

State Security:
  - S3 bucket versioning
  - S3 bucket encryption (SSE-KMS)
  - Bucket policy (restrict access)
  - MFA delete protection
  - Access logging
```

### 3.3 CI/CD Pipeline

```yaml
GitHub Actions Workflow:
  name: Infrastructure Deployment

  triggers:
    - Pull request (plan only)
    - Push to main (apply)

  jobs:
    terraform-check:
      steps:
        - Checkout code
        - Setup Terraform
        - Format check
        - Validate
        - Security scan (tfsec)
        - Cost estimation (Infracost)

    terraform-plan:
      steps:
        - Configure AWS credentials
        - Initialize Terraform
        - Select workspace
        - Generate plan
        - Post plan to PR

    terraform-apply:
      if: github.ref == 'refs/heads/main'
      steps:
        - Configure AWS credentials
        - Initialize Terraform
        - Select workspace
        - Apply with auto-approve
        - Notify Slack

    post-deployment:
      steps:
        - Smoke tests
        - Update documentation
        - Tag release

Rollback Strategy:
  - State file versioning
  - Previous version in S3
  - Terraform state pull/push
  - Blue-green for critical changes
```

## 4. Security & Compliance

### 4.1 Security Architecture

```yaml
Network Security:
  Perimeter Defense:
    - AWS WAF (Web ACL)
    - AWS Shield Standard
    - CloudFront (DDoS protection)

  Network Isolation:
    - Private subnets for compute
    - Database subnet isolation
    - Security group chaining
    - NACLs for additional control

  Zero Trust Model:
    - No implicit trust
    - Verify every request
    - Least privilege access
    - Micro-segmentation

Identity & Access:
  IAM Roles:
    - Service-specific roles
    - Temporary credentials
    - No long-lived keys

  Secrets Management:
    - AWS Secrets Manager
    - Automatic rotation
    - Encryption at rest
    - Audit logging

  MFA Enforcement:
    - Console access
    - Programmatic access (sensitive)
    - Git commits (GPG)

Data Protection:
  Encryption at Rest:
    - RDS: AWS KMS (CMK)
    - S3: SSE-S3 or SSE-KMS
    - EBS: Volume encryption
    - ElastiCache: Redis AUTH + TLS

  Encryption in Transit:
    - TLS 1.3 everywhere
    - Certificate management (ACM)
    - mTLS for service-to-service

  Data Loss Prevention:
    - S3 Object Lock
    - Backup encryption
    - Cross-region replication
```

**설계 근거:**

**Zero Trust 아키텍처 채택:**
전통적인 성곽형(perimeter) 보안의 한계를 극복하고, 내부 위협에도 대응 가능한 모델입니다. 특히 P2E 게임은 금전적 이득이 직접적이어서 공격 대상이 되기 쉽습니다.

**AWS WAF 규칙 전략:**

- **Rate Limiting:** 사용자당 분당 100 요청 제한 (봇 방지)
- **Geo Blocking:** 초기에는 한국/일본/미국만 허용 (공격 표면 감소)
- **SQL Injection/XSS:** Core Rule Set 적용
- **비용:** 월 $20 + 요청당 $0.0006 (Shield Standard는 무료)

**Secrets Manager vs Parameter Store:**

- **Secrets Manager 선택:** 자동 로테이션, 세밀한 권한 제어, 감사 로깅
- **비용 차이:** 월 $10 추가이지만 보안 사고 한 번이면 수백만원 손실
- **로테이션 주기:** 데이터베이스 30일, API 키 90일

**TLS 1.3 전면 도입:**

- **2025년 표준:** TLS 1.2는 더 이상 권장되지 않음
- **성능 개선:** 핸드셰이크 1-RTT 감소 (모바일에서 중요)
- **보안 강화:** 완벽한 순방향 비밀성(PFS) 기본 제공

**KMS 키 관리 전략:**

- **서비스별 CMK:** 블라스트 반경(blast radius) 최소화
- **자동 로테이션:** 연 1회 (규제 준수)
- **키 정책:** 특정 서비스만 특정 키 사용 가능

**고려했던 대안:**

- **HashiCorp Vault vs AWS Secrets Manager:** AWS 네이티브 통합의 편의성으로 SM 선택
- **CloudFlare vs CloudFront:** AWS 통합과 Shield 연동으로 CloudFront 선택
- **VPN vs PrivateLink:** 확장성과 관리 용이성으로 PrivateLink 선택

### 4.2 Compliance & Auditing

```yaml
Compliance Standards:
  - PCI DSS (payment processing)
  - GDPR (EU users)
  - KISA (Korean regulations)
  - SOC 2 Type II (future)

Auditing:
  CloudTrail:
    - All API calls logged
    - Multi-region trail
    - Log file validation
    - S3 storage with MFA delete

  VPC Flow Logs:
    - All traffic logged
    - Stored in S3
    - Analyzed with Athena

  Application Logs:
    - Centralized in CloudWatch
    - Structured logging (JSON)
    - Correlation IDs
    - PII masking

Vulnerability Management:
  - AWS Inspector (EC2, ECR)
  - ECR image scanning
  - Dependency scanning (Snyk)
  - Penetration testing (quarterly)
```

**설계 근거:**

**PCI DSS 준비:**
게임 내 결제 처리를 위해 필수입니다. 초기에는 Level 4 (연간 2만 건 이하)로 시작하여 Self-Assessment Questionnaire로 충분합니다.

- **네트워크 분리:** 카드 데이터는 별도 서브넷
- **암호화:** 저장/전송 시 모두 암호화
- **접근 제어:** MFA 필수, 최소 권한 원칙

**GDPR 컴플라이언스:**
EU 사용자 확보를 위한 필수 요건입니다.

- **데이터 최소화:** 필요한 정보만 수집
- **삭제권:** 30일 내 완전 삭제 프로세스
- **이동권:** 데이터 export API 제공
- **암호화:** Pseudonymization 적용

**CloudTrail 설정:**

- **Multi-region:** 모든 리전의 활동 추적 (우회 공격 방지)
- **Log Validation:** 변조 방지를 위한 다이제스트 파일
- **MFA Delete:** 로그 삭제 시 MFA 필수
- **90일 보관:** 규제 요구사항 충족

**VPC Flow Logs 분석:**

- **Athena 쿼리:** SQL로 손쉬운 분석
- **이상 탐지:** 비정상 트래픽 패턴 식별
- **비용 최적화:** S3 저장으로 CloudWatch 대비 80% 저렴
- **압축:** Parquet 포맷으로 90% 용량 절감

**취약점 스캔 주기:**

- **일일:** ECR 이미지 스캔 (CI/CD 파이프라인)
- **주간:** 의존성 업데이트 체크
- **월간:** AWS Inspector 전체 스캔
- **분기:** 외부 펜테스트 (연 $20,000 예산)

## 5. Monitoring & Observability

### 5.1 Metrics & Monitoring

```yaml
CloudWatch Metrics:
  System Metrics:
    - CPU, Memory, Disk, Network
    - Collected every 60 seconds
    - Retained for 15 months

  Custom Metrics:
    - Application performance
    - Business KPIs
    - User behavior
    - Cost metrics

Prometheus + Grafana:
  Deployment:
    - ECS service or Lambda
    - Persistent storage (EBS)
    - High availability setup

  Metrics Collection:
    - Service discovery
    - Auto-scraping
    - Push gateway for batch

  Dashboards:
    - System overview
    - Service health
    - Business metrics
    - Cost tracking

Application Performance:
  AWS X-Ray:
    - Distributed tracing
    - Service map
    - Latency analysis
    - Error debugging

  OpenTelemetry:
    - Vendor-neutral
    - Auto-instrumentation
    - Custom spans
    - Baggage propagation
```

### 5.2 Logging Architecture

```yaml
Log Collection:
  CloudWatch Logs:
    - ECS container logs
    - Lambda function logs
    - VPC Flow Logs
    - CloudTrail logs

  Log Aggregation:
    - Kinesis Data Firehose
    - S3 for long-term storage
    - OpenSearch for analysis

Log Processing:
  Stream Processing:
    - Kinesis Data Streams
    - Lambda processors
    - Real-time alerts

  Batch Processing:
    - S3 → Glue → Athena
    - Daily aggregations
    - Cost analysis

Log Retention:
  Hot Storage (CloudWatch):
    - 7 days: Debug logs
    - 30 days: Application logs
    - 90 days: Audit logs

  Cold Storage (S3):
    - Standard: 30 days
    - Infrequent Access: 90 days
    - Glacier: 1 year+
```

### 5.3 Alerting & Incident Response

```yaml
Alert Configuration:
  Priority Levels:
    P1 (Critical):
      - Service down
      - Data loss risk
      - Security breach
      - Response: Immediate

    P2 (High):
      - Performance degradation
      - High error rate
      - Capacity issues
      - Response: 30 minutes

    P3 (Medium):
      - Warning thresholds
      - Non-critical errors
      - Response: 2 hours

    P4 (Low):
      - Info only
      - Trends
      - Response: Next business day

Alert Channels:
  PagerDuty:
    - P1/P2 alerts
    - On-call rotation
    - Escalation policies

  Slack:
    - All alerts
    -  #alerts-critical
    -  #alerts-warning

  Email:
    - Daily summaries
    - Weekly reports

Runbooks:
  - Automated remediation
  - Step-by-step guides
  - Rollback procedures
  - Contact information
```

## 6. Disaster Recovery

### 6.1 Backup Strategy

```yaml
Database Backups:
  Aurora:
    - Continuous backup to S3
    - Point-in-time recovery (35 days)
    - Automated snapshots (daily)
    - Manual snapshots (weekly)
    - Cross-region replication

  ElastiCache:
    - Daily snapshots
    - Retention: 7 days
    - Export to S3

  Application Data:
    - S3 versioning
    - Cross-region replication
    - Lifecycle policies

Backup Testing:
  - Monthly restore tests
  - Documented procedures
  - Time to restore metrics
  - Data integrity checks
```

### 6.2 Disaster Recovery Plan

```yaml
RTO/RPO Targets:
  Critical Services:
    - RTO: 1 hour
    - RPO: 15 minutes

  Non-Critical:
    - RTO: 4 hours
    - RPO: 1 hour

DR Scenarios:
  AZ Failure:
    - Automatic failover
    - No manual intervention
    - RTO: < 5 minutes

  Region Failure:
    - Pilot light in N. California/Oregon
    - Database replication
    - DNS failover
    - RTO: 1 hour

  Data Corruption:
    - Point-in-time recovery
    - S3 versioning
    - Backup restoration
    - RPO: 15 minutes

  Account Compromise:
    - Cross-account backups
    - MFA everywhere
    - Break-glass procedures
    - RTO: 2 hours
```

## 7. Cost Optimization

### 7.1 Cost Management Strategy

```yaml
Compute Optimization:
  Spot Instances:
    - 70% of workload on Spot
    - Fargate Spot for services
    - Spot Fleet for batch
    - Savings: 60-90%

  Right-Sizing:
    - AWS Compute Optimizer
    - Weekly reviews
    - Automated scaling
    - Idle resource detection

  Serverless First:
    - Lambda for event processing
    - Fargate for containers
    - Aurora Serverless for DB
    - Pay only for usage

Storage Optimization:
  S3 Lifecycle:
    - Intelligent-Tiering
    - Infrequent Access (30 days)
    - Glacier (90 days)
    - Expiration policies

  EBS Optimization:
    - GP3 vs GP2 analysis
    - Snapshot management
    - Unused volume detection

Data Transfer:
  - VPC Endpoints (S3, ECR)
  - CloudFront caching
  - Same-AZ transfers
  - NAT Instance vs NAT Gateway

Reserved Capacity:
  - Reserved Instances (RDS)
  - Savings Plans (Compute)
  - Capacity Reservations
  - 1-year vs 3-year analysis
```

**설계 근거:**

**70% Spot 전략의 실현 가능성:**

- **Fargate Spot:** 2분 경고로 graceful shutdown 가능
- **다중 AZ 분산:** 특정 AZ Spot 부족 시 다른 AZ로 자동 전환
- **Capacity Rebalancing:** AWS가 자동으로 새 인스턴스 프로비저닝
- **실제 절감액:** 월 $500 → $150 (Location/Game 서비스)

**Serverless의 숨겨진 비용 관리:**

- **Lambda 콜드 스타트:** Reserved Concurrency로 예측 가능한 비용
- **Aurora Serverless 스파이크:** Max ACU 제한으로 비용 폭탄 방지
- **API Gateway:** 캐싱 활성화로 백엔드 호출 50% 감소
- **Break-even point:** 40% 이상 idle time이면 Serverless가 유리

**S3 Intelligent-Tiering 세부 전략:**

- **자동 티어링:** 30일 미접근 → IA, 90일 → Archive
- **메타데이터 제외:** 작은 객체(<128KB)는 Standard 유지
- **실제 절감:** 게임 에셋의 80%가 IA로 이동, 월 $200 절약

**데이터 전송 비용 함정 회피:**

- **NAT Gateway:** 시간당 $0.045 + GB당 $0.045 (월 $100+)
- **VPC Endpoints:** S3/ECR 전송 비용 0 (월 $50 절약)
- **Same-AZ 전송:** RDS-ECS 같은 AZ 배치로 cross-AZ 비용 제거
- **CloudFront:** 오리진 히트율 90% 달성 시 전송 비용 80% 절감

**Reserved Capacity 의사결정 프레임워크:**

```
if (확실한 베이스라인 워크로드) {
  if (현금 여유 있음) {
    3년 All Upfront RI (72% 할인)
  } else {
    1년 No Upfront Savings Plan (30% 할인)
  }
} else {
  Spot + On-Demand 조합 유지
}
```

**고려했던 대안:**

- **자체 비용 도구 vs AWS Cost Explorer:** 초기에는 CE로 충분, $100K/월 넘으면 자체 도구 검토
- **Multi-cloud arbitrage:** 복잡성 대비 이득 적음, AWS 단일 클라우드 유지

### 7.2 Cost Monitoring

```yaml
AWS Cost Explorer:
  - Daily cost tracking
  - Service breakdown
  - Tag-based allocation
  - Anomaly detection

Cost Allocation:
  Tags:
    - Project: ORE
    - Environment: dev/prod
    - Service: location/game/auth
    - Team: backend/frontend
    - Cost-Center: engineering

  Reports:
    - Daily email summary
    - Weekly Slack report
    - Monthly executive dashboard

Budgets & Alerts:
  - Monthly budget limits
  - 80% threshold alerts
  - Forecast-based alerts
  - Action on overspend

FinOps Practices:
  - Weekly cost reviews
  - Quarterly optimization
  - Showback/Chargeback
  - Cost per user metrics
```

**설계 근거:**

**태그 전략의 중요성:**

- **필수 태그 5개:** 이것만으로도 90% 비용 추적 가능
- **자동화:** Terraform으로 태그 누락 방지
- **Enforcement:** Tag Policy로 태그 없는 리소스 생성 차단
- **ROI:** 정확한 비용 귀속으로 월 10-20% 절감 기회 발견

**알림 임계값 설정:**

- **80% 경고:** 대응 시간 확보
- **100% 차단:** 예산 초과 시 개발 환경 자동 중지
- **일일 이상 탐지:** 전일 대비 50% 증가 시 즉시 알림
- **예측 알림:** 월말 예상 초과 시 15일 전 경고

**FinOps 성숙도 모델:**

```
Level 1 (현재): 비용 가시성
  - 태그 기반 추적
  - 기본 리포트

Level 2 (6개월): 비용 최적화
  - 자동화된 절감
  - Reserved Instance 관리

Level 3 (1년): 비용 거버넌스
  - 팀별 예산 책임
  - 자동 시정 조치
```

**비용 지표 벤치마크:**

- **Cost per MAU:** $0.50-0.80 (업계 평균 $1.20)
- **인프라 비용/매출:** 15-20% (업계 평균 25%)
- **Utilization Rate:** 65%+ (업계 평균 45%)

**월간 최적화 체크리스트:**

1. Unattached EBS/EIP 삭제 (월 $50)
2. 오래된 스냅샷 정리 (월 $30)
3. 미사용 로드밸런서 제거 (월 $20)
4. S3 불완전 업로드 정리 (월 $10)
5. CloudWatch 로그 보존 기간 조정 (월 $40)

## 8. Scaling Strategy

### 8.1 Horizontal Scaling

```yaml
Auto Scaling Policies:
  Target Tracking:
    - CPU Utilization: 70%
    - Memory Utilization: 80%
    - Request Count: Custom
    - Queue Depth: SQS

  Step Scaling:
    - Aggressive scale-out
    - Conservative scale-in
    - Cooldown periods

  Predictive Scaling:
    - ML-based predictions
    - Daily/weekly patterns
    - Event-based scaling

Service Scaling:
  ECS Services:
    Min/Max Tasks:
      - Location: 2/10
      - Game: 2/10
      - Realtime: 3/15
      - Auth: 1/5

    Scaling Metrics:
      - ECS Service Average CPU
      - ECS Service Average Memory
      - ALB Target Response Time
      - Custom CloudWatch Metrics

  Database Scaling:
    Aurora Serverless v2:
      - Auto scaling ACUs
      - Read replica auto-scaling
      - Storage auto-scaling

    ElastiCache:
      - Cluster mode resharding
      - Replica auto-scaling
```

### 8.2 Vertical Scaling

```yaml
Instance Type Optimization:
  Compute Optimized:
    - C6i series for CPU-intensive
    - C7g (Graviton3) for cost savings

  Memory Optimized:
    - R6i series for cache
    - X2idn for in-memory DB

  GPU Instances:
    - G5 for ML inference
    - P4d for training (future)

Burst Capacity:
  - T3/T4g instances for variable load
  - Unlimited mode for consistent performance
  - Credit monitoring
```

## 9. Development Workflow

### 9.1 Environment Strategy

```yaml
Environments:
  Development:
    - Smallest instance sizes
    - Single AZ deployment
    - Reduced redundancy
    - Auto-shutdown at night
    - Cost: ~$200/month
    - Branch: dev

  Production:
    - Full HA setup
    - Multi-AZ everything
    - Auto-scaling enabled
    - Complete monitoring
    - Cost: $800-5000/month
    - Branch: main

Environment Promotion:
  - Git flow branching (dev → main)
  - Automated testing
  - Manual approval gates
  - Blue-green deployments
  - Instant rollback
```

### 9.2 IaC Development Workflow

```yaml
Local Development:
  Tools:
    - Terraform 1.6+
    - AWS CLI v2
    - LocalStack (optional)
    - pre-commit hooks

  Workflow: 1. Feature branch
    2. Local validation
    3. terraform plan
    4. PR with plan output
    5. Review & approve
    6. Merge & auto-apply

Code Quality:
  Pre-commit Hooks:
    - terraform fmt
    - terraform validate
    - tflint
    - tfsec
    - checkov

  PR Checks:
    - Plan preview
    - Cost estimation
    - Security scanning
    - Compliance check

Documentation:
  - README per module
  - Architecture diagrams
  - Runbooks
  - Disaster recovery procedures
```

## 10. Migration Roadmap

### 10.1 Phase 1: MVP (Months 1-3)

```yaml
Infrastructure:
  - Single region (N. California)
  - ECS Fargate only
  - Managed services
  - Basic monitoring

Services:
  - 4 core services
  - ALB routing
  - CloudFront CDN
  - Aurora Serverless

Cost Target: $500-800/month
Users: 1,000
```

### 10.2 Phase 2: Growth (Months 4-6)

```yaml
Infrastructure:
  - Add spot instances
  - Enhanced monitoring
  - Backup region setup
  - On-premise integration

Services:
  - 8 microservices
  - Service mesh
  - Advanced caching
  - ML inference

Cost Target: $3,000-5,000/month
Users: 10,000
```

### 10.3 Phase 3: Scale (Months 7-12)

```yaml
Infrastructure:
  - Multi-region active
  - EKS migration
  - Edge locations
  - Full DR setup

Services:
  - 15+ microservices
  - GraphQL gateway
  - Real-time analytics
  - Blockchain integration

Cost Target: $15,000-25,000/month
Users: 100,000
```

### 10.4 Phase 4: Enterprise (Year 2)

```yaml
Infrastructure:
  - Global presence
  - Multi-cloud ready
  - Edge computing
  - Custom hardware

Services:
  - 30+ microservices
  - Event sourcing
  - CQRS pattern
  - AI/ML pipeline

Cost Target: $50,000+/month
Users: 1,000,000
```

## 11. Operational Excellence

### 11.1 Automation

```yaml
Infrastructure Automation:
  - Everything as code
  - GitOps workflow
  - Automated testing
  - Self-healing systems

Operational Automation:
  - Auto-scaling
  - Auto-remediation
  - Automated backups
  - Compliance scanning

Development Automation:
  - CI/CD pipelines
  - Automated deployments
  - Feature flags
  - A/B testing
```

### 11.2 Documentation

```yaml
Required Documentation:
  - Architecture diagrams
  - API documentation
  - Runbooks
  - Disaster recovery plans
  - Security procedures
  - Cost analysis

Documentation Tools:
  - Confluence/Notion
  - Draw.io/Lucidchart
  - Swagger/OpenAPI
  - README files

Review Cycle:
  - Weekly updates
  - Monthly reviews
  - Quarterly audits
```

## Conclusion

이 인프라 명세서는 Project ORE의 성장 단계별 요구사항을 충족하도록 설계되었습니다:

### 핵심 설계 결정 요약

**왜 하이브리드 클라우드인가?**

- **AWS 85%:** 관리 부담 최소화, 빠른 확장, 글로벌 인프라 활용
- **온프레미스 15%:** 블록체인 노드 안정성, GPU 비용 절감, 초저지연 요구사항
- **점진적 전환:** 단순하게 시작하여 필요에 따라 복잡도 증가

**왜 ECS Fargate로 시작하는가?**

- **K8s 복잡성 회피:** 1인 개발자가 비즈니스 로직에 집중
- **서버리스 이점:** 패치, 스케일링, 모니터링 자동화
- **명확한 마이그레이션 경로:** 10K 사용자 시점에 EKS로 전환

**왜 Serverless 우선 전략인가?**

- **예측 불가능한 MVP 트래픽:** 0에서 1000까지 자동 대응
- **비용 효율:** 사용한 만큼만 지불 (40% 이상 idle 시 유리)
- **운영 부담 제로:** 패치, 백업, 스케일링 모두 AWS 관리

### 비용 최적화의 핵심

```yaml
월간 비용 목표 달성 전략:
  MVP (1K users): $500-800
    - Fargate Spot 70%
    - Serverless 서비스
    - 단일 리전

  성장기 (10K users): $3,000-5,000
    - Reserved Instances
    - 온프레미스 활용
    - 비용 자동화

  확장기 (100K users): $15,000-25,000
    - 하이브리드 최적화
    - FinOps 성숙도 3
    - 멀티 리전 준비
```

### 기술 부채 관리

**의도적 기술 부채:**

- NAT Gateway (vs NAT Instance): 안정성 우선
- ALB (vs Kong): 관리 편의성 우선
- Single Region: 복잡성 최소화

**명확한 해결 시점:**

- Phase 2: NAT Instance 검토
- Phase 3: Kong/Envoy 도입
- Phase 4: Multi-region 구현

### 성공 지표

**기술적 성공:**

- ✅ 99.9% 가동률 (월 43분 다운타임 허용)
- ✅ P95 응답시간 < 100ms
- ✅ 비용 대비 10x 사용자 처리

**비즈니스 성공:**

- ✅ 12주 내 MVP 출시
- ✅ Cost per MAU < $0.80
- ✅ 인프라 비용/매출 < 20%

### AI-Native 인프라 구현

이 명세서는 AI 도구를 활용한 인프라 구현을 전제로 작성되었습니다:

- **Terraform 모듈:** Claude Code로 2시간 내 생성
- **CI/CD 파이프라인:** GitHub Copilot으로 1일 구현
- **모니터링 대시보드:** AI 어시스턴트로 30분 설정
- **총 구현 시간:** 2주 (전통적 방식 대비 80% 단축)

### 최종 메시지

> "**Start Simple, Scale Smart, Automate Everything**"
>
> MVP에서 시작하여 100만 사용자까지, 재설계 없이 점진적으로 확장 가능한 인프라입니다.

---

_Version: 1.1_  
_Last Updated: 2024-12-20_  
_Infrastructure Decision Records (IDR) available in /docs/infrastructure/_
_Design Rationale added for transparency and education_
