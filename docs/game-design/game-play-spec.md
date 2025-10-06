# ORE Game - Gameplay Specification

_순수 게임 메커니즘과 밸런스 정의서_

## 1. Core Game Loop

### 1.1 Primary Loop (60 seconds)

| Phase    | Duration | Action                   | Output            |
| -------- | -------- | ------------------------ | ----------------- |
| Move     | 10s      | Player physical movement | Position update   |
| Discover | 5s       | AR mode activation       | Ore visibility    |
| Mine     | 5s       | Touch interaction        | Mining animation  |
| Collect  | 5s       | Collection validation    | Points/XP gain    |
| Repeat   | 35s      | Next target selection    | Loop continuation |

**설계 의도:**
60초 루프는 모바일 게임의 '한 입 크기(bite-sized)' 플레이를 극대화합니다. 1분 안에 완전한 보상 사이클을 경험함으로써 "한 개만 더" 심리를 자극하고, 짧은 휴식 시간에도 의미 있는 진전을 느낄 수 있게 설계했습니다. 각 단계의 시간 배분은 실제 테스트를 통해 지루함과 조급함 사이의 최적점을 찾은 결과입니다.

### 1.2 Session Structure

| Session Type | Duration  | Frequency | Expected Output      |
| ------------ | --------- | --------- | -------------------- |
| Morning      | 10-15 min | 1x daily  | 20-30 ores           |
| Lunch        | 5-10 min  | 1x daily  | 10-20 ores           |
| Evening      | 15-20 min | 1x daily  | 30-40 ores           |
| Night        | 5 min     | Optional  | Inventory management |

**설계 의도:**
일반적인 직장인의 일과를 고려한 세션 구조입니다. 출퇴근길과 점심시간이라는 '죽은 시간'을 게임으로 활성화하되, 과도한 플레이를 요구하지 않습니다. 총 일일 플레이 시간을 30-45분으로 제한함으로써 번아웃을 방지하고 장기적인 리텐션을 추구합니다.

## 2. Fracture → Vein → Core System

### 2.1 Core Mechanic Overview

```yaml
Fracture (균열):
  정의: 현실에 생긴 디지털 균열, 광석이 존재하는 영역
  크기: 150m x 150m (MVP) → Dynamic sizing (Post-MVP)
  가시성: Map.unity에서 균열 이펙트/하이라이트로 표시
  목적: 플레이어에게 대략적인 채굴 위치 제공
  생성: 서버 알고리즘 기반 (S2 Geometry Level 15 기반)

  Dynamic Sizing (Post-MVP):
    - Urban/Dense: 100m x 100m
    - Suburban: 150m x 150m
    - Park/Open: 200m x 200m
    - Rarity-based: Legendary Fractures can be 300m+

Vein (광맥):
  정의: Fracture 내부의 정확한 광석 위치
  좌표: 정확한 GPS 좌표 (위도/경도)
  가시성: Fracture 진입 전까지 숨겨짐, AR 모드에서만 발견 가능
  발견 범위: 10m (일반 유저) / 3m (Genesis 1000)
  목적: AR 탐색 게임플레이 제공

Core (광석 코어):
  정의: 채굴 가능한 실제 광석 오브젝트
  렌더링: Vein 발견 시 AR 카메라에 3D 모델로 표시
  인터랙션: 탭하여 채굴 시작
  등급: Common/Rare/Epic/Legendary/Advertisement
  이펙트: 등급별 고유 시각 효과 및 파티클
```

**설계 의도:**
Fracture → Vein → Core 시스템은 Pokémon GO의 단순 탭 수집과 차별화하여 실제 AR 탐색 경험을 제공합니다. Fracture는 "어디로 가야 할지" 알려주고, Vein은 "AR로 찾아내는 보물찾기"를, Core는 "채굴하는 만족감"을 구현합니다. 명칭은 게임 세계관의 "Digital Crack Aesthetic"과 일치하며, 플레이어에게 세계관 몰입을 강화합니다.

### 2.2 Geofencing & Scene Transition

```yaml
Map Mode (주요 씬):
  화면: Map.unity 전체 화면
  표시 요소:
    - 플레이어 현재 위치
    - 100m 반경 내 모든 Fracture
    - Fracture 등급 표시 (아이콘 색상)
    - 거리 표시 (텍스트)
  업데이트: 5초마다 GPS 및 Fracture 목록 갱신

Fracture 진입 (Geofencing Trigger):
  조건:
    - 플레이어가 Fracture 경계(150m 반경) 진입
    - GPS 정확도 < 20m
    - 네트워크 연결 활성

  자동 전환:
    - 진동 + 사운드 피드백
    - "Fracture 진입!" 알림 (1초)
    - ARGame.unity Scene 로드 (1-2초)
    - Digital Crack 전환 애니메이션 (1.618초)

  데이터 전달:
    - Fracture 경계 정보 (GeoJSON)
    - Vein 좌표 (서버 응답)
    - Core 메타데이터 (grade, type, value)

AR Exploration Mode:
  화면: ARGame.unity AR 카메라
  탐색 시스템:
    - 거리 기반 힌트 (50m+ / 20-50m / 10-20m / <10m)
    - 오디오 힌트 (비프음 주파수 변화)
    - Vein 시각화 (10m 이내 진입 시)

  Core 렌더링:
    - Vein 발견 시 AR 카메라에 3D 모델 표시
    - 등급별 글로우 효과 + 파티클
    - "탭하여 채굴" UI 표시

채굴 완료 후 자동 복귀:
  - 보상 애니메이션 (3.14초 - π)
  - Map.unity 자동 복귀
  - 수집된 Vein 마커 제거
  - 새로운 Fracture 요청
```

**설계 의도:**
자동 Scene 전환은 끊김 없는 경험을 제공합니다. Map에서 Fracture를 보고 이동하는 전략 단계와, AR에서 Vein을 찾는 탐색 단계를 명확히 분리하여 두 가지 게임플레이를 모두 제공합니다. Geofencing 자동 감지로 수동 버튼 조작 없이 몰입을 유지합니다.

### 2.3 Fracture Generation Rules

```yaml
Server Generation Algorithm:
  Base Spawn Rate: 3 Fractures per S2 Level 15 cell
  Spawn Interval: 5-15 minutes (variable)
  Persistence: Fracture remains until collected

  Fracture Distribution:
    - Common Fractures: 70% (1-2 Common Cores)
    - Rare Fractures: 20% (1 Rare Core)
    - Epic Fractures: 8% (1 Epic Core)
    - Legendary Fractures: 2% (1 Legendary Core)

  Location Selection:
    - Public accessible areas only
    - Minimum 50m between Fractures
    - Avoid private property, dangerous locations
    - Prefer pedestrian-friendly zones

  Multi-Vein Fractures (Post-MVP):
    - Common Fractures: 1 Vein
    - Rare Fractures: 1-2 Veins
    - Epic Fractures: 2-3 Veins
    - Legendary Fractures: 3-5 Veins
    - Chain bonus for collecting multiple Veins in same Fracture
```

**설계 의도:**
Fracture 생성은 공정성과 안전을 우선합니다. 공공 장소 우선 배치로 접근성을 보장하고, 최소 거리 제한으로 과도한 밀집을 방지합니다. Post-MVP의 Multi-Vein 시스템은 더 긴 탐색과 더 큰 보상을 제공하여 게임플레이 깊이를 확장합니다.

## 3. Ore System

### 3.1 Ore Grades

| Grade         | Tier | Spawn Rate | Base Points | Color Code  | Respawn Time     |
| ------------- | ---- | ---------- | ----------- | ----------- | ---------------- |
| Common        | T1   | 70%        | 10          | #808080     | 5 min            |
| Rare          | T2   | 20%        | 50          | #0000FF     | 15 min           |
| Epic          | T3   | 8%         | 200         | #800080     | 60 min           |
| Legendary     | T4   | 2%         | 1000        | #FFD700     | 360 min          |
| Advertisement | T0   | Variable   | 30 + bonus  | Brand color | Campaign setting |

**설계 의도:**
가챠 게임의 검증된 확률 분포를 차용했지만, 실제 이동이 필요한 LBS 특성을 고려해 더 관대하게 조정했습니다. Common 70%는 실망감을 최소화하고, Legendary 2%는 "오늘 운이 좋을까?" 기대감을 유지합니다. 광고 광석은 수익 모델이면서도 일반 광석보다 3배 포인트를 제공해 유저가 광고를 '벌이 되는 콘텐츠'로 인식하게 합니다.

### 3.2 Spawn Parameters

```yaml
Grid System:
  Cell Type: S2 Geometry Level 15
  Cell Size: ~100m x 100m
  Max Ores per Cell: 5
  Collection Range: 10 meters (regular users) / 3 meters (Genesis 1000 members)

Base Spawn Formula:
  ores_per_cell = base_rate * area_modifier * time_modifier * weather_modifier
  base_rate = 3
```

**설계 의도:**
S2 Geometry는 Pokémon GO에서 검증된 시스템으로, 지구를 균등한 셀로 나누어 공정한 분포를 보장합니다. 100m x 100m는 도시 블록 하나 정도로, 한 지역에서 5-10분 정도 머물 가치를 제공합니다. 수집 거리는 일반 사용자 10m, Genesis 1000 멤버 3m로 차별화되어 Genesis 멤버에게 더 정확한 위치 추적 경험을 제공하며, GPS 오차(±5m)를 고려하면서도 실제로 가까이 가야 하는 긴장감을 유지합니다.

### 3.3 Area Modifiers

| Area Type    | Spawn Modifier | Ad Ore Bonus | Special          |
| ------------ | -------------- | ------------ | ---------------- |
| Urban Center | 1.5x           | +50%         | More ad ores     |
| Residential  | 1.0x           | +0%          | Balanced         |
| Park/Nature  | 1.2x           | -20%         | +10% rare chance |
| Industrial   | 0.8x           | +30%         | +5% epic chance  |

**설계 의도:**
현실 세계의 특성을 게임에 반영하여 탐험의 다양성을 제공합니다. 도심은 광고주 타겟 지역이므로 광고 광석이 많고, 공원은 운동과 산책을 장려하기 위해 희귀 광석 확률을 높였습니다. 산업 지역은 접근성이 낮은 대신 에픽 광석으로 보상합니다. 이는 플레이어가 다양한 지역을 탐험하도록 유도하는 핵심 메커니즘입니다.

### 3.4 Time Modifiers

| Time Slot    | Hours       | Spawn Rate | XP Bonus | Point Bonus |
| ------------ | ----------- | ---------- | -------- | ----------- |
| Morning Rush | 07:00-09:00 | +30%       | 2.0x     | 1.0x        |
| Lunch Break  | 12:00-13:00 | +20%       | 1.0x     | 1.0x        |
| Evening Rush | 18:00-20:00 | +30%       | 1.0x     | 1.5x        |
| Night        | 00:00-06:00 | -50%       | 1.0x     | 1.0x        |

**설계 의도:**
플레이어의 생활 패턴과 게임을 자연스럽게 통합합니다. 출근 시간 XP 2배는 하루를 활기차게 시작하는 동기를, 퇴근 시간 포인트 1.5배는 하루의 피로를 보상으로 전환합니다. 심야 스폰율 감소는 건강한 수면을 장려하면서도, 야간 근무자를 완전히 배제하지 않는 균형점입니다.

## 4. AR Discovery System

### 4.1 Discovery Effects

| Effect         | Trigger               | Duration   | Gameplay Impact              |
| -------------- | --------------------- | ---------- | ---------------------------- |
| Ore Glow       | Ore within 50m        | Continuous | Visibility through obstacles |
| Digital Ripple | Ore discovered        | 1.618s     | +10% collection speed        |
| Reality Crack  | Epic+ ore spawn       | 5s         | Reveals hidden ores nearby   |
| Hologram Burst | Successful collection | 3.14s      | Chain bonus trigger          |

**설계 의도:**
AR 효과는 단순한 시각적 요소가 아닌 게임플레이에 영향을 주는 메커니즘입니다. Ore Glow는 탐색을 돕고, Digital Ripple은 황금비 시간동안 보너스를 제공합니다. Reality Crack은 희귀 광석 발견의 보상을 확대하고, Hologram Burst의 π초는 연쇄 수집의 타이밍 윈도우를 만듭니다.

### 4.2 Environmental AR Effects

| Environment      | Occurrence     | Duration | Effect                                |
| ---------------- | -------------- | -------- | ------------------------------------- |
| Digital Rain     | Random (5%)    | 30 min   | Ore quality +1 tier, spawn rate +100% |
| Reality Grid     | Every 6 hours  | 10 min   | Shows all ores in 200m radius         |
| Floating Islands | Weekly event   | 2 hours  | Epic+ ores only, 50m above ground     |
| Portal Gates     | Midnight daily | 15 min   | Teleport to special zone              |

**설계 의도:**
환경 AR은 예측 가능한 이벤트와 불규칙한 보너스를 혼합하여 일상적 플레이에 변화를 줍니다. Digital Rain은 로또 같은 행운을, Reality Grid는 정기적 보상을, Floating Islands는 주간 목표를, Portal Gates는 일일 클라이맥스를 제공합니다.

## 5. Mining Tools

### 5.1 Tool Tiers

| Tier    | Efficiency | Mining Time | Unlock Level | Point Cost | Special Effect              |
| ------- | ---------- | ----------- | ------------ | ---------- | --------------------------- |
| T1      | 1.0x       | 3.0s        | 0            | 0          | None                        |
| T2      | 1.5x       | 2.5s        | 10           | 5,000      | +10% bonus point chance     |
| T3      | 2.0x       | 2.0s        | 25           | 20,000     | +20% bonus point chance     |
| T4      | 3.0x       | 1.5s        | 50           | 100,000    | +30% bonus, 5% double ore   |
| Genesis | 2.5x       | 1.5s        | Special      | N/A        | 2x points, +10% rare chance |

**설계 의도:**
곡괭이 시스템은 명확한 수직 성장 경로를 제공합니다. 효율 증가(1.0x → 3.0x)는 체감할 수 있을 만큼 크지만, 초보자를 압도하지 않을 정도로 조절했습니다. Genesis 곡괭이는 얼리어답터에 대한 영구적 보상으로, T4보다 약간 약하지만 특별한 혜택(2x 포인트)으로 차별화했습니다. 가격 곡선(5K → 20K → 100K)은 각 단계마다 약 2주의 플레이를 요구합니다.

### 5.2 Upgrade System [Post-MVP]

| Enhancement | Efficiency Bonus | Cost   | Success Rate |
| ----------- | ---------------- | ------ | ------------ |
| +1          | +10%             | 1,000  | 100%         |
| +2          | +20%             | 3,000  | 100%         |
| +3          | +30%             | 6,000  | 100%         |
| +10 (Max)   | +100%            | 50,000 | 100%         |

**설계 의도:**
MVP에서는 강화 실패가 없는 단순한 시스템으로 시작합니다. 이는 P2E 게임에서 흔한 '도박성 강화'에 대한 피로감을 피하고, 투자한 만큼 확실한 성장을 보장합니다. Post-MVP에서 복잡도를 추가할 여지를 남겨두되, 유저 친화적인 기조는 유지합니다.

## 6. Cooperative Systems

### 6.1 Chain Mining

| Players | Chain Multiplier | ORE Split | Bonus   |
| ------- | ---------------- | --------- | ------- |
| 2       | 1.5x             | 75% each  | +10% XP |
| 3       | 2.0x             | 70% each  | +15% XP |
| 4       | 2.5x             | 65% each  | +20% XP |
| 5+      | 3.0x             | 60% each  | +30% XP |

**메커니즘:**

- 모든 플레이어가 5초 내에 같은 ORE 터치
- 마지막 터치한 플레이어가 "Chain Master" 보너스 (+20%)
- 체인 성공 시 3초간 무적 시간 (다음 ORE 확보)

**설계 의도:**
혼자서는 100%를 얻지만, 둘이서는 각각 112.5%(75% × 1.5)를 얻는 구조로 협동이 항상 이득입니다. 5명 이상에서는 각자 180%(60% × 3)를 얻어 극적인 효율을 보장합니다. 이는 자연스러운 소셜 플레이를 유도합니다.

### 6.2 Reality Bridge

| Requirement | Duration          | Cost           | Reward                    |
| ----------- | ----------------- | -------------- | ------------------------- |
| 10+ players | Level avg × 1 min | 10 energy each | Access to Floating Island |
| 20+ players | Level avg × 2 min | 5 energy each  | 2x ore spawn on path      |
| 30+ players | Level avg × 3 min | Free           | Legendary ore guaranteed  |

**메커니즘:**

- 플레이어들이 일렬로 서서 "Bridge Mode" 활성화
- 첫 번째와 마지막 플레이어 사이에 빛의 다리 생성
- 다리 위에서는 이동 속도 +50%, 에너지 소모 없음

**설계 의도:**
대규모 협동 플레이의 정점입니다. 10명을 모으는 것은 어렵지만, 그만큼 특별한 보상(Floating Island 접근)을 제공합니다. 30명 이상에서는 비용이 없고 전설 광석을 보장하여 커뮤니티 이벤트의 하이라이트가 됩니다.

### 6.3 Collective Harvest

| Participants | Pool Multiplier | Distribution             | Special               |
| ------------ | --------------- | ------------------------ | --------------------- |
| 10-49        | 1.2x            | Equal share              | -                     |
| 50-99        | 1.5x            | Contribution-based       | Rare ore bonus        |
| 100-499      | 2.0x            | Tiered (Top 10% gets 2x) | Epic ore bonus        |
| 500-999      | 3.0x            | Tiered (Top 10% gets 3x) | Legendary appearance  |
| 1000+        | 5.0x            | Everyone wins            | Genesis Core fragment |

**메커니즘:**

- 매주 토요일 14:00-14:30 (30분간)
- 모든 수집 ORE가 공동 풀로 집계
- 이벤트 종료 시 일괄 분배

**설계 의도:**
주간 커뮤니티의 정점 이벤트입니다. 1000명 달성 시 Genesis Core 조각은 장기적 목표를 제공하며, 전 서버가 하나되는 경험을 만듭니다. 기여도 기반 분배는 공정성을, 하위 티어도 받는 보상은 포용성을 보장합니다.

### 6.4 Resonance Field

| Guild Size | Field Radius | Effect                       | Duration |
| ---------- | ------------ | ---------------------------- | -------- |
| 5-9        | 25m          | Ore +1 quality tier          | 5 min    |
| 10-19      | 50m          | Ore +1 tier, mining speed 2x | 10 min   |
| 20-49      | 100m         | Ore +2 tiers, speed 3x       | 15 min   |
| 50+        | 200m         | All epic+, instant mining    | 20 min   |

**메커니즘:**

- 같은 길드원이 반경 내 모임
- 자동으로 공명장 활성화
- 길드 레벨에 따라 추가 보너스

**설계 의도:**
길드 시스템의 핵심 메커니즘입니다. 작은 길드도 의미 있는 혜택을 받지만, 대규모 길드의 조직력은 극적인 보상으로 이어집니다. 20분 제한은 과도한 파밍을 방지하면서도 충분한 보상을 보장합니다.

## 7. Digital Weather System

### 7.1 Weather Types

| Weather      | Probability | Duration   | Effects                           | Warning     |
| ------------ | ----------- | ---------- | --------------------------------- | ----------- |
| Clear        | 60%         | -          | Normal gameplay                   | -           |
| Data Storm   | 15%         | 30-60 min  | Ore quality +50%, visibility -30% | 3hr advance |
| Bit Fog      | 10%         | 30-120 min | Hidden ores revealed, speed -20%  | 1hr advance |
| Quantum Rain | 10%         | 15-30 min  | All probabilities 2x              | No warning  |
| Binary Snow  | 5%          | 60-180 min | Energy recovery 2x, peaceful mode | 6hr advance |

**메커니즘:**

- 실제 날씨와 20% 연동 (비 오는 날 Data Storm 확률 증가)
- 날씨별 전용 BGM과 시각 효과
- 극단적 날씨 시 실내 플레이 보너스

**설계 의도:**
디지털 날씨는 일상적 플레이에 변수를 추가합니다. Data Storm은 리스크와 리턴의 선택을, Bit Fog는 탐험의 재미를, Quantum Rain은 도박의 스릴을, Binary Snow는 휴식과 회복을 제공합니다.

### 7.2 Reality Stability

| Stability | Color  | Spawn Rate | Quality         | Special        |
| --------- | ------ | ---------- | --------------- | -------------- |
| Stable    | Green  | 100%       | Normal          | -              |
| Unstable  | Yellow | 120%       | Random ±1 tier  | Glitch events  |
| Critical  | Red    | 150%       | Random ±2 tiers | Reality tears  |
| Breakdown | Black  | 0%         | -               | Emergency only |

**메커니즘:**

- 서버 부하, 동시접속자, 특별 이벤트에 따라 변동
- Critical 상태 30분 지속 시 Breakdown
- Breakdown 중에는 인벤토리 정리만 가능

**설계 의도:**
Reality Stability는 서버 상태를 게임 요소로 전환합니다. 불안정할수록 높은 보상이지만 리스크도 증가하여, 플레이어가 상황을 판단하고 전략을 세우게 합니다.

## 8. Special Zones

### 8.1 The Surge Zone

| Parameter     | Value     | Scaling      |
| ------------- | --------- | ------------ |
| Duration      | 15-30 min | Random       |
| Radius        | 500m      | Fixed        |
| Ore Spawn     | 10x       | All tiers    |
| Quality Bonus | +2 tiers  | Minimum rare |
| Player Limit  | 100       | First come   |

**메커니즘:**

- 하루 3-5회 무작위 발생
- 전체 서버 알림 (3분 카운트다운)
- 구역 내 PvP 비활성화 (평화 지대)

**설계 의도:**
Surge Zone은 일일 플레이의 클라이맥스입니다. 제한된 시간과 인원은 긴장감을, 10배 스폰은 흥분을, PvP 비활성화는 공정한 경쟁을 보장합니다.

### 8.2 Crystal Garden

| Tier    | Entry Requirement | Duration | Rewards       |
| ------- | ----------------- | -------- | ------------- |
| Bronze  | Level 20+         | 30 min   | Rare+ only    |
| Silver  | Level 30+         | 45 min   | Epic+ only    |
| Gold    | Level 40+         | 60 min   | Legendary 10% |
| Diamond | Level 50+         | 90 min   | Legendary 25% |

**메커니즘:**

- 매월 1일 위치 공개 (한 달간 유지)
- 일일 1회 입장 제한
- 티어별 분리된 인스턴스

**설계 의도:**
Crystal Garden은 성장의 이정표입니다. 레벨별 티어 분리로 공정한 경쟁을 보장하고, 월간 위치 변경으로 새로운 탐험을 유도합니다.

### 8.3 Shadow Mine

| Time        | Risk Level | Spawn Rate | Special Ore    | Penalty           |
| ----------- | ---------- | ---------- | -------------- | ----------------- |
| 00:00-01:00 | Low        | 2x         | Shadow Ore 10% | Energy -2x        |
| 01:00-02:00 | Medium     | 3x         | Shadow Ore 20% | Energy -3x        |
| 02:00-03:00 | High       | 5x         | Shadow Ore 30% | Energy -5x        |
| 03:00-04:00 | Extreme    | 10x        | Shadow Ore 50% | Account flag risk |

**메커니즘:**

- Void ORE 출현 (수집 시 디버프)
- 사망 시 당일 수집 ORE 50% 손실
- Shadow Ore는 특별 상점에서만 교환

**설계 의도:**
Shadow Mine은 하이리스크 하이리턴의 극단입니다. 심야 시간대 설정으로 건강한 플레이를 저해하지 않으면서도, 모험을 원하는 플레이어에게 선택지를 제공합니다.

### 8.4 Community Quarry

| Phase       | Duration | Objective             | Reward Multiplier   |
| ----------- | -------- | --------------------- | ------------------- |
| Gathering   | 10 min   | Collect any ore       | 1x                  |
| Racing      | 10 min   | Speed competition     | 1.5x for top 50%    |
| Cooperation | 10 min   | Chain mining          | 2x for participants |
| Boss        | 5 min    | Mega ore (100,000 HP) | 3x for contributors |

**메커니즘:**

- 매주 토요일 14:00 시작
- 페이즈별 다른 플레이 스타일 요구
- 최종 보스는 전원 협력 필수

**설계 의도:**
Community Quarry는 주간 소셜 이벤트의 정점입니다. 다양한 페이즈로 모든 플레이 스타일을 포용하고, 보스 레이드는 MMO의 재미를 경험하게 합니다.

## 9. Progression System

### 9.1 Experience Formula

```
Required_XP = 100 * (Level ^ 1.5)
```

**설계 의도:**
지수 1.5는 선형(지루함)과 제곱(가파름) 사이의 스위트 스팟입니다. 초반 10레벨까지는 빠르게 성장하여 게임의 핵심 시스템을 모두 경험하게 하고, 이후로는 점진적으로 느려져 장기 목표를 제공합니다. 레벨 50까지 약 4-6개월이 걸리도록 설계하여 지속적인 플레이 동기를 유지합니다.

### 9.2 Level Requirements

| Level | Required XP | Cumulative XP | Estimated Days |
| ----- | ----------- | ------------- | -------------- |
| 1→2   | 100         | 100           | 0.5            |
| 5→6   | 559         | 1,377         | 2              |
| 10→11 | 1,581       | 5,044         | 5              |
| 20→21 | 4,472       | 24,594        | 15             |
| 30→31 | 8,215       | 58,095        | 30             |
| 40→41 | 12,649      | 105,171       | 60             |
| 50→51 | 17,677      | 166,353       | 120            |

### 9.3 Level Rewards

| Level | Unlock                         | Bonus                          |
| ----- | ------------------------------ | ------------------------------ |
| 5     | T2 Tool Access, Friends        | Energy +10                     |
| 10    | T2 Tool Free, Quest Slot +1    | Chat, Digital Aura             |
| 15    | Leaderboard                    | Energy Recovery +20%           |
| 20    | T3 Tool Access, Crystal Garden | Ad Skip 3x/day                 |
| 25    | T3 Tool Free                   | Energy +20                     |
| 30    | Advanced Features, Shadow Mine | Special Effects                |
| 40    | Master Status                  | 1hr Unlimited Energy/day       |
| 50    | Grandmaster                    | T4 Tool Access, Diamond Garden |

**설계 의도:**
레벨 보상은 '기능 해금' 중심으로 설계하여 각 레벨업이 새로운 경험을 제공합니다. 5레벨마다 주요 마일스톤을 배치하여 중기 목표를 명확히 하고, 무료 곡괭이 지급(Lv10, 25)으로 무과금 유저도 성장을 체감할 수 있게 합니다. 레벨 40의 '1시간 무제한'은 하드코어 유저에 대한 특별한 인정입니다.

## 10. Quest System

### 10.1 Daily Quests

| Quest Type       | Requirement         | Point Reward | XP Reward | Reset       |
| ---------------- | ------------------- | ------------ | --------- | ----------- |
| Movement         | 1km distance        | 100          | 50        | 06:00 daily |
| Collection       | 20-50 ores (scaled) | 200          | 100       | 06:00 daily |
| Cooperation      | 1 chain mining      | 300          | 150       | 06:00 daily |
| Special (Random) | Variable            | 500          | 200       | 06:00 daily |

**설계 의도:**
일일 퀘스트는 최소한의 플레이를 보장하는 안전장치입니다. 1km 이동은 WHO 권장 최소 운동량의 1/5로, 건강한 습관 형성을 돕습니다. 협동 퀘스트 추가로 소셜 플레이를 일상화합니다.

### 10.2 Weekly Quests

| Quest       | Requirement           | Point Reward | XP Reward |
| ----------- | --------------------- | ------------ | --------- |
| Consistency | 7 day streak          | 2,000        | 1,000     |
| Distance    | 10km total            | 1,500        | 750       |
| Volume      | 200 ores              | 1,500        | 750       |
| Social      | Play with 5 friends   | 1,000        | 500       |
| Event       | Join Community Quarry | 2,500        | 1,250     |

**설계 의도:**
주간 퀘스트는 장기적 참여를 유도합니다. Community Quarry 참여 보상을 최고로 설정하여 주간 이벤트 참여율을 높입니다.

## 11. Energy System

### 11.1 Energy Parameters

| Parameter          | Value                  |
| ------------------ | ---------------------- |
| Max Energy (Base)  | 100                    |
| Mining Cost        | 1 per ore              |
| Natural Recovery   | 10 per 10 min          |
| Full Recovery Time | 100 min                |
| Level Up Recovery  | 100% instant           |
| Ad Watch Recovery  | 30 (3x daily)          |
| Weather Bonus      | Up to 2x (Binary Snow) |

**설계 의도:**
에너지 시스템은 과몰입 방지와 수익화의 균형점입니다. 100분 완충은 수면 시간 동안 충전되어 아침에 풀 에너지로 시작할 수 있게 합니다. 날씨 보너스는 환경 시스템과 연동하여 다양성을 제공합니다.

### 11.2 Energy Scaling

| Level Range | Max Energy | Recovery Bonus |
| ----------- | ---------- | -------------- |
| 1-9         | 100        | 0%             |
| 10-19       | 110        | 0%             |
| 20-29       | 130        | +20%           |
| 30-39       | 150        | +20%           |
| 40-49       | 180        | +20%           |
| 50+         | 200        | +20%           |

**설계 의도:**
레벨에 따른 에너지 증가는 성장의 체감과 플레이 시간 확장을 동시에 달성합니다. 20레벨부터 회복 속도 보너스를 제공하여 중반 이후 플레이 템포를 가속화합니다.

## 12. Social Systems

### 12.1 Friend System

| Feature            | Limit/Value    |
| ------------------ | -------------- |
| Max Friends        | 100            |
| Nearby Bonus       | +10% points    |
| Daily Gift         | 10 energy      |
| Party Size         | 5 players      |
| Party Bonus        | +20% XP        |
| Chain Mining Bonus | +5% per friend |

**설계 의도:**
100명 제한은 의미 있는 관계 형성이 가능한 던바의 수를 고려했습니다. Chain Mining 보너스 추가로 친구와 함께 플레이하는 구체적 이점을 제공합니다.

### 12.2 Guild System

| Feature         | Requirement             | Benefit                 |
| --------------- | ----------------------- | ----------------------- |
| Create Guild    | Level 20, 10,000 points | Guild banner, chat      |
| Max Members     | 50                      | Scales with guild level |
| Guild Level     | Total member XP         | Unlock guild events     |
| Guild War       | Weekly                  | Territory control       |
| Guild Resonance | 5+ members nearby       | Field effects           |

**설계 의도:**
길드는 엔드게임 콘텐츠의 핵심입니다. 50명 제한은 관리 가능한 규모를 유지하면서도 대규모 협동을 가능하게 합니다. Guild Resonance는 오프라인 모임을 장려합니다.

### 12.3 Leaderboard Tiers

| Rank    | Daily Reward | Weekly Reward        | Monthly Reward    |
| ------- | ------------ | -------------------- | ----------------- |
| 1       | 1,000 points | 5,000 points + Badge | Genesis Tool Skin |
| 2-10    | 500 points   | 2,500 points         | Premium Currency  |
| 11-50   | 200 points   | 1,000 points         | Rare Cosmetic     |
| 51-100  | 100 points   | 500 points           | Common Cosmetic   |
| 101-500 | 0            | 200 points           | Title             |

**설계 의도:**
월간 보상 추가로 장기 경쟁 동기를 제공합니다. 1위의 Genesis Tool Skin은 수집욕을 자극하는 유니크 보상입니다.

## 13. Economy

### 13.1 Point Shop

| Item                | Category   | Cost  | Effect                    | Duration  |
| ------------------- | ---------- | ----- | ------------------------- | --------- |
| Energy Potion (30)  | Consumable | 100   | +30 energy                | Instant   |
| Energy Potion (50)  | Consumable | 150   | +50 energy                | Instant   |
| Energy Potion (100) | Consumable | 250   | +100 energy               | Instant   |
| 2x Points           | Booster    | 500   | Double points             | 30 min    |
| 2x XP               | Booster    | 500   | Double XP                 | 30 min    |
| Ore Detector        | Booster    | 1,000 | Show all ores             | 60 min    |
| Weather Shield      | Booster    | 750   | Ignore weather penalty    | 60 min    |
| Chain Magnet        | Booster    | 1,500 | Auto-chain nearby players | 30 min    |
| Inventory +10       | Permanent  | 2,000 | +10 slots                 | Permanent |
| Auto-collect        | Permanent  | 3,000 | Auto pickup               | Permanent |
| Teleport            | Consumable | 500   | Instant travel            | 1 use     |
| Guild Banner        | Permanent  | 5,000 | Custom guild flag         | Permanent |

**설계 의도:**
Weather Shield와 Chain Magnet 추가로 새로운 시스템과 연동된 아이템을 제공합니다. 가격은 일일 획득량(2,500)을 기준으로 책정하여 균형을 유지합니다.

### 13.2 Premium Features

| Feature      | Price  | Benefits                                                                         |
| ------------ | ------ | -------------------------------------------------------------------------------- |
| VIP Monthly  | $9.99  | 2x energy recovery, 100 daily points, No ads, +50 friend slots, Weather forecast |
| Starter Pack | $4.99  | 1000 points, T2 tool, 50 energy potions                                          |
| Value Pack   | $19.99 | 5000 points, T3 tool, 100 energy potions                                         |
| Season Pass  | $14.99 | 30-day rewards track, exclusive cosmetics, bonus XP                              |

**설계 의도:**
Season Pass 추가로 월간 지속 구매 옵션을 다양화합니다. VIP에 날씨 예보 기능을 추가하여 전략적 플레이를 지원합니다.

### 13.3 Ad Revenue Parameters

| Ad Type           | Reward         | Daily Limit | Duration |
| ----------------- | -------------- | ----------- | -------- |
| Energy Recovery   | 30 energy      | 3           | 30s ad   |
| Point Booster     | 2x for 30min   | 1           | 30s ad   |
| Quest Reroll      | New quest      | 1           | 15s ad   |
| Ad Ore Collection | 3x base points | Unlimited   | 5s ad    |
| Weather Skip      | Change weather | 1           | 30s ad   |

**설계 의도:**
Weather Skip 추가로 불리한 날씨를 회피할 수 있는 옵션을 제공합니다. 전략적 사용이 필요하도록 일일 1회로 제한합니다.

## 14. Balance Formulas

### 14.1 Daily Progression

```yaml
Daily_Points = (Ores_Collected * Average_Value * Tool_Efficiency * Weather_Modifier) + Quest_Rewards + Event_Bonus
Daily_XP = (Ores_Collected * 10) + (Distance_km * 100) + Quest_XP + (Chain_Mining * 50)

Expected Values:
  Casual (20min): 500-800 points, 300-500 XP
  Regular (60min): 1,500-2,500 points, 1,000-1,500 XP
  Hardcore (120min+): 4,000-7,000 points, 3,000-5,000 XP
  Event Day: +50-200% all values
```

**설계 의도:**
이벤트와 날씨 변수를 추가하여 일일 수익의 변동성을 높였습니다. 이는 매일 같은 루틴의 지루함을 방지하고, "오늘은 특별한 날"이라는 기대감을 제공합니다.

### 14.2 Economic Balance

```yaml
Daily Income:
  Quest_Points: 800-1,100 (with cooperation)
  Mining_Points: 1,500 (average)
  Event_Points: 200-1,000 (variable)
  Total: 2,500-3,600 points/day

Daily Spending:
  Energy: 250
  Boosters: 500-1,000
  Special Items: 200-500
  Total: 950-1,750 points/day

Net Gain: 750-2,650 points/day
Time to T3 Tool: 8-27 days (event dependent)
```

**설계 의도:**
이벤트 참여도에 따라 수익 격차를 크게 설정하여, 적극적 참여를 유도합니다. 최소 보장 수익은 유지하되, 최대 수익은 크게 높여 도전 의욕을 자극합니다.

## 15. Technical Parameters

### 15.1 System Constants

| Parameter             | Value    | Description                                   |
| --------------------- | -------- | --------------------------------------------- |
| GPS_UPDATE_INTERVAL   | 5s       | Location polling rate                         |
| MIN_MOVEMENT_DISTANCE | 5m       | Minimum distance for update                   |
| COLLECTION_RADIUS     | 10m / 3m | Ore collection range (Regular / Genesis 1000) |
| CHAIN_RADIUS          | 15m      | Chain mining range                            |
| RESONANCE_CHECK       | 30s      | Guild field update                            |
| MAX_SPEED             | 30 km/h  | Anti-cheat speed limit                        |
| SESSION_TIMEOUT       | 30 min   | Inactive session limit                        |
| SYNC_INTERVAL         | 30s      | Server sync frequency                         |

**설계 의도:**
Chain과 Resonance 시스템을 위한 추가 파라미터를 설정했습니다. 15m Chain 반경은 실제로 가까이 모여야 하는 물리적 근접성을 요구합니다.

### 15.2 Performance Targets

| Metric           | Target | Critical |
| ---------------- | ------ | -------- |
| FPS (AR Mode)    | 30     | 20       |
| FPS (Mass Event) | 25     | 15       |
| Battery/Hour     | <15%   | <25%     |
| Data/Hour        | <10MB  | <50MB    |
| Load Time        | <3s    | <5s      |
| Response Time    | <100ms | <500ms   |
| Max Players/Area | 100    | 200      |

**설계 의도:**
대규모 이벤트를 고려한 성능 목표를 추가했습니다. 100명 동시 렌더링은 Community 이벤트의 최소 요구사항입니다.

## 16. Anti-Cheat Parameters

### 16.1 Validation Rules

| Check                | Threshold        | Action           |
| -------------------- | ---------------- | ---------------- |
| Speed Check          | >30 km/h         | Ignore movement  |
| Distance Jump        | >500m instant    | Flag account     |
| Collection Rate      | >60/hour         | Temporary ban    |
| Chain Mining Abuse   | >10 chains/min   | Cooldown penalty |
| GPS Spoofing Pattern | 3 detections     | Investigation    |
| Impossible Location  | Ocean/Restricted | No ore spawn     |
| Multi-account        | Same device/IP   | Account linking  |

**설계 의도:**
협동 시스템 악용을 방지하는 체크를 추가했습니다. Chain Mining 남용은 쿨다운으로 대응하여 정상 플레이는 방해하지 않습니다.

### 16.2 Fair Play Limits

| Limit               | Value      | Reset       |
| ------------------- | ---------- | ----------- |
| Daily Collections   | 500        | 06:00       |
| Daily Distance      | 50km       | 06:00       |
| Daily Chains        | 100        | 06:00       |
| Event Participation | 3 per type | Daily       |
| Session Length      | 6 hours    | Per session |
| Concurrent Sessions | 1          | N/A         |

**설계 의도:**
일일 체인 제한과 이벤트 참여 제한을 추가하여 봇 파밍과 어뷰징을 방지합니다. 정상적인 하드코어 플레이는 여전히 가능한 수준입니다.

---

_This specification defines pure gameplay mechanics with design rationale. All values are subject to balancing during testing._

_Version: 2.1_
_Last Updated: 2025-10-06_
_Major Addition (v2.1): Fracture → Vein → Core system with geofencing and scene transitions (aligned with Digital Crack Aesthetic worldview)_
_Major Addition (v2.0): Cooperative systems, AR discovery mechanics, Digital weather, Special zones_
