# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Project ORE** is a comprehensive AR location-based P2E (Play-to-Earn) platform that combines the engagement of Pokémon GO with blockchain transparency. This repository (`ore-docs`) contains all technical specifications, business documentation, and AI development guides for the platform.

## Repository Structure

This is a **documentation-only repository** with markdown files organized by purpose:

- `docs/business/` - Strategy documents, MVP definition, branding guide, AI team strategy
- `docs/technical/` - System architecture specifications for backend, frontend, and infrastructure
- `docs/game-design/` - Gameplay mechanics, worldview, and UX guidelines
- `docs/operations/` - Marketing roadmap, live operations, Genesis 1000 community management
- `docs/ai-guides/` - AI-native development patterns and best practices for each technical domain

## Key Architecture Decisions

The project follows an **AI-Native development approach** with these core principles:

### Backend Architecture

- **Microservices**: 8 services split between Rust (performance-critical) and Go (business logic)
- **Event-driven**: Kafka-based with CQRS pattern
- **Performance targets**: 100K requests/sec for core services, designed to scale to 1M users without rewrite
- **Core services in Rust**: Location, Game, Realtime, Blockchain
- **Business services in Go**: Auth, User, Ad, Analytics

### Frontend Architecture

- **Unity 2023.3 LTS** with AR Foundation 5.1
- **Server-authoritative design**: Client displays only, all validation on server
- **Network-required**: Offline mode completely disabled for security
- **Performance targets**: 60 FPS, <500MB RAM, <10% battery/hour

### Infrastructure Architecture

- **2-tier environment strategy**: dev + prod (no staging)
- **Branch mapping**: dev branch → dev environment, main branch → prod environment
- **AWS naming convention**: "dev/prod" for consistent resource tagging
- **Hybrid cloud**: 85% AWS managed services, 15% on-premise (blockchain nodes, GPU)
- **Progressive complexity**: Start simple with ECS Fargate, migrate to EKS at scale

### Genesis 1000 Special Features

- **Exclusive community**: First 1000 members with special privileges
- **2x reward multiplier** for all in-game actions
- **Auto-collect range**: 3m vs 10m standard
- **Special UI styling**: Golden theme with pulsing animations
- **Airdrop eligibility**: 3% of total token supply reserved

## Common Development Commands

This is a documentation repository with no build system. Common operations:

### Documentation Management

```bash
# Check all markdown links (if you have a link checker installed)
markdown-link-check docs/**/*.md

# Generate table of contents for long documents
markdown-toc docs/technical/backend-spec.md

# Validate markdown syntax
markdownlint docs/**/*.md
```

### Content Validation

```bash
# Search for broken internal links
grep -r "](docs/" docs/ --include="*.md"

# Find TODO items across documentation
grep -r "TODO\|FIXME" docs/ --include="*.md"

# Check for outdated version references
grep -r "Version:" docs/ --include="*.md"
```

## AI Development Context

When working on this documentation:

1. **Technical Accuracy**: All specifications reflect production-ready architecture decisions, not theoretical designs
2. **Implementation Guidance**: Code examples and prompts are meant for actual implementation by AI agents
3. **Performance Focus**: All systems designed with specific performance targets that must be maintained
4. **Genesis Priority**: Any features should consider Genesis 1000 member benefits and special handling

## File Relationships

Key document dependencies to understand when editing:

- `docs/business/mvp-definition.md` → Drives all technical specifications
- `docs/technical/backend-spec.md` → Referenced by all AI guides and frontend spec
- `docs/ai-guides/backend-ai-guide.md` → Contains implementation prompts for backend-spec
- `docs/business/ai-team-strategy.md` → Defines overall AI development philosophy

## Documentation Standards

When updating documentation:

1. **Version control**: Update version numbers and dates at end of documents
2. **Cross-references**: Use relative links between related documents
3. **Code examples**: All code samples should be production-ready, not pseudocode
4. **Performance metrics**: Include specific numbers and benchmarks where applicable
5. **Genesis considerations**: Note any special behavior for Genesis 1000 members

## Project Status

- **Current Phase**: MVP Development (Weeks 1-16, Target: Dec 31, 2025)
- **Active Development**: Backend services in Rust/Go, Unity AR client
- **Target Metrics**: 1K concurrent users (MVP), 10K users (6 months), 100K users (1 year)
- **No Code**: This repository contains only documentation - actual implementation is in `ore-platform` repository

## External Dependencies

Documentation references these external systems:

- **Main codebase**: `ore-protocol/ore-platform` (Rust + Go + Unity)
- **AWS Infrastructure**: ECS Fargate, RDS, ElastiCache, MSK (Kafka)
- **Blockchain**: Polygon network for tokenomics (Post-MVP)
- **Third-party**: Google/Apple OAuth, Discord/Twitter verification for Genesis

## Important Notes

- This is purely documentation - no executable code in this repository
- All architectural decisions prioritize AI-native development patterns
- Performance targets are non-negotiable requirements, not aspirational goals
- Genesis 1000 features are core product differentiators, not optional add-ons
- The 16-week MVP timeline is aggressive but achievable with AI assistance

