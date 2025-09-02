# Project ORE - Documentation Hub

## Open Reality Engine: 완전한 문서 및 명세서

> **Project ORE 개발을 위한 중앙 문서 저장소**

[![Documentation](https://img.shields.io/badge/docs-wiki-blue)](https://github.com/ore-protocol/ore-docs/wiki)
[![Status](https://img.shields.io/badge/status-MVP%20Development-orange)](https://github.com/orgs/ore-protocol/projects/1)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

---

## 🎯 목적

이 저장소는 **Project ORE** - 차세대 AR 위치 기반 P2E 광고 플랫폼의 모든 문서를 포함합니다. 문서는 **소스 형식**(이 레포)과 **Wiki 형식**(편리한 열람용) 두 가지로 구성되어 있습니다.

## 🚀 빠른 접근

### 📖 [**Wiki 읽기**](https://github.com/ore-protocol/ore-docs/wiki) ← **여기서 시작하세요!**

### 🔗 주요 문서

- **[Executive Brief](https://github.com/ore-protocol/ore-docs/wiki/Executive-Brief)** - 프로젝트 개요 (1페이지)
- **[MVP Definition](https://github.com/ore-protocol/ore-docs/wiki/MVP-Definition)** - 개발 범위 및 목표
- **[Development Roadmap](https://github.com/ore-protocol/ore-docs/wiki/Development-Roadmap)** - 16주 타임라인
- **[Business Plan](https://github.com/ore-protocol/ore-docs/wiki/Business-Plan)** - 완전한 비즈니스 모델

### 🔧 개발자를 위한 문서

- **[Backend Architecture](https://github.com/ore-protocol/ore-docs/wiki/Backend-Architecture)** - Rust + Go 시스템 설계
- **[AI Development Guide](https://github.com/ore-protocol/ore-docs/wiki/Backend-AI-Guide)** - Claude Code 베스트 프랙티스
- **[Setup Guide](https://github.com/ore-protocol/ore-platform)** - 개발 환경 설정

---

## 📚 문서 구조

### **🏢 Business & Strategy**

이해관계자, 투자자, 리더십 팀을 위한 전략 문서

| 문서                                                        | 목적                  | 대상              |
| ----------------------------------------------------------- | --------------------- | ----------------- |
| [Business Plan](docs/business/business-plan.md)             | 시장 분석, 수익 모델  | CEO, 투자자       |
| [MVP Definition](docs/business/mvp-definition.md)           | MVP 범위 및 성공 기준 | Product, Dev 팀   |
| [Executive Brief](docs/business/executive-brief.md)         | 1페이지 프로젝트 요약 | 모든 이해관계자   |
| [Branding Guide](docs/business/branding-guide.md)           | 플랫폼 브랜드 정체성  | Marketing, Design |
| [AI-Native Strategy](docs/business/ai-team-strategy.md)     | AI 우선 개발 접근법   | CTO, Dev 팀       |
| [Development Roadmap](docs/business/development-roadmap.md) | 16주 MVP 타임라인     | 모든 팀           |

### **🔧 Technical Architecture**

시스템 설계 및 구현 명세

| 문서                                                           | 목적                     | 대상           |
| -------------------------------------------------------------- | ------------------------ | -------------- |
| [Backend Specification](docs/technical/backend-spec.md)        | Rust + Go 마이크로서비스 | Backend 개발자 |
| [Frontend Specification](docs/technical/frontend-spec.md)      | Unity AR 클라이언트 설계 | Unity 개발자   |
| [Infrastructure Design](docs/technical/infrastructure-spec.md) | AWS 하이브리드 클라우드  | DevOps         |
| [API Documentation](docs/technical/api-documentation.md)       | REST & WebSocket APIs    | 모든 개발자    |

### **🎮 Game & Experience Design**

게임 메커니즘, 세계관, 사용자 경험 명세

| 문서                                                        | 목적                 | 대상             |
| ----------------------------------------------------------- | -------------------- | ---------------- |
| [Gameplay Specification](docs/game-design/gameplay-spec.md) | 핵심 게임 메커니즘   | Game Designer    |
| [Game Worldview](docs/game-design/game-worldview.md)        | Reality Layer 스토리 | Art, Marketing   |
| [UX Design Guide](docs/game-design/ux-guide.md)             | 사용자 경험 철학     | UX, Unity 개발자 |

### **🚀 Operations & Community**

마케팅, 커뮤니티 관리, 라이브 운영 가이드

| 문서                                                        | 목적                   | 대상           |
| ----------------------------------------------------------- | ---------------------- | -------------- |
| [Operations Roadmap](docs/operations/operations-roadmap.md) | 마케팅 & 운영 타임라인 | Marketing, Ops |
| [Mystery Campaign](docs/operations/mystery-campaign.md)     | 런칭 전 바이럴 전략    | Marketing      |
| [Genesis 1000](docs/operations/genesis-operations.md)       | 파운딩 커뮤니티        | Community 팀   |
| [Live Operations](docs/operations/live-ops-playbook.md)     | 출시 후 운영           | Operations     |

### **🤖 AI Development Guides**

AI-Native 개발 패턴 및 베스트 프랙티스

| 문서                                                                 | 목적                         | 대상           |
| -------------------------------------------------------------------- | ---------------------------- | -------------- |
| [Backend AI Guide](docs/ai-guides/backend-ai-guide.md)               | Backend용 Claude Code        | Backend 개발자 |
| [Frontend AI Guide](docs/ai-guides/frontend-ai-guide.md)             | AI 지원 Unity 개발           | Unity 개발자   |
| [Infrastructure AI Guide](docs/ai-guides/Infrastructure-ai-guide.md) | Infrastructure용 Cluade Code | DevOps 개발    |

---

## 🎯 현재 상태: MVP Development

### **Sprint 1 Focus** (Week 1-2)

- 🏗️ 개발 환경 설정
- ⚡ Location Service (Rust) - GPS & S2 Geometry
- 🎮 Game Service (Rust) - 코인 수집 로직
- 🚪 API Gateway (Go) - 요청 라우팅

### **주요 마일스톤**

| 날짜         | 마일스톤              | 상태 |
| ------------ | --------------------- | ---- |
| Sep 1, 2025  | Development 시작      | 🔄   |
| Oct 15, 2025 | Mystery Campaign 런칭 | 📋   |
| Nov 15, 2025 | Genesis 1000 선발     | 📋   |
| Dec 31, 2025 | **MVP 완성**          | 🎯   |

📊 **[진행 상황 추적](https://github.com/orgs/ore-protocol/projects/1)** - GitHub Project Board에서 확인

---

## 🤝 기여하기

### **팀 멤버를 위해**

1. **[Contributing Guide](CONTRIBUTING.md)** 읽기
2. **[AI-Native Team Strategy](docs/business/ai-team-strategy.md)** 검토
3. 현재 **[Sprint Goals](https://github.com/orgs/ore-protocol/projects/1)** 확인

### **문서 업데이트**

- **소스 파일**: 이 레포의 `.md` 파일 편집
- **Wiki 동기화**: GitHub Actions로 자동화
- **리뷰 프로세스**: 주요 변경사항은 PR 필수

### **AI-Native Development**

우리는 **Claude Code**, **Cursor**, **GitHub Copilot**을 사용하여 10배의 개발 속도를 달성합니다. 베스트 프랙티스는 **[AI Development Guides](docs/ai-guides/)**를 참조하세요.

---

## 🔗 Project Ecosystem

### **🏗️ Repositories**

- **[ore-platform](https://github.com/ore-protocol/ore-platform)** - 메인 코드베이스 (Rust + Go + Unity)
- **[ore-docs](https://github.com/ore-protocol/ore-docs)** - Documentation Hub (이 레포)
- **ore-contracts** - Smart Contracts (Coming Q1 2026)

### **🌐 Community** (Coming Soon)

- **Discord**: Genesis 1000 커뮤니티 (2025년 11월)
- **Twitter**: [@ore_protocol](https://twitter.com/ore_protocol)
- **Website**: [ore.io](https://ore.io) (개발 중)

---

## 📈 비전

### **Mission**

**Open Reality Engine** 구축 - 걷는 것이 수익이 되고, 광고가 투명해지며, 커뮤니티가 미래를 만들어가는 곳.

### **기술적 우수성**

- **2명 = 20명의 아웃풋** - AI-Native 개발을 통해
- **Rust로 성능** 극대화 (Location, Game 서비스)
- **Go로 속도** 확보 (Auth, Ad, Analytics 서비스)
- **Unity AR Foundation**으로 몰입형 경험 구현

### **비즈니스 모델**

- **사용자**: 일상 활동으로 월 $50-200 수익
- **광고주**: 100% 투명성으로 30배 높은 전환율
- **플랫폼**: 모든 거래에서 지속가능한 10% 수수료

---

## 🏆 성공 지표

| 지표               | MVP 목표 (Dec 31) | Post-MVP (Q1 2026) |
| ------------------ | ----------------- | ------------------ |
| Concurrent Users   | 1,000             | 10,000             |
| Genesis Community  | 1,000             | 유지               |
| Daily Active Users | 500               | 5,000              |
| Monthly Revenue    | $0 (Points)       | $10,000 (Tokens)   |

---

## 📞 연락처

- **Technical**: [dev@5010.tech](mailto:dev@5010.tech)
- **Business**: [biz@5010.tech](mailto:biz@5010.tech)
- **Community**: Discord (11월 중 오픈)

---

_"Between the zeros and ones, lies infinite possibility."_  
_"May your cracks be deep!" ⛏️_

**Built with**: 🤖 AI-Native Development • ⚡ Rust Performance • 🚀 Go Velocity  
**Target**: 🎯 Dec 31, 2025 MVP • 👑 Genesis 1000 Community  
**License**: MIT - 함께 미래를 만들어갑니다
