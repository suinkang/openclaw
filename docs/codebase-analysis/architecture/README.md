# OpenClaw 아키텍처 문서

UML 다이어그램과 소프트웨어 아키텍처 개념을 사용한 시스템 설계 문서. 모든 다이어그램은 [Mermaid](https://mermaid.js.org/) 형식.

## 목차

| 문서 | 다이어그램 | 내용 |
|------|------------|------|
| [00-system-context.md](./00-system-context.md) | C4 Context + Container | 외부 시스템 경계, 사용자/채널/LLM/Storage 관계 |
| [01-component-diagram.md](./01-component-diagram.md) | UML Component | 주요 모듈 (Gateway/Agents/Channels/Plugins/Storage) 의존 관계 |
| [02-class-diagrams.md](./02-class-diagrams.md) | UML Class | `AgentRuntimePlan`, `MessageReceipt`, `SessionEntry`, `AuthProfile` 등 핵심 도메인 모델 |
| [03-data-flow.md](./03-data-flow.md) | Sequence | 메시지 처리 / Subagent spawn / WebSocket 핸드셰이크 / OAuth refresh 시퀀스 |
| [04-state-machines.md](./04-state-machines.md) | StateDiagram | 8개 상태 머신 (Session, Message, Agent Run, WS, Pairing, Plugin, Memory, Compaction) |
| [05-storage-persistence.md](./05-storage-persistence.md) | ER + Tree | 디렉토리 트리, 파일 포맷, JSONL 트랜스크립트, SQLite 스키마 |
| [06-concurrency.md](./06-concurrency.md) | Activity + Sequence | Lane/Queue 모델, AbortController, Rate Limiter, Backpressure |
| [07-error-resilience.md](./07-error-resilience.md) | Decision Tree | Retry / Circuit Breaker / Failover / Durability 패턴 |
| [08-design-patterns.md](./08-design-patterns.md) | (참조) | 사용된 GoF/엔터프라이즈 패턴 카탈로그 |
| [09-deployment.md](./09-deployment.md) | Deployment | Docker/Fly.io/Render/모바일 노드 배치 |

## 분석 방법론

1. **Top-down**: System Context → Container → Component → Class
2. **Behavior**: Sequence (어떻게 흐르는가) + StateDiagram (어떻게 변하는가)
3. **Static**: Class diagram (무엇이 있는가) + ER (어떻게 저장되는가)
4. **Cross-cutting**: 동시성/에러/패턴 (모든 컴포넌트에 적용되는 원칙)

## 분석 기반

이 아키텍처 문서들은 **실제 `.ts` 소스 코드**를 직접 읽어 작성됐습니다. 약 500개 파일, 모든 주요 서브시스템(`src/gateway/`, `src/agents/`, `src/channels/`, `src/plugins/`, `src/config/`, `src/infra/`, `src/auto-reply/`, `extensions/*`)을 탐색.

## 관련 문서

- [../README.md](../README.md) — 분석 색인
- [../deep-dive/](../deep-dive/) — 영역별 deep dive
- 기존 개요 문서 (`01-architecture.md` ~ `10-design-principles.md`)
