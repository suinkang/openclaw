# 00. System Context & Container Diagram (C4)

## C4 Level 1 — System Context

OpenClaw가 어떤 외부 시스템과 상호작용하는지 보여주는 최상위 뷰.

```mermaid
flowchart TB
    User(["👤 사용자<br/>(개인 사용자)"])
    
    subgraph OC["🏠 OpenClaw 시스템 (자체 호스팅)"]
        OClaw["OpenClaw Gateway<br/>+ Plugins + Apps"]
    end
    
    subgraph Channels["📨 메시징 채널 (외부)"]
        TG[Telegram]
        DC[Discord]
        SL[Slack]
        WA[WhatsApp]
        IM[iMessage]
        Etc[기타 15+]
    end
    
    subgraph LLMs["🧠 LLM Providers (외부)"]
        AN[Anthropic]
        OA[OpenAI]
        GG[Google]
        Loc[Ollama / LM Studio]
    end
    
    subgraph Tools["🔧 외부 도구 (외부)"]
        GH[GitHub]
        Notion[Notion]
        MCP[MCP Servers]
        Apple[Apple Services]
    end
    
    subgraph Storage["💾 저장 (로컬)"]
        FS[(파일 시스템<br/>~/.openclaw)]
        SQL[(SQLite<br/>plugin-state)]
        Lance[(LanceDB<br/>memory)]
    end
    
    User <-->|음성/텍스트| Channels
    Channels <-->|봇 API| OClaw
    OClaw <-->|HTTPS/SSE| LLMs
    OClaw <-->|MCP/REST/oauth| Tools
    OClaw <--> Storage
    User -.->|macOS/iOS/Android 앱| OClaw
    User -.->|CLI / Web UI| OClaw
    
    style OC fill:#FFD700
    style Channels fill:#87CEEB
    style LLMs fill:#DDA0DD
    style Tools fill:#90EE90
    style Storage fill:#F0E68C
```

### 액터 정의

| 액터 | 역할 |
|------|------|
| **사용자** | 단일 개인 사용자 (멀티 테넌트 X). 자기 채널/모델/데이터 모두 소유 |
| **메시징 채널** | Telegram, Discord 등 20+ 외부 메시징 서비스 |
| **LLM Providers** | Anthropic/OpenAI/Google 등 모델 API |
| **외부 도구** | 스킬이 호출하는 SaaS (GitHub, Notion, ...) |
| **저장** | 모두 로컬 파일 시스템 (멀티 머신 동기화 X) |

### 핵심 제약

- **단일 사용자**: 멀티 테넌트 아님
- **자체 호스팅**: 모든 데이터/credentials 사용자 머신
- **로컬 우선**: 외부 클라우드 의존 없이 동작 가능 (로컬 LLM 사용 시)

---

## C4 Level 2 — Container Diagram

OpenClaw 내부 주요 컨테이너(프로세스/배포 단위).

```mermaid
flowchart TB
    User(["👤 사용자"])
    
    subgraph Apps["클라이언트 앱 (배포 단위)"]
        macOS["macOS 앱<br/>(SwiftUI + Sparkle)"]
        iOS["iOS 앱<br/>(SwiftUI + Watch + Share Ext)"]
        Android["Android 앱<br/>(Kotlin + Compose)"]
        WebUI["Web UI<br/>(TypeScript/React)"]
        CLI["CLI<br/>(openclaw command)"]
    end
    
    subgraph Server["Gateway 프로세스 (Node 22+)"]
        WS["WebSocket Server<br/>(ws + TypeBox/AJV)"]
        RPC["RPC Dispatcher<br/>(method registry)"]
        AUTH["Auth Manager<br/>(6 auth modes)"]
        SESS["Session Manager<br/>(lifecycle)"]
        AGENT["Agent Runtime<br/>(pi-embedded-runner)"]
        ROUTING["Routing Layer<br/>(channel→agent)"]
    end
    
    subgraph Plugins["플러그인 (lazy loaded)"]
        ChPlugins["Channel Plugins<br/>(Telegram, Discord, ...)"]
        ProvPlugins["Provider Plugins<br/>(Anthropic, OpenAI, ...)"]
        MemPlugins["Memory Plugins<br/>(active-memory, lancedb)"]
        ToolPlugins["Tool Plugins<br/>(canvas, browser, mcp)"]
    end
    
    subgraph Sub["Subprocess (필요시)"]
        Docker["Docker Sandbox<br/>(non-main 에이전트)"]
        IMSG["imsg CLI<br/>(JSON-RPC subprocess)"]
        MCP_S["MCP Servers<br/>(stdio subprocess)"]
    end
    
    subgraph Storage["로컬 저장 (~/.openclaw)"]
        Config[(openclaw.json)]
        Sessions[(sessions.json<br/>+ JSONL transcripts)]
        Auth[(auth-profiles.json)]
        PluginDB[(plugin-state.sqlite)]
        Memory[(memory/wiki.md<br/>+ LanceDB)]
        Cron[(cron/jobs.json)]
    end
    
    User --> Apps
    Apps -->|WebSocket ws://| WS
    CLI -->|stdio| WS
    
    WS --> RPC
    WS --> AUTH
    RPC --> SESS
    SESS --> AGENT
    AGENT --> ROUTING
    AGENT --> Plugins
    
    ChPlugins -->|HTTPS/Webhook| ExtCh["메시징 서비스"]
    ProvPlugins -->|HTTPS/SSE| ExtLLM["LLM API"]
    
    AGENT --> Sub
    
    SESS --> Sessions
    AUTH --> Auth
    AGENT --> Config
    Plugins --> PluginDB
    MemPlugins --> Memory
    SESS --> Cron
    
    style Server fill:#FFE4B5
    style Plugins fill:#E0FFFF
    style Storage fill:#F0E68C
    style Apps fill:#FFB6C1
    style Sub fill:#D8BFD8
```

### 컨테이너별 책임

| 컨테이너 | 책임 | 기술 |
|---------|------|------|
| **macOS 앱** | 메뉴바, voice wake, push-to-talk, 자동 업데이트 | SwiftUI 5.9, Sparkle |
| **iOS 앱** | 모바일 노드, 음성, Canvas, Watch, Share Ext | SwiftUI, Xcode |
| **Android 앱** | 모바일 노드, talk mode, foreground service | Kotlin, Compose |
| **Web UI** | WebChat, 설정 대시보드, 로그/진단 | TypeScript/React |
| **CLI** | `openclaw` 명령, onboard/doctor/agent | Node.js |
| **Gateway 프로세스** | WebSocket RPC, 모든 비즈니스 로직 호스팅 | Node 22+, ws, AJV, TypeBox |
| **Channel Plugins** | 메시징 서비스 통합 (인바운드/아웃바운드 정규화) | grammy, discord-api-types, imsg, ... |
| **Provider Plugins** | LLM 호출 (auth/catalog/runtime) | `@mariozechner/pi-ai` (공통) |
| **Docker Sandbox** | non-main 에이전트 격리 실행 | Docker daemon |
| **MCP Servers** | 외부 도구 통합 | stdio / HTTP+SSE |

---

## C4 Level 3 — Component Diagram (Gateway)

Gateway 프로세스 내부 주요 컴포넌트. (자세한 컴포넌트 다이어그램은 [01-component-diagram.md](./01-component-diagram.md) 참고)

```mermaid
flowchart TB
    subgraph Net["Network Layer"]
        WSS[ws.WebSocketServer]
        TLS[TLS Runtime]
    end
    
    subgraph Proto["Protocol Layer"]
        Schema[TypeBox Schemas]
        AJV[AJV Compiled Validators]
        Frames[Request/Response/Event Frames]
    end
    
    subgraph Auth["Authentication"]
        AuthMgr[GatewayAuthResult Resolver]
        RateLim[AuthRateLimiter<br/>10/min, 5min lockout]
        Pairing[Device Pairing]
    end
    
    subgraph Session["Session Management"]
        SessLifecycle[SessionLifecycleState]
        SessStore[SessionStore<br/>JSON + JSONL]
        AbortCtl[ChatAbortControllers Map]
        AgentRunSeq[agentRunSeq Map]
    end
    
    subgraph Lanes["Concurrency Lanes"]
        MainLane[main lane]
        SubLane[subagent lane]
        CronLane[cron lane]
        SessLane["session:{id} lanes"]
    end
    
    subgraph PluginRT["Plugin Runtime"]
        PluginReg[PluginRegistry<br/>pinned]
        ManCache[Manifest LRU Cache]
        LazyRT[Lazy Runtime Modules]
    end
    
    subgraph Broadcast["Event Broadcasting"]
        Bcast[broadcast / broadcastToConnIds]
        Presence[Presence Snapshot]
        Health[Health Snapshot]
    end
    
    Net --> Proto
    Proto --> Auth
    Auth --> Session
    Session --> Lanes
    Session --> PluginRT
    Session --> Broadcast
    
    style Net fill:#FFE4E1
    style Proto fill:#E0FFFF
    style Auth fill:#FFFACD
    style Session fill:#F0FFF0
    style Lanes fill:#FFF0F5
    style PluginRT fill:#E6E6FA
    style Broadcast fill:#FFEFD5
```

---

## 핵심 외부 의존성

| 카테고리 | 라이브러리 | 용도 |
|---------|----------|------|
| **HTTP / WS** | `ws` | WebSocket 서버 |
| | `undici` | HTTP 클라이언트 (Node 표준) |
| **스키마 / 검증** | `typebox` | JSON Schema 생성 |
| | `ajv` | 컴파일된 검증 |
| **저장** | `@lancedb/lancedb` | 벡터 DB (memory) |
| | `better-sqlite3` (추정) | plugin-state |
| **LLM 추상화** | `@mariozechner/pi-ai` | OpenAI/Anthropic/Google 통합 |
| **채널** | `grammy` | Telegram |
| | `@buape/carbon` 미사용 → `discord-api-types` + `ws` | Discord |
| | `@discordjs/voice`, `opusscript` | Discord 음성 |
| **번들** | `rolldown` | A2UI bundle |
| **UI** | `lit`, `@a2ui/lit` | Canvas |
| **Dev tools** | `tsgo` (typecheck), `oxlint` (lint), `oxfmt` (format), `vitest` (test) | Rust/Go 기반 (성능) |

---

## 배포 단위

```mermaid
flowchart LR
    subgraph DesktopHost["사용자 데스크탑/서버"]
        GP[Gateway Process]
        FS1[(Storage)]
    end
    
    subgraph Mobile["모바일 디바이스"]
        iOSApp[iOS 노드]
        AndroidApp[Android 노드]
    end
    
    subgraph Cloud["클라우드 (선택)"]
        FlyApp[Fly.io 인스턴스<br/>+ /data 볼륨]
    end
    
    GP --- FS1
    iOSApp -.-> GP
    AndroidApp -.-> GP
    iOSApp -.-> FlyApp
    AndroidApp -.-> FlyApp
    
    style DesktopHost fill:#FFE4B5
    style Mobile fill:#FFB6C1
    style Cloud fill:#87CEEB
```

자세한 배포 토폴로지는 [09-deployment.md](./09-deployment.md) 참조.
