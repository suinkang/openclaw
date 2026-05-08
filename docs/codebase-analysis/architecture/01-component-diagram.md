# 01. UML Component Diagram

OpenClaw 시스템의 컴포넌트 의존 관계. UML Component Diagram 형식.

## 1. Top-level Component Diagram

```mermaid
flowchart TB
    subgraph CLI["📟 CLI Container"]
        CliCmd["openclaw command<br/>(src/cli/)"]
    end
    
    subgraph Apps["📱 Native Apps"]
        macOSApp["macOS App"]
        iOSApp["iOS App"]
        AndroidApp["Android App"]
        WebUI["Web UI (ui/)"]
    end
    
    subgraph GW["⚙️ Gateway Process"]
        WSL["WebSocket Layer<br/>(src/gateway/server/)"]
        ProtoL["Protocol Layer<br/>(src/gateway/protocol/)"]
        Methods["RPC Methods<br/>(src/gateway/server-methods/)"]
        AuthL["Auth Manager<br/>(src/gateway/auth.ts)"]
        SessL["Session Manager<br/>(src/session/, src/config/sessions/)"]
        AgRT["Agent Runtime<br/>(src/agents/)"]
        Routing["Routing<br/>(src/routing/, src/channels/)"]
        ChL["Channel Core<br/>(src/channels/)"]
        AutoR["Auto-Reply<br/>(src/auto-reply/)"]
        Cron["Cron Scheduler<br/>(src/cron/)"]
        ToolReg["Tool Registry<br/>(src/tools/)"]
        Mem["Memory Host<br/>(src/memory/)"]
        ConfigL["Config Layer<br/>(src/config/)"]
        SecL["Secrets<br/>(src/secrets/)"]
    end
    
    subgraph PluginRT["🔌 Plugin Runtime"]
        PluginL["Plugin Loader<br/>(src/plugins/)"]
        SDK["Plugin SDK<br/>(src/plugin-sdk/, packages/plugin-sdk/)"]
        BundledP["Bundled Plugins<br/>(extensions/*)"]
    end
    
    subgraph External["🌐 External (out of process)"]
        DockerS["Docker Sandbox"]
        IMSGCli["imsg CLI"]
        MCPSrv["MCP Servers"]
        ExtAPIs["External APIs<br/>(Telegram, Discord, OpenAI, ...)"]
    end
    
    subgraph FS["💾 File System"]
        Cfg["~/.openclaw/openclaw.json"]
        SessFS["~/.openclaw/agents/{id}/sessions/"]
        AuthFS["~/.openclaw/agents/{id}/agent/auth-profiles.json"]
        PluginFS["~/.openclaw/plugin-state/state.sqlite"]
        MemFS["~/.openclaw/memory/"]
        CronFS["~/.openclaw/cron/jobs.json"]
    end
    
    %% CLI/Apps → Gateway
    CliCmd -->|RPC over stdio/WS| WSL
    macOSApp -->|WebSocket| WSL
    iOSApp -->|WebSocket| WSL
    AndroidApp -->|WebSocket| WSL
    WebUI -->|WebSocket| WSL
    
    %% Gateway internal
    WSL --> ProtoL
    WSL --> AuthL
    WSL --> Methods
    Methods --> SessL
    Methods --> AgRT
    Methods --> Routing
    AgRT --> Routing
    AgRT --> AutoR
    AgRT --> ToolReg
    AgRT --> Mem
    Routing --> ChL
    Cron --> AgRT
    SessL --> ConfigL
    AuthL --> SecL
    
    %% Plugin
    AgRT --> PluginL
    Methods --> PluginL
    ChL --> PluginL
    PluginL --> SDK
    PluginL --> BundledP
    BundledP -.->|implements| SDK
    
    %% External
    BundledP --> ExtAPIs
    AgRT --> DockerS
    BundledP --> IMSGCli
    ToolReg --> MCPSrv
    
    %% Storage
    SessL --> SessFS
    AuthL --> AuthFS
    ConfigL --> Cfg
    PluginL --> PluginFS
    Mem --> MemFS
    Cron --> CronFS
    
    style GW fill:#FFE4B5
    style PluginRT fill:#E0FFFF
    style External fill:#D8BFD8
    style FS fill:#F0E68C
    style Apps fill:#FFB6C1
    style CLI fill:#FFB6C1
```

---

## 2. Gateway 내부 컴포넌트 (확대)

```mermaid
flowchart LR
    subgraph TransportL["🌐 Transport"]
        WSS["WSS<br/>(ws lib)"]
        TLS["TLS Runtime"]
        Pair["Pairing"]
    end
    
    subgraph ProtoL["📋 Protocol"]
        TyBox["TypeBox Schemas"]
        AJV["AJV Validators"]
        Frames["Request/Response/Event"]
        Ver["Protocol Version"]
    end
    
    subgraph AuthC["🔐 Auth"]
        AuthRes["authorizeGatewayConnect"]
        RateL["AuthRateLimiter<br/>10/min, 5min lockout"]
        DevTok["DeviceToken Verify"]
        Boot["BootstrapToken Verify"]
        TS["Tailscale Whois"]
        Proxy["Trusted Proxy Check"]
    end
    
    subgraph SessMgmt["📝 Session"]
        Lifecycle["session-lifecycle-state"]
        Store["SessionStore<br/>(JSON + JSONL)"]
        Envelope["session-envelope"]
        Compact["compaction"]
        Resolution["conversation-resolution"]
    end
    
    subgraph LaneSys["🚦 Lane / Queue"]
        Main["main lane"]
        Sub["subagent lane"]
        Cron["cron / cron-nested"]
        SessLane["session:{id}"]
        Q["enqueueCommandInLane"]
        Concurrent["maxConcurrent"]
    end
    
    subgraph Abort["⏹️ Abort"]
        AbortMap["chatAbortControllers Map"]
        AbortCtl["AbortController"]
        RunSeq["agentRunSeq"]
    end
    
    subgraph RPC["📞 RPC"]
        Reg["MethodRegistry"]
        Handlers["Handlers<br/>(agent.*, chat.*, channels.*)"]
        Approval["ExecApprovalManager"]
    end
    
    subgraph PluginIntegr["🔌 Plugin"]
        Pin["pinned PluginRegistry"]
        Discovery["plugin discovery"]
        Manifest["manifest LRU 512"]
        Lazy["createLazyRuntimeModule"]
    end
    
    subgraph Bcast["📡 Broadcast"]
        BcastFn["broadcast"]
        Connds["broadcastToConnIds"]
        Slow["dropIfSlow"]
        StateVer["stateVersion"]
    end
    
    TransportL --> ProtoL
    ProtoL --> AuthC
    AuthC --> SessMgmt
    SessMgmt --> LaneSys
    SessMgmt --> Abort
    SessMgmt --> RPC
    RPC --> PluginIntegr
    LaneSys --> Bcast
    Abort --> Bcast
    
    style TransportL fill:#FFE4E1
    style ProtoL fill:#E0FFFF
    style AuthC fill:#FFFACD
    style SessMgmt fill:#F0FFF0
    style LaneSys fill:#FFF0F5
    style Abort fill:#FFB6C1
    style RPC fill:#FFEFD5
    style PluginIntegr fill:#E6E6FA
    style Bcast fill:#FFE4B5
```

---

## 3. Agent Runtime 내부 컴포넌트

```mermaid
flowchart TB
    Inbound["Inbound Message<br/>from Channel"]
    
    subgraph AgRT["Agent Runtime (src/agents/)"]
        Runner["pi-embedded-runner<br/>runEmbeddedPiAgent()"]
        PlanBuild["runtime-plan/build.ts<br/>buildAgentRuntimePlan()"]
        
        subgraph Plan["AgentRuntimePlan"]
            ResRef["resolvedRef"]
            ProvHandle["providerRuntimeHandle"]
            AuthPlan["AuthPlan"]
            PromptPlan["PromptPlan"]
            ToolsPlan["ToolsPlan"]
            TranscriptPolicy["TranscriptPolicy<br/>(lazy getter)"]
            Delivery["DeliveryPlan"]
            Outcome["OutcomePlan"]
            Transport["TransportPlan<br/>(lazy getter)"]
        end
        
        Harness["harness/<br/>모델별 출력 정규화"]
        Subagent["subagent-spawn.ts<br/>spawnSubagentDirect()"]
        Sandbox["sandbox/<br/>(docker, ssh, direct)"]
        ToolPolicy["tool-policy"]
        AuthProfiles["auth-profiles/<br/>69 files"]
        Tools["tools/<br/>65+ tools"]
        Compact["compaction.ts<br/>(BASE_CHUNK_RATIO=0.4)"]
        FailoverErr["failover-error.ts"]
        FailoverPol["failover-policy.ts"]
        AutoR["auto-reply/<br/>get-reply, directives"]
        Memory["memory-search.ts<br/>(circuit breaker)"]
    end
    
    subgraph Provider["Provider Plugin<br/>(extensions/anthropic/)"]
        Stream["stream-wrappers"]
        Replay["replay-policy"]
        Discovery["provider-discovery"]
        PiAi["@mariozechner/pi-ai"]
    end
    
    subgraph Channel["Channel Plugin<br/>(extensions/telegram/)"]
        Outbound["outbound-adapter"]
        Send["send.ts"]
        Probe["probe"]
    end
    
    Inbound --> Runner
    Runner --> PlanBuild
    PlanBuild --> Plan
    Runner --> Harness
    Runner --> Memory
    Runner --> Tools
    Runner --> Subagent
    Subagent --> Sandbox
    Tools --> ToolPolicy
    Plan --> AuthProfiles
    Runner --> Compact
    Compact --> FailoverErr
    FailoverErr --> FailoverPol
    Runner --> AutoR
    Plan --> Provider
    Provider --> PiAi
    Runner --> Channel
    Channel --> Outbound
    
    style AgRT fill:#FFE4B5
    style Plan fill:#FFFACD
    style Provider fill:#DDA0DD
    style Channel fill:#87CEEB
```

---

## 4. Plugin System 컴포넌트

```mermaid
flowchart TB
    subgraph PluginCore["Plugin Core (src/plugins/)"]
        Disc["discovery.ts<br/>(보안 검증 포함)"]
        Manifest["manifest.ts<br/>(JSON5, LRU 512)"]
        Eligi["manifest-contract-eligibility"]
        Loader["loader.ts"]
        Reg["registry.ts<br/>(PluginRegistry)"]
        ActiveReg["active-runtime-registry"]
        ActPlanner["activation-planner"]
        Lazy["lazy-runtime-shared"]
    end
    
    subgraph SDK["Plugin SDK<br/>(packages/plugin-sdk/, src/plugin-sdk/)"]
        APIts["api.ts pattern"]
        RuntimeAPIts["runtime-api.ts pattern"]
        
        subgraph SDKExports["50+ Subpath Exports"]
            E1["plugin-entry"]
            E2["provider-entry"]
            E3["channel-entry-contract"]
            E4["channel-core"]
            E5["channel-message"]
            E6["provider-auth"]
            E7["provider-model-shared"]
            E8["provider-usage"]
            EN["..."]
        end
    end
    
    subgraph BundledP["Bundled Plugins (extensions/*)"]
        subgraph Channels["Channel"]
            TG["telegram<br/>(grammy 1.42)"]
            DC["discord<br/>(discord-api-types)"]
            SL["slack"]
            IM["imessage<br/>(imsg CLI)"]
            EtcCh["...20+"]
        end
        
        subgraph Providers["Provider"]
            AN["anthropic<br/>(pi-ai)"]
            OA["openai"]
            GG["google"]
            EtcPv["..."]
        end
        
        subgraph MemP["Memory (slot)"]
            ActMem["active-memory"]
            LancePM["memory-lancedb"]
            WikiM["memory-wiki"]
        end
        
        subgraph Toolp["Tools"]
            Canvas["canvas<br/>(Lit + a2ui)"]
            Browser["browser"]
            MCP_ext["mcp"]
        end
    end
    
    subgraph FSPlugin["FS"]
        Manifests["openclaw.plugin.json"]
        InstalledIdx["~/.openclaw/plugins/installed-index.json"]
        StateDB["~/.openclaw/plugin-state/state.sqlite"]
    end
    
    Disc --> Manifest
    Manifest --> Eligi
    Eligi --> ActPlanner
    ActPlanner --> Loader
    Loader --> Reg
    Reg --> ActiveReg
    Loader --> Lazy
    
    BundledP -->|imports from| SDK
    BundledP --> Manifests
    PluginCore --> InstalledIdx
    BundledP --> StateDB
    
    SDK --> APIts
    SDK --> RuntimeAPIts
    SDK --> SDKExports
    
    style PluginCore fill:#E0FFFF
    style SDK fill:#FFE4B5
    style BundledP fill:#F0FFF0
    style FSPlugin fill:#F0E68C
```

---

## 5. Channels 컴포넌트

```mermaid
flowchart TB
    subgraph ChCore["Channel Core (src/channels/)"]
        ChTypes["plugins/types.{adapters,core,plugin}.ts"]
        ChReg["registry.ts"]
        TargetParse["plugins/target-parsing.ts"]
        ConvRes["conversation-resolution.ts"]
        SessEnv["session-envelope.ts"]
        ChIds["ids.ts"]
        
        subgraph MsgL["message/"]
            Mtypes["types.ts<br/>MessageReceipt, MessageSendContext"]
            Send["send.ts"]
            Recv["receive.ts"]
            Live["live.ts<br/>(draft preview)"]
            Caps["capabilities.ts"]
            ReplyPipe["reply-pipeline.ts"]
            Receipt["receipt.ts"]
        end
        
        subgraph TurnL["turn/"]
            Durable["durable-delivery.ts"]
        end
    end
    
    subgraph Adapters["Channel Adapter (각 플러그인)"]
        Outbound["ChannelOutboundAdapter<br/>{sendText, sendMedia, sendPayload}"]
        Inbound["ChannelInboundAdapter<br/>{onMessage}"]
        ChMsg["ChannelMessageAdapter<br/>(Outbound + receive + live)"]
    end
    
    subgraph Specific["채널 구현 (extensions/)"]
        TGImpl["telegram/<br/>- bot-handlers.runtime.ts<br/>- monitor.ts (polling/webhook)<br/>- send.ts (4096 chunk)<br/>- error-policy (4h cooldown)"]
        DCImpl["discord/<br/>- channel.ts (live caps)<br/>- probe.ts (intent check)<br/>- voice (@discordjs/voice)"]
        IMImpl["imessage/<br/>- client.ts (imsg JSON-RPC)<br/>- spawn(cliPath, ['rpc'])"]
    end
    
    ChCore --> Adapters
    Adapters -.->|implements| Specific
    
    style ChCore fill:#87CEEB
    style Adapters fill:#FFE4B5
    style Specific fill:#F0FFF0
```

---

## 6. Storage Components

```mermaid
flowchart TB
    subgraph Config["Config (src/config/)"]
        Paths["paths.ts<br/>resolveStateDir, resolveConfigPath"]
        IO["io.ts<br/>(JSON5, $include, env subst)"]
        Types["types.openclaw.ts"]
        Secrets["secrets/<br/>SecretRef resolver"]
    end
    
    subgraph SessConfig["Sessions (src/config/sessions/)"]
        SessPaths["paths.ts<br/>resolveDefaultSessionStorePath"]
        StoreL["store.ts<br/>load/save/update"]
        StoreCache["store-cache.ts<br/>(serialized + object)"]
        StoreLock["store-writer.ts<br/>file lock"]
        TranscriptL["transcript.ts<br/>JSONL append"]
    end
    
    subgraph AuthSt["Auth (src/agents/auth-profiles/)"]
        AuthPaths["paths.ts"]
        AuthTypes["types.ts<br/>{ApiKey,Token,OAuth}Credential"]
        OAuthRT["oauth.ts<br/>refresh + lock"]
        ProfState["AuthProfileState<br/>order, lastGood, usageStats"]
    end
    
    subgraph PluginSt["Plugin State (src/plugin-state/)"]
        PSStore["plugin-state-store.sqlite.ts<br/>SQLite, 64KB max value"]
    end
    
    subgraph CronSt["Cron (src/cron/)"]
        CronStore["store.ts<br/>jobs.json + jobs-state.json"]
        CronTypes["job types"]
    end
    
    subgraph MemSt["Memory (src/memory/)"]
        MemFiles["root-memory-files.ts"]
        MemSearch["memory-search.ts"]
    end
    
    Config -.->|reads| FS1[(~/.openclaw)]
    SessConfig -.->|RW| FS1
    AuthSt -.->|RW| FS1
    PluginSt -.->|RW| FS1
    CronSt -.->|RW| FS1
    MemSt -.->|RW| FS1
    
    style Config fill:#FFE4B5
    style SessConfig fill:#F0FFF0
    style AuthSt fill:#FFFACD
    style PluginSt fill:#E0FFFF
    style CronSt fill:#FFE4E1
    style MemSt fill:#F0E68C
```

---

## 7. 의존성 방향 규칙

UML 컴포넌트 다이어그램의 핵심 원칙은 **의존성이 한 방향**이라는 것. OpenClaw는 다음 규칙 강제:

```mermaid
flowchart TB
    AppsCli["Apps / CLI"] -->|WS only| Gateway
    Gateway -->|via SDK| Plugins
    Plugins -->|via SDK| Plugins
    Plugins -.->|❌ FORBIDDEN| Gateway
    Plugins -.->|❌ FORBIDDEN| OtherPlugins["Other Plugins<br/>(direct import)"]
    
    style AppsCli fill:#FFB6C1
    style Gateway fill:#FFE4B5
    style Plugins fill:#E0FFFF
```

`pnpm check:architecture` + `pnpm check:import-cycles` + madge로 강제 검증.

### 금지된 import (실패 시 빌드 실패)

```
extensions/foo/src/* 
  → import "src/internals/*"           ❌
  → import "../../bar/src/*"           ❌  
  → import "../../../src/something"    ❌
  → import "openclaw/plugin-sdk/*"     ✅

src/* 
  → import "extensions/foo/src/*"      ❌
  → import "extensions/foo/api"         ✅
  → import "extensions/foo/runtime-api" ✅ (lazy)
```
