# ORE - Live Operations Playbook

_라이브 서비스 운영 실행 가이드_

## Overview

- **목적**: ORE 라이브 서비스의 안정적이고 효율적인 운영을 위한 종합 가이드
- **독자**: 운영팀, CS팀, 커뮤니티 매니저, AI 네이티브 운영자
- **관련 문서**: ore-genesis-operations.md, ore-game-worldview.md, ore-game-play-spec.md
- **적용 시점**: 2026년 1월 15일 공식 런칭 이후
- **최종 수정**: 2024-12-20
- **버전**: 1.0

---

## 1. 운영 철학과 목표

### 1.1 Live Ops 철학

```yaml
핵심 원칙:
  "Living Reality Layer":
    - 게임은 살아있는 세계
    - Digital Weather는 실제로 변한다
    - 플레이어가 Reality를 shape한다
    - 매일이 새로운 발견

운영 가치:
  - Responsive: 24시간 내 모든 이슈 대응
  - Dynamic: 주간 단위 콘텐츠 업데이트
  - Fair: 모든 Miner에게 공정한 기회
  - Engaging: 일일 접속 이유 제공
  - Sustainable: 장기 운영 가능한 시스템
```

### 1.2 Success Metrics

| 지표               | 목표    | 측정 주기 | 위험 수준 |
| ------------------ | ------- | --------- | --------- |
| DAU                | 10,000+ | 일간      | <5,000    |
| WAU                | 50,000+ | 주간      | <25,000   |
| D1 Retention       | 50%+    | 일간      | <30%      |
| D7 Retention       | 30%+    | 주간      | <15%      |
| D30 Retention      | 15%+    | 월간      | <8%       |
| 서버 가동률        | 99.9%   | 실시간    | <99%      |
| CS 응답시간        | <24hr   | 일간      | >48hr     |
| NPS Score          | 50+     | 월간      | <30       |
| ARPU               | $5+     | 월간      | <$2       |
| 일일 이벤트 참여율 | 40%+    | 일간      | <20%      |

---

## 2. 일일 운영 (Daily Operations)

### 2.1 24시간 운영 사이클

```yaml
00:00-06:00 KST - Shadow Shift:
  00:00: 일일 리셋
    - 퀘스트 초기화
    - 리더보드 갱신
    - Shadow Mine 오픈

  03:14: π Midnight (자동)
    - 3.14x 보상 (5분간)
    - Hidden ORE 출현

  06:00: 일일 보상 지급
    - 로그인 보상
    - Genesis 보너스
    - VIP 혜택

06:00-12:00 KST - Morning Layer:
  06:00: Digital Weather 예보
    - 오늘의 날씨 공지
    - Mining 추천 지역
    - Special Event 예고

  09:00: Reality Check
    - 서버 상태 점검
    - 버그 리포트 확인
    - CS 티켓 분류

  10:00: Content Push
    - 소셜 미디어 포스팅
    - 커뮤니티 공지
    - 뉴스레터 발송

12:00-18:00 KST - Active Mining:
  12:00: Lunch Surge
    - 점심 이벤트 시작
    - 2x ORE (1시간)
    - Chain Mining 보너스

  15:14: π Time Event
    - 3.14초 스피드 마이닝
    - 황금비 보상 (1.618x)
    - Community Challenge

18:00-24:00 KST - Prime Time:
  19:00: Evening Assembly
    - 일일 통계 발표
    - Top Miners 공개
    - 내일 예고

  20:00: Surge Zone Activation
    - 무작위 지역 선정
    - 10x ORE 스폰
    - 30분 한정

  22:00: Community Quarry
    - 협동 이벤트
    - 대규모 보스 ORE
    - 길드 대항전

  23:00: Night Report
    - 일일 정산
    - 이슈 리포트
    - 익일 준비
```

### 2.2 Daily Checklist

```yaml
운영팀 체크리스트:
  [ ] 00:00 - 일일 리셋 확인
  [ ] 06:00 - Weather 예보 발송
  [ ] 09:00 - 서버/버그 체크
  [ ] 12:00 - 점심 이벤트 활성화
  [ ] 15:14 - π Time 모니터링
  [ ] 19:00 - 일일 통계 발표
  [ ] 20:00 - Surge Zone 검증
  [ ] 22:00 - Community Event
  [ ] 23:00 - 일일 리포트 작성

CS팀 체크리스트:
  [ ] 티켓 분류 (3회/일)
  [ ] 긴급 이슈 에스컬레이션
  [ ] FAQ 업데이트
  [ ] 커뮤니티 감정 체크
  [ ] 보상 지급 처리
```

### 2.3 Digital Weather System Operation

```yaml
날씨 운영 규칙:
  기본 로테이션:
    - Clear: 60% (안정)
    - Data Storm: 15% (도전)
    - Bit Fog: 10% (탐험)
    - Quantum Rain: 10% (행운)
    - Binary Snow: 5% (휴식)

  수동 조작 가능:
    - 특별 이벤트 시
    - 서버 부하 조절
    - 보상 밸런싱
    - 커뮤니티 요청 (투표)

날씨별 운영 포인트:
  Data Storm:
    - 사전 공지 필수
    - 서버 모니터링 강화
    - CS 대기 인원 증가

  Quantum Rain:
    - 경제 지표 모니터링
    - 인플레이션 체크
    - 긴급 중단 준비
```

---

## 3. 주간 운영 (Weekly Operations)

### 3.1 Weekly Schedule

```yaml
Monday - "Mining Monday":
  테마: 새로운 시작
  이벤트:
    - 주간 퀘스트 리셋
    - 2x XP (오전)
    - Weekly Challenge 시작
  운영:
    - 주간 목표 설정
    - 지난 주 리뷰

Tuesday - "Together Tuesday":
  테마: 협동의 날
  이벤트:
    - Chain Mining 2x 보너스
    - 길드 퀘스트
    - Friend Referral 2x
  운영:
    - 커뮤니티 피드백 수집

Wednesday - "Weather Wednesday":
  테마: 날씨 특집
  이벤트:
    - 특별 Weather 보장
    - Weather 예측 대회
    - Weather Master 선정
  운영:
    - 패치 배포 (필요시)

Thursday - "Throwback Thursday":
  테마: 과거 회상
  이벤트:
    - Classic ORE 출현
    - Genesis Story 공유
    - Old Timer 보상
  운영:
    - 다음 주 콘텐츠 준비

Friday - "Fortune Friday":
  테마: 행운의 날
  이벤트:
    - Legendary 확률 UP
    - Lucky Draw
    - Surge Zone 2회
  운영:
    - 주말 이벤트 준비

Saturday - "Social Saturday":
  테마: 커뮤니티
  이벤트:
    - Community Quarry XL
    - Guild War
    - 24h Mining Marathon 시작
  운영:
    - 증원 운영

Sunday - "Sacred Sunday":
  테마: 특별 지역
  이벤트:
    - Crystal Garden 오픈
    - Shadow Mine 확장
    - VIP 전용 Zone
  운영:
    - 주간 정산
    - 다음 주 계획
```

### 3.2 Weekly Tasks

```yaml
월요일 오전:
  - 주간 KPI 리뷰
  - 이벤트 성과 분석
  - 경제 지표 점검
  - 주간 목표 설정

수요일 오후:
  - 패치 노트 작성
  - 업데이트 배포
  - 긴급 수정 적용
  - A/B 테스트 시작

금요일 오후:
  - 주말 이벤트 점검
  - CS 인력 배치
  - 서버 스케일링
  - 모니터링 강화

일요일 저녁:
  - 주간 리포트 작성
  - 다음 주 콘텐츠 확정
  - 팀 미팅 준비
  - 이슈 리스트 정리
```

---

## 4. 월간 운영 (Monthly Operations)

### 4.1 Monthly Cycle

```yaml
월초 (1-7일):
  - 월간 테마 시작
  - 새로운 시즌 패스
  - Crystal Garden 위치 변경
  - Genesis Day 준비 (14일)

중순 (8-21일):
  - π Day Event (14일)
  - 대규모 업데이트
  - 월간 토너먼트
  - 밸런스 패치

월말 (22-31일):
  - 월간 정산 이벤트
  - 다음 달 예고
  - 시즌 패스 마감
  - Special Reward
```

### 4.2 Monthly Themes (2026)

```yaml
January: "New Reality"
  - 신규 유저 환영
  - Genesis 1000 기념
  - Tutorial 강화

February: "Love in the Layer"
  - 발렌타인 이벤트
  - Couple Mining
  - Pink ORE 출현

March: "π Month"
  - 3.14 대축제
  - Digital Big Bang 기념
  - The Architects 선발

April: "Spring Bloom"
  - 벚꽃 ORE
  - 신규 지역 오픈
  - Renewal 이벤트

May: "Mining Festival"
  - 어린이날 이벤트
  - Family Chain
  - Toy ORE

June: "Summer Surge"
  - 하계 이벤트 시작
  - Beach Zone 오픈
  - Water ORE

July: "Heat Wave"
  - 폭염 Weather
  - Ice ORE 출현
  - Cool Mining

August: "Reality Games"
  - 올림픽 이벤트
  - 국가 대항전
  - Medal ORE

September: "Autumn Crack"
  - 추석 이벤트
  - Harvest ORE
  - 풍요 보상

October: "Digital Halloween"
  - 호러 테마
  - Ghost ORE
  - Void 이벤트

November: "Thanks Mining"
  - 감사 이벤트
  - Gratitude ORE
  - 1주년 준비

December: "Anniversary"
  - 1주년 기념
  - Genesis Return
  - Legendary Rain
```

---

## 5. 이벤트 운영 프로세스

### 5.1 Event Planning

```yaml
이벤트 기획 (E-14일):
  컨셉:
    - 테마 결정
    - 세계관 연결
    - 보상 설계

  검증:
    - 경제 영향 분석
    - 기술 구현 가능성
    - CS 부하 예측

개발/테스트 (E-7일):
  구현:
    - 서버 로직
    - 클라이언트 UI
    - 보상 시스템

  QA:
    - 기능 테스트
    - 부하 테스트
    - 보상 검증

홍보 (E-3일):
  예고:
    - 소셜 미디어
    - 인게임 공지
    - 커뮤니티 티징

  준비:
    - CS 브리핑
    - FAQ 준비
    - 모니터링 설정
```

### 5.2 Event Types

```yaml
Regular Events (매일/매주):
  특징:
    - 자동화 운영
    - 예측 가능한 보상
    - 낮은 운영 부담

  예시:
    - π Time (일일)
    - Surge Zone (일일)
    - Community Quarry (주간)

Special Events (월간/시즌):
  특징:
    - 수동 운영 필요
    - 특별 보상
    - 마케팅 연계

  예시:
    - π Day (월간)
    - Genesis Day (연간)
    - Season Festival

Emergency Events (필요시):
  발동 조건:
    - 서버 이슈 보상
    - 경쟁사 대응
    - 지표 하락 대응

  예시:
    - Apology ORE Rain
    - Emergency Surge
    - Double Everything
```

### 5.3 Event Monitoring

```yaml
실시간 모니터링:
  - 참여율 (목표 대비)
  - 서버 부하
  - 오류 발생률
  - 경제 지표 변동

대응 기준:
  참여율 < 50% 목표:
    - 보상 상향
    - 추가 홍보
    - 진입 장벽 하향

  서버 과부하:
    - 인스턴스 추가
    - 큐 시스템 활성화
    - 단계별 진입

  경제 이상:
    - 즉시 중단 가능
    - 보상 조정
    - 롤백 준비
```

---

## 6. 긴급 대응 매뉴얼

### 6.1 Severity Levels

```yaml
SEV 1 - Critical (즉시 대응):
  정의: 전체 서비스 중단
  예시:
    - 서버 전체 다운
    - 데이터 손실
    - 보안 침해
  대응:
    - 전 팀원 소집
    - CEO 보고
    - 1시간 내 복구

SEV 2 - Major (1시간 내):
  정의: 주요 기능 장애
  예시:
    - 결제 오류
    - 로그인 불가
    - 게임 진행 불가
  대응:
    - 핵심 팀 대응
    - 공지 발송
    - 4시간 내 복구

SEV 3 - Minor (4시간 내):
  정의: 일부 기능 오류
  예시:
    - UI 버그
    - 특정 지역 오류
    - 이벤트 오작동
  대응:
    - 담당팀 처리
    - 모니터링
    - 24시간 내 수정

SEV 4 - Low (24시간 내):
  정의: 사용성 불편
  예시:
    - 번역 오류
    - 그래픽 깨짐
    - 밸런스 이슈
  대응:
    - 다음 패치 포함
    - 우선순위 조정
```

### 6.2 Crisis Response Protocol

```yaml
Step 1 - Detection (5분):
  - 이슈 확인
  - 영향 범위 파악
  - SEV 레벨 결정
  - 대응팀 소집

Step 2 - Communication (15분):
  - 내부 채널 공유
  - 초기 공지 발송
  - CS 템플릿 준비
  - 소셜 미디어 대응

Step 3 - Mitigation (30분):
  - 임시 조치 적용
  - 추가 피해 방지
  - 데이터 백업
  - 롤백 준비

Step 4 - Resolution (목표 시간 내):
  - 근본 원인 해결
  - 테스트 검증
  - 단계적 복구
  - 모니터링 강화

Step 5 - Recovery (해결 후):
  - 서비스 정상화
  - 보상 계획 수립
  - 상세 리포트
  - 포스트모템
```

### 6.3 Communication Templates

```yaml
초기 공지 (Detection): "Miners, we've detected an anomaly in the Reality Layer.
  Our team is investigating the crack.
  Stand by for updates.
  Time: [timestamp]
  #RealityStatus"

진행 상황 (Mitigation): "Reality Update: The crack is being sealed.
  Current Status: [상태]
  Estimated Time: [예상 시간]
  Affected: [영향 범위]
  Your ORE is safe."

해결 공지 (Resolution): "Reality Restored! The Layer is stable again.
  Compensation: [보상 내용]
  Duration: [다운타임]
  We apologize for the instability.
  May your cracks be deep! ⛏️"
```

---

## 7. 고객 지원 (Customer Support)

### 7.1 CS Guidelines

```yaml
응답 시간 SLA:
  Priority 1 (결제): 2시간
  Priority 2 (계정): 12시간
  Priority 3 (게임): 24시간
  Priority 4 (제안): 48시간

톤 앤 매너:
  - "Miner" 호칭 사용
  - Reality Layer 용어 활용
  - 친근하되 전문적
  - "May your cracks be deep!" 인사

에스컬레이션:
  Level 1: CS Agent
  Level 2: Senior CS
  Level 3: CS Manager
  Level 4: Operation Lead
```

### 7.2 Common Issues & Solutions

```yaml
계정 문제:
  로그인 불가:
    - 캐시 삭제 안내
    - 비밀번호 리셋
    - 계정 복구 프로세스

  진행 상황 손실:
    - 백업 데이터 확인
    - 복구 가능 여부
    - 보상 처리

결제 문제:
  중복 결제:
    - 거래 내역 확인
    - 환불 프로세스
    - 보너스 지급

  미지급 아이템:
    - 구매 로그 확인
    - 수동 지급
    - 추가 보상

게임플레이:
  버그 리포트:
    - 재현 단계 수집
    - 개발팀 전달
    - 임시 해결책 제공

  부정 행위 신고:
    - 증거 수집
    - 조사 프로세스
    - 결과 통보
```

### 7.3 Compensation Policy

```yaml
보상 기준:
  서버 다운:
    - 1시간 미만: 100 ORE
    - 1-4시간: 500 ORE
    - 4시간 이상: 1000 ORE + Premium Item

  버그 피해:
    - 아이템 손실: 2x 복구
    - 진행 롤백: 경험치 부스터
    - 이벤트 참여 불가: 이벤트 보상

  CS 실수:
    - 잘못된 정보: 50 ORE
    - 지연 응답: 100 ORE
    - 계정 오류: VIP 1일
```

---

## 8. 커뮤니티 관리

### 8.1 Community Channels

```yaml
Discord:
  규모: 10,000+ 멤버
  운영:
    - 24/7 모더레이션
    - 봇 자동화
    - 이벤트 채널
    - VIP 라운지

Reddit:
  r/OREMining:
    - 공식 서브레딧
    - 주간 개발자 AMA
    - 커뮤니티 콘테스트

Social Media:
  Twitter/X:
    - 일일 업데이트
    - 실시간 이슈 대응

  Instagram:
    - 비주얼 콘텐츠
    - Stories 이벤트

  TikTok:
    - 게임플레이 클립
    - Digital Weather 예보
```

### 8.2 Community Programs

```yaml
Ambassador Program:
  Genesis 1000:
    - 특별 권한
    - 월간 보상
    - 직접 피드백 채널

  Regional Leaders:
    - 지역별 커뮤니티 관리
    - 번역 지원
    - 로컬 이벤트

Content Creator:
  지원 내용:
    - Creator Code
    - 독점 정보
    - 프로모션 지원
    - 수익 공유

  요구사항:
    - 월 4개 콘텐츠
    - 품질 기준 충족
    - 커뮤니티 가이드라인
```

### 8.3 Moderation Policy

```yaml
금지 행위:
  즉시 영구 정지:
    - 핵/치트 사용
    - RMT (현금거래)
    - 개인정보 유출
    - 혐오 발언

  경고 후 정지:
    - 스팸/도배
    - 욕설/비방
    - 허위 정보
    - 규칙 위반

처벌 단계:
  1차: 경고
  2차: 24시간 정지
  3차: 7일 정지
  4차: 30일 정지
  5차: 영구 정지
```

---

## 9. 업데이트 관리

### 9.1 Update Cycle

```yaml
정기 업데이트:
  Daily Hotfix:
    - 긴급 버그 수정
    - 밸런스 조정
    - 이벤트 활성화

  Weekly Patch:
    - 수요일 점검
    - 1-2시간
    - 신규 콘텐츠

  Monthly Update:
    - 대규모 업데이트
    - 2-4시간 점검
    - 시스템 개선

점검 시간:
  정기 점검: 수요일 02:00-04:00 KST
  긴급 점검: 최소 1시간 전 공지
  확장 점검: 30분 단위 연장
```

### 9.2 Patch Notes Template

```yaml
[Version X.X.X] Reality Layer Update

🌟 New Features
  - [Feature 1]: [설명]
  - [Feature 2]: [설명]

⚡ Improvements
  - [개선 1]: [효과]
  - [개선 2]: [효과]

🔧 Bug Fixes
  - Fixed: [버그 설명]
  - Fixed: [버그 설명]

⚖️ Balance Changes
  - [아이템/시스템]: [변경 전] → [변경 후]

📅 Events
  - [이벤트명]: [기간]
  - Rewards: [보상 내용]

💎 Store Updates
  - New: [상품명]
  - Sale: [할인 정보]

Maintenance Compensation:
  - All Miners: 200 ORE
  - VIP: Additional 100 ORE

"May your cracks be deep!"
- The ORE Team
```

---

## 10. 성과 분석 및 리포팅

### 10.1 Daily Reports

```yaml
일일 리포트 (09:00 발송):
  수신: 경영진, 팀 리드

  핵심 지표:
    - DAU/CCU
    - 매출/ARPU
    - 신규/이탈
    - NPS 변화

  운영 현황:
    - 이벤트 참여율
    - 서버 상태
    - CS 처리율
    - 주요 이슈

  액션 아이템:
    - 긴급 대응 필요
    - 오늘 예정 사항
    - 리스크 요인
```

### 10.2 Weekly Business Review

```yaml
주간 리뷰 (월요일 14:00):
  참석: 전체 팀

  어젠다:
    - KPI 리뷰 (30분)
    - 이벤트 성과 (20분)
    - 이슈 및 해결 (20분)
    - 다음 주 계획 (20분)
    - Q&A (10분)

  산출물:
    - Action Items
    - Risk Register
    - Next Week Plan
```

### 10.3 Monthly Report

```yaml
월간 보고서:
  Executive Summary:
    - 월간 하이라이트
    - 핵심 성과
    - 주요 이슈

  Detailed Analysis:
    - 사용자 분석
    - 매출 분석
    - 콘텐츠 성과
    - 기술 안정성

  Insights:
    - 사용자 행동 변화
    - 시장 트렌드
    - 경쟁사 동향

  Recommendations:
    - 개선 제안
    - 신규 기회
    - 리스크 대응
```

---

## 11. AI 네이티브 운영

### 11.1 자동화 영역

```yaml
완전 자동화:
  - Digital Weather 시스템
  - π Time 이벤트
  - 일일 리셋/보상
  - 리더보드 갱신
  - 기본 CS 응답

AI 지원 운영:
  - 이벤트 밸런싱
  - 콘텐츠 생성
  - 데이터 분석
  - 이상 탐지
  - 감정 분석
```

### 11.2 AI Tools & Prompts

```yaml
일일 브리핑 생성: "Create a daily operations briefing for ORE Mining Game.
  Include: Yesterday's KPIs (DAU, Revenue, Issues),
  Today's focus (Events, Updates, Risks),
  Digital Weather forecast,
  Motivational message for the team.
  Tone: Professional but energetic.
  End with 'Let's mine another great day!'"

이벤트 아이디어: "Generate a weekly event idea for ORE Mining Game.
  Theme: [Current Month/Season]
  Duration: 3-7 days
  Must include: Unique mechanics, Reward structure,
  Connection to Reality Layer lore,
  Digital Weather integration.
  Constraints: No P2W, Accessible to all levels."

CS 응답 생성: "Draft a CS response for [Issue Type].
  Tone: Friendly, understanding, helpful.
  Include: Acknowledgment of issue,
  Clear solution steps,
  Compensation if applicable,
  Use Reality Layer terminology.
  Sign off: 'May your cracks be deep!'"
```

### 11.3 Human-in-the-Loop

```yaml
인간 결정 필요:
  - 보상 정책 변경
  - 경제 밸런스 조정
  - 위기 상황 대응
  - 커뮤니티 갈등 중재
  - 장기 전략 수립

인간 검토 필요:
  - AI 생성 콘텐츠
  - 자동 응답 품질
  - 이상 탐지 결과
  - 예측 모델 출력
```

---

## 12. 시즌 및 장기 운영

### 12.1 Season Structure

```yaml
시즌 기간: 3개월

Season 1 (Q1 2026): "The Awakening"
  - 테마: Reality Layer 발견
  - 신규 유저 유입 집중
  - Genesis 1000 스토리

Season 2 (Q2 2026): "The Expansion"
  - 테마: 영역 확장
  - 신규 지역 추가
  - Guild 시스템 강화

Season 3 (Q3 2026): "The Convergence"
  - 테마: 현실과 디지털 융합
  - PvP 콘텐츠 추가
  - 대규모 이벤트

Season 4 (Q4 2026): "The Revolution"
  - 테마: 1주년 + 블록체인
  - 토큰 이코노미 활성화
  - Web3 전환
```

### 12.2 Content Pipeline

```yaml
월간 콘텐츠:
  신규 추가:
    - 2-3개 이벤트
    - 1개 신규 지역
    - 5-10개 아이템
    - 1개 시스템 개선

  밸런스:
    - 경제 조정
    - 난이도 튜닝
    - 보상 최적화

분기별 대형 업데이트:
  - 신규 게임 모드
  - 시스템 리워크
  - 시즌 전환
  - 대규모 이벤트
```

---

## Appendix A: Emergency Contacts

```yaml
24/7 On-Call:
  Primary: [Lead Operator]
  Secondary: [Senior Operator]
  Escalation: [Operations Manager]

Technical:
  Server: [DevOps Lead]
  Database: [DBA]
  Security: [Security Lead]

Business:
  CEO: [Contact]
  CTO: [Contact]
  CFO: [Contact]

External:
  AWS Support: [Premium Support]
  Payment Provider: [24/7 Line]
  Legal: [Law Firm Contact]
```

---

## Appendix B: Tools & Dashboards

```yaml
Monitoring:
  - Grafana: Real-time metrics
  - CloudWatch: AWS monitoring
  - New Relic: APM
  - Sentry: Error tracking

Analytics:
  - Amplitude: User behavior
  - Tableau: Business intelligence
  - Google Analytics: Web traffic
  - Firebase: Mobile analytics

Communication:
  - Slack: Internal
  - Discord: Community
  - Zendesk: CS tickets
  - Mailchimp: Email marketing

Operations:
  - Jira: Task management
  - Confluence: Documentation
  - GitHub: Version control
  - Jenkins: CI/CD
```

---

## Appendix C: Quick Reference

### Event Codes

```
EVT001: Daily Reset
EVT002: π Time
EVT003: Surge Zone
EVT004: Community Quarry
EVT005: Digital Weather Change
EVT006: Emergency Maintenance
EVT007: Compensation Distribution
```

### Error Codes

```
ERR100: Connection Failed
ERR200: Authentication Error
ERR300: Transaction Failed
ERR400: Data Sync Error
ERR500: Server Error
ERR600: Maintenance Mode
```

### Compensation Codes

```
COMP001: Sorry for the crack (100 ORE)
COMP002: Reality restored (500 ORE)
COMP003: Deep apology (1000 ORE)
COMP004: Genesis gift (Special)
```

---

_본 플레이북은 ORE 라이브 서비스 운영의 바이블입니다._
_Reality Layer는 살아있으며, 우리는 그것을 함께 만들어갑니다._

_"Between the zeros and ones, we operate reality"_
_"May your operations be smooth and your cracks be deep!"_

_Version: 1.0_
_Effective Date: 2026-01-15_
_Last Updated: 2024-12-20_
_Next Review: 2026-02-01_

