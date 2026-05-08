# OpenClaw 코드베이스 분석

OpenClaw(`github.com/openclaw/openclaw`) 프로젝트의 아키텍처와 구현을 상세히 분석한 문서 모음입니다.

## 개요

OpenClaw는 **TypeScript/Node.js 기반 개인 AI 어시스턴트 플랫폼**입니다. 사용자가 평소 사용하는 메시징 채널(Telegram, Slack, Discord, WhatsApp, Signal, iMessage 등 20여 개)에서 AI 비서로 작동하며, 자체 호스팅(self-hosted)을 기본으로 합니다.

> ⚠️ 이름의 "EXFOLIATE!" 슬로건은 옛 Claw 게임 오마주이지만, 이 프로젝트는 게임 재구현이 **아닙니다**. 1997년 Captain Claw 플랫포머 재구현은 별도 프로젝트(`github.com/pjasicek/OpenClaw`)입니다.

## 목차

### 개요 (Overview)
가이드 문서(AGENTS.md/CLAUDE.md) 기반의 광범위 개요. 빠른 이해용.

| 문서 | 내용 |
|------|------|
| [01-architecture.md](./01-architecture.md) | 모노레포 구조와 전체 아키텍처 개요 |
| [02-gateway.md](./02-gateway.md) | Gateway 서버, 프로토콜, RPC |
| [03-plugin-system.md](./03-plugin-system.md) | 플러그인/익스텐션 시스템과 SDK |
| [04-channels.md](./04-channels.md) | 메시징 채널 통합 계층 |
| [05-providers.md](./05-providers.md) | LLM Provider 추상화 |
| [06-agents.md](./06-agents.md) | Agent 런타임과 메모리 시스템 |
| [07-canvas-skills.md](./07-canvas-skills.md) | Canvas, A2UI, Skills |
| [08-clients.md](./08-clients.md) | macOS/iOS/Android/Web 클라이언트 |
| [09-tooling.md](./09-tooling.md) | 개발/테스트/배포 도구체인 |
| [10-design-principles.md](./10-design-principles.md) | 핵심 설계 철학과 보안 모델 |

### Deep Dive (실제 코드 분석)
실제 `.ts` 소스 코드를 직접 읽어 검증한 분석. 파일 경로 + 라인 번호 인용. 가이드 문서가 아닌 진짜 구현 기준.

| 문서 | 내용 |
|------|------|
| [deep-dive/01-gateway.md](./deep-dive/01-gateway.md) | Gateway 서브시스템 — `ws` + TypeBox/AJV, 6가지 인증 모드, 핸드셰이크 trace, startup flow 9단계 |
| [deep-dive/02-plugin-loader.md](./deep-dive/02-plugin-loader.md) | 플러그인 로더 — 매니페스트 256KB/JSON5, LRU 512, POSIX 검증, lazy runtime 실제 구현, Telegram(grammy)/Anthropic(`@mariozechner/pi-ai`) 의존성 |
| [deep-dive/03-agent-runtime.md](./deep-dive/03-agent-runtime.md) | Agent 런타임 — `AgentRuntimePlan` 9영역, `runEmbeddedPiAgent` 실제 흐름, subagent spawn 구현, Active Memory **15초 timeout** + circuit breaker, promptStyle 6가지 |
| [deep-dive/04-channels-canvas.md](./deep-dive/04-channels-canvas.md) | 채널 & Canvas — Discord(carbon **안 씀**), iMessage(**imsg CLI + JSON-RPC**), Canvas(**Rolldown** 번들러, **Lit + @a2ui/lit**), MCP(stdio/SSE/streamable-http) |

> ⚠️ Deep Dive 문서들이 일부 개요 문서의 추정치를 정정합니다 (예: Active Memory timeout 5초→15초, Discord carbon 미사용 등).

## 분석 대상

- 분석 시점: 2026-05-08
- 분석 대상 커밋: `main` 브랜치 최신
- 분석 범위: 레포 루트, `src/`, `extensions/`, `apps/`, `packages/`, `skills/`, `AGENTS.md`/`CLAUDE.md`, 빌드/CI 설정

## 핵심 요약

| 항목 | 값 |
|------|-----|
| 언어 | TypeScript (core), Swift (macOS/iOS), Kotlin (Android) |
| 패키지 매니저 | pnpm (Node 22+) |
| 모노레포 | pnpm workspace (`., ui, packages/*, extensions/*`) |
| 익스텐션 수 | 130+ 플러그인 (`extensions/`) |
| 번들 스킬 수 | 55+ (`skills/`) |
| 지원 채널 | 20+ (Telegram, Discord, Slack, WhatsApp, Signal, iMessage 등) |
| 지원 LLM 프로바이더 | OpenAI, Anthropic, Google, Bedrock, Ollama, LM Studio 등 |
| 빌드 도구 | tsgo (타입체크), oxlint (린트), oxfmt (포맷), vitest (테스트) |
| 배포 | Docker, Fly.io, Render, npm |
