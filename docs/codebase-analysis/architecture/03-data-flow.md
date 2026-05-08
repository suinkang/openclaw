# 03. Data Flow & Sequence Diagrams

OpenClaw의 핵심 데이터 흐름을 UML Sequence Diagram으로 표현.

## 1. 메시지 처리 End-to-End (Telegram 예시)

```mermaid
sequenceDiagram
    actor User as 👤 User
    participant TG as 📱 Telegram Server
    participant TGPlugin as Telegram Plugin<br/>(grammy)
    participant Gateway as Gateway<br/>(WS RPC)
    participant SessLane as session:{id}<br/>Lane Queue
    participant Runner as runEmbeddedPiAgent
    participant PlanBuild as buildAgentRuntimePlan
    participant ActMem as Active Memory
    participant Provider as Anthropic Plugin<br/>(pi-ai)
    participant LLM as 🧠 Anthropic API
    participant Tools as Tool Registry
    participant Outbound as Telegram Outbound
    
    User->>TG: "What's on my calendar?"
    TG-->>TGPlugin: Update via long-polling (30s)
    TGPlugin->>TGPlugin: bot-handlers.runtime.ts<br/>parse message
    TGPlugin->>TGPlugin: createTelegramUpdateTracker<br/>track update_id
    TGPlugin->>Gateway: chat.send RPC
    
    Gateway->>Gateway: validateRequestFrame (AJV)
    Gateway->>Gateway: ensureAuthorized (deviceToken)
    Gateway->>Gateway: createDedupeCache check
    Gateway->>SessLane: enqueueCommandInLane(<br/>"session:abc", task)
    
    Note over SessLane: maxConcurrent=1<br/>(세션당 순차 처리)
    
    SessLane->>Runner: runEmbeddedPiAgent(params)
    Runner->>Runner: resolveSessionLane()<br/>resolveGlobalLane()
    Runner->>Runner: ensureRuntimePluginsLoaded()
    Runner->>Runner: resolveModelAsync()
    Runner->>PlanBuild: buildAgentRuntimePlan(...)
    PlanBuild-->>Runner: AgentRuntimePlan<br/>(auth, prompt, tools, ...)
    
    Runner->>Runner: registerChatAbortController(runId)<br/>chatAbortControllers.set(runId, ctl)
    Runner->>Runner: resolveSystemPromptContribution()
    Runner->>Runner: tools.normalize(provider)
    
    Runner->>ActMem: queryActiveMemory(transcript)
    Note over ActMem: timeoutMs: 15000<br/>circuit breaker: 3 timeouts
    ActMem->>Provider: separate LLM call (smaller model)
    Provider-->>ActMem: relevant facts
    ActMem-->>Runner: facts injected
    
    Runner->>Provider: stream(messages, tools, ...)
    Provider->>LLM: HTTPS POST /v1/messages
    LLM-->>Provider: SSE stream events
    
    loop streaming
        Provider-->>Runner: text_delta / tool_use_delta
        Runner->>Gateway: broadcast("chat", {seq++, delta})
        Gateway->>TGPlugin: sessionMessage event
    end
    
    LLM-->>Provider: tool_use ended
    Provider-->>Runner: tool_call detected
    
    alt Tool: sessions_spawn
        Runner->>Runner: spawnSubagentDirect(...)<br/>depth/children check
    else Tool: web_search
        Runner->>Tools: execute web_search
    else Tool: bash
        Runner->>Tools: execute in sandbox<br/>(Docker/SSH/direct)
    end
    
    Tools-->>Runner: tool result
    Runner->>Provider: continue with tool_result
    Provider->>LLM: continue stream
    LLM-->>Provider: assistant final response
    Provider-->>Runner: stop event
    
    Runner->>Runner: delivery.resolveFollowupRoute()
    Runner->>Outbound: send(reply, target="telegram:12345")
    Outbound->>TG: bot.sendMessage(chatId, text)<br/>+ chunk if > 4096 chars
    TG-->>User: 답장 메시지
    
    Runner->>Gateway: broadcast("chat", {state:"done"})
    Runner->>Runner: chatAbortControllers.delete(runId)
    Runner-->>SessLane: EmbeddedPiRunResult
    SessLane->>SessLane: pump() → 다음 작업
```

---

## 2. WebSocket 핸드셰이크

```mermaid
sequenceDiagram
    participant Client as 클라이언트<br/>(macOS/iOS/CLI)
    participant WSS as ws.WebSocketServer
    participant MsgHdlr as message-handler<br/>(on demand)
    participant AuthMgr as authorizeGatewayConnect
    participant RateLim as AuthRateLimiter
    participant Pairing as Device Pairing
    
    Client->>WSS: WebSocket upgrade<br/>ws://localhost:18789
    WSS->>WSS: connection accepted<br/>connId = randomUUID()
    WSS->>WSS: setup preauth budget
    WSS->>Client: event "connect.challenge"<br/>{nonce, ts}
    WSS->>WSS: handshakeTimer (10s)
    
    Note over WSS: 메시지 핸들러는 lazy load<br/>로드 중 메시지는 큐 (max 16)
    
    Client->>WSS: req: connect<br/>{params: ConnectParams}
    WSS->>MsgHdlr: parse and route
    
    MsgHdlr->>MsgHdlr: validateRequestFrame (AJV)
    MsgHdlr->>MsgHdlr: validateConnectParams (AJV)
    
    MsgHdlr->>MsgHdlr: 프로토콜 버전 협상<br/>minProtocol ≤ PROTOCOL_VERSION ≤ maxProtocol
    
    alt 버전 불일치
        MsgHdlr-->>Client: error 1002<br/>protocol mismatch
        WSS->>WSS: close
    end
    
    MsgHdlr->>RateLim: check(ip, scope)
    
    alt Rate limited
        RateLim-->>MsgHdlr: {allowed: false, retryAfterMs}
        MsgHdlr-->>Client: 4001 Rate limited
        WSS->>WSS: close
    end
    
    MsgHdlr->>AuthMgr: authorizeGatewayConnect(params)
    
    alt 인증 모드별
        AuthMgr->>AuthMgr: token / password / device-token / bootstrap-token / tailscale / trusted-proxy
        AuthMgr->>AuthMgr: safeEqualSecret (constant time)
    end
    
    alt 인증 실패
        AuthMgr-->>MsgHdlr: {ok: false, reason}
        RateLim->>RateLim: recordFailure(ip)
        MsgHdlr-->>Client: 1008 Unauthorized
        WSS->>WSS: close
    end
    
    AuthMgr-->>MsgHdlr: {ok: true, method, user}
    
    MsgHdlr->>Pairing: reconcileNodePairingOnConnect()
    Pairing-->>MsgHdlr: paired device info
    
    MsgHdlr->>WSS: setClient(GatewayWsClient)<br/>clients.add()
    MsgHdlr->>WSS: clear handshakeTimer<br/>release preauth budget
    MsgHdlr->>WSS: setup ping (25s interval)
    
    MsgHdlr-->>Client: res hello-ok<br/>{protocolVersion, server, features,<br/>snapshot, auth, policy}
    
    Note over Client,WSS: ✅ Authenticated
    
    loop 일반 RPC
        Client->>WSS: req: chat.send / agent.* / etc
        WSS->>MsgHdlr: route to handler
        MsgHdlr-->>Client: res / events
    end
```

---

## 3. Subagent Spawn 시퀀스

```mermaid
sequenceDiagram
    participant ParentRunner as Parent Agent<br/>runEmbeddedPiAgent
    participant SpawnFn as spawnSubagentDirect
    participant SessStore as Session Store
    participant SubReg as Subagent Registry
    participant Gateway as Gateway RPC
    participant ChildLane as subagent Lane
    participant ChildRunner as Child Agent<br/>runEmbeddedPiAgent
    
    ParentRunner->>ParentRunner: LLM tool_use:<br/>sessions_spawn
    ParentRunner->>SpawnFn: spawnSubagentDirect(params, ctx)
    
    SpawnFn->>SessStore: getSubagentDepthFromSessionStore<br/>(parentKey)
    SessStore-->>SpawnFn: callerDepth
    
    alt callerDepth >= MAX_SPAWN_DEPTH (1)
        SpawnFn-->>ParentRunner: {status: "forbidden",<br/>error: "depth exceeded"}
    end
    
    SpawnFn->>SubReg: countActiveRunsForSession<br/>(parentKey)
    SubReg-->>SpawnFn: activeChildren
    
    alt activeChildren >= MAX_CHILDREN (5)
        SpawnFn-->>ParentRunner: {status: "forbidden",<br/>error: "max children exceeded"}
    end
    
    SpawnFn->>SpawnFn: childSessionKey =<br/>"agent:{targetAgentId}:subagent:{uuid}"
    
    SpawnFn->>SpawnFn: resolveSubagentContextMode<br/>(full|partial|empty)
    SpawnFn->>SpawnFn: resolveSubagentCapabilities<br/>(role: leaf if depth >= MAX)
    
    SpawnFn->>SessStore: write child session<br/>{spawnDepth, role, controlScope, ...}
    
    SpawnFn->>Gateway: callSubagentGateway<br/>method: "sessions.patch" / "agent"<br/>scopes: [ADMIN_SCOPE]
    
    Gateway->>ChildLane: enqueueCommandInLane<br/>("subagent", task)
    Note over ChildLane: maxConcurrent=8
    
    ChildLane->>ChildRunner: runEmbeddedPiAgent(childParams)
    
    Note over ChildRunner: 자식이 또 spawn하려 하면<br/>role=leaf → 거부됨
    
    ChildRunner->>ChildRunner: 처리 후 결과 반환
    ChildRunner-->>ChildLane: result
    ChildLane-->>Gateway: gatewayCfg
    Gateway-->>SpawnFn: gatewayCfg
    
    SpawnFn->>SubReg: registerSubagentRun<br/>(parentKey, childKey, runId)
    
    SpawnFn-->>ParentRunner: {status: "accepted",<br/>childSessionKey, runId, modelApplied}
    
    ParentRunner->>ParentRunner: tool_result with childSessionKey<br/>→ next LLM turn
```

---

## 4. OAuth Token Refresh

```mermaid
sequenceDiagram
    participant Code as Provider Plugin<br/>or Auth Resolver
    participant OAuth as oauth.ts
    participant LockFile as ~/.openclaw/_auth_locks/<br/>{provider}-{profile}.lock
    participant ProvAPI as Provider OAuth Endpoint
    participant Store as auth-profiles.json
    
    Code->>OAuth: resolveAuth(profileId)
    OAuth->>OAuth: load credential
    
    alt expires - 60s <= now (즉시 만료 임박)
        OAuth->>LockFile: acquireFileLock<br/>(timeoutMs: 30000)
        
        Note over LockFile: 다른 프로세스가 락 보유 시 대기<br/>(같은 토큰 재사용 가능)
        
        LockFile-->>OAuth: lock acquired
        
        OAuth->>OAuth: re-read credential<br/>(다른 프로세스가 갱신했나?)
        
        alt 이미 갱신됨
            OAuth->>LockFile: release()
            OAuth-->>Code: refreshed token
        end
        
        OAuth->>ProvAPI: POST /token<br/>grant_type=refresh_token<br/>refresh_token, client_id
        
        alt refresh 실패
            ProvAPI-->>OAuth: 401/403
            OAuth-->>Code: throw FailoverError<br/>{reason: "auth"}
        end
        
        ProvAPI-->>OAuth: {access, refresh, expires_in}
        
        OAuth->>OAuth: credential.access = new<br/>credential.refresh = new<br/>credential.expires = now + expires_in
        
        OAuth->>Store: upsertAuthProfile(storePath, ...)
        
        OAuth->>LockFile: release()
    end
    
    OAuth-->>Code: valid credential
```

---

## 5. Active Memory Recall

```mermaid
sequenceDiagram
    participant Runner as Agent Runner
    participant Mem as Active Memory<br/>(extensions/active-memory)
    participant CB as Circuit Breaker State
    participant LLM as Memory LLM<br/>(separate model)
    participant Cache as activeRecallCache
    
    Runner->>Mem: recallForTurn(transcript, config)
    
    Mem->>CB: isCircuitBreakerOpen(key)
    
    alt circuit open AND now - lastTimeout < cooldownMs(60s)
        CB-->>Mem: open
        Mem-->>Runner: skip (return empty)
    end
    
    CB-->>Mem: closed
    
    Mem->>Cache: get(transcriptHash)
    
    alt cache hit
        Cache-->>Mem: cached result
        Mem-->>Runner: cached facts
    end
    
    Mem->>Mem: build memory query<br/>based on queryMode<br/>(message|recent|full)
    
    Mem->>Mem: build prompt based on<br/>promptStyle (6 options)
    
    Note over Mem: timeoutMs: 15000 (default)
    
    Mem->>LLM: stream call<br/>(separate from main)
    
    alt timeout
        LLM-->>Mem: AbortError
        Mem->>CB: increment consecutiveTimeouts
        
        alt consecutiveTimeouts >= 3
            CB->>CB: open circuit<br/>lastTimeoutAt = now
        end
        
        Mem-->>Runner: empty (graceful)
    end
    
    LLM-->>Mem: relevant facts
    Mem->>CB: reset (consecutiveTimeouts = 0)
    Mem->>Cache: store(transcriptHash, facts)
    Mem-->>Runner: facts<br/>→ injected into system prompt
```

---

## 6. Compaction Flow

```mermaid
sequenceDiagram
    participant Runner as Agent Runner
    participant Compact as compaction.ts
    participant LLM as Summary LLM
    participant Store as Session Store
    
    Runner->>Runner: estimateMessagesTokens
    
    alt tokens >= contextLimit * BASE_CHUNK_RATIO (0.4)
        Runner->>Compact: shouldRunMemoryFlush ?
        Compact-->>Runner: true
        
        Runner->>Runner: emit compaction-before event
        
        Runner->>LLM: generateSummary(messages,<br/>MERGE_SUMMARIES_INSTRUCTIONS)
        Note over LLM: SAFETY_MARGIN 1.2 (20% buffer)
        LLM-->>Runner: summary text
        
        Runner->>Compact: createSessionCompactionCheckpoint
        Compact->>Store: append checkpoint<br/>(keep latest 25)
        Compact->>Store: write postCompaction transcript ref
        
        Runner->>Runner: replace messages[k:] with<br/>{role: "assistant",<br/>content: "[Previous summary: ...]"}
        
        Runner->>Runner: emit compaction-after event<br/>{summaryTokens, kept, compacted}
        
        Note over Runner: 다음 turn에서 줄어든 컨텍스트로 진행
    else 임계값 미달
        Runner->>Runner: continue normal turn
    end
```

---

## 7. Session Lifecycle Event 처리

```mermaid
sequenceDiagram
    participant Source as Source<br/>(Runner / Channel)
    participant Lifecycle as session-lifecycle-state
    participant Resolver as resolvePhase<br/>resolveTerminalStatus
    participant Persist as persistGatewayLifecycleEvent
    participant Store as Session Store
    participant Bcast as broadcast
    
    Source->>Lifecycle: deriveGatewayLifecycleSnapshot(event)
    
    Lifecycle->>Resolver: resolveLifecyclePhase(event)
    Resolver-->>Lifecycle: "start" | "end" | "error"
    
    alt phase = "start"
        Lifecycle->>Lifecycle: snapshot = {status: "running",<br/>startedAt: ts, abortedLastRun: false}
    else phase = "error"
        Lifecycle->>Lifecycle: snapshot.status = "failed"<br/>endedAt = ts
    else phase = "end"
        Resolver->>Resolver: stopReason == "aborted" ?
        alt aborted
            Lifecycle->>Lifecycle: status = "killed"<br/>abortedLastRun = true
        else aborted=true (timeout)
            Lifecycle->>Lifecycle: status = "timeout"
        else 정상 종료
            Lifecycle->>Lifecycle: status = "done"
            Lifecycle->>Lifecycle: runtimeMs = endedAt - startedAt
        end
    end
    
    Lifecycle-->>Source: snapshot
    
    Source->>Persist: persistGatewaySessionLifecycleEvent
    Persist->>Store: updateSessionStoreEntry(<br/>storePath, sessionKey, patch)
    Store-->>Persist: ok
    
    Persist->>Bcast: broadcast("session", snapshot)
    Bcast-->>Source: 모든 클라이언트에 전파
```

---

## 8. Plugin Discovery & Activation

```mermaid
sequenceDiagram
    participant Boot as Gateway Startup
    participant Disc as discovery.ts
    participant FS as ~/extensions/, ~/.openclaw/plugins
    participant Manifest as manifest.ts
    participant Cache as Manifest LRU 512
    participant Eligibility as manifest-contract-eligibility
    participant Planner as activation-planner
    participant Loader as loader.ts
    participant Registry as PluginRegistry
    
    Boot->>Disc: scanForPlugins()
    Disc->>FS: readdir(extensions/)
    FS-->>Disc: plugin dirs
    
    loop 각 플러그인
        Disc->>Disc: checkSourceEscapesRoot
        Disc->>Disc: 파일 권한 0o002 검사
        Disc->>Disc: POSIX UID 검증<br/>(non-bundled만)
        Disc->>Disc: 심볼릭/하드링크 거부
        
        alt 보안 검증 실패
            Disc->>Disc: skip plugin
        end
        
        Disc->>Manifest: loadPluginManifest(dir)
        Manifest->>Cache: get(dir)
        
        alt cache miss
            Manifest->>FS: read openclaw.plugin.json
            FS-->>Manifest: bytes
            Manifest->>Manifest: 256KB 제한 체크
            Manifest->>Manifest: JSON5 parse
            Manifest->>Manifest: 타입 체크
            Manifest->>Cache: store(dir, manifest)
        end
        
        Cache-->>Manifest: manifest
        Manifest-->>Disc: manifest
    end
    
    Disc-->>Boot: discovered plugins
    
    Boot->>Eligibility: check(plugin, env, config)
    Note over Eligibility: env vars, config, allowlist, denylist<br/>14가지 cause
    Eligibility-->>Boot: PluginActivationDecision
    
    Boot->>Planner: planActivation(decisions)
    Planner-->>Boot: ordered plugin list
    
    loop 순서대로
        Boot->>Loader: loadPlugin(manifest)
        Loader->>Loader: import api.ts (정적)
        Loader->>Registry: register
        
        Note over Loader: runtime-api.ts는 lazy<br/>(첫 호출 시만)
    end
    
    Boot->>Boot: pinActivePluginChannelRegistry
    Note over Boot: ✅ Plugins ready
```

---

## 9. Cron Job Execution

```mermaid
sequenceDiagram
    participant Sched as Cron Scheduler
    participant Store as cron/jobs-state.json
    participant Lane as cron / cron-nested Lane
    participant Runner as Agent Runner
    
    loop every check interval
        Sched->>Store: readJobs()
        
        loop each enabled job
            Sched->>Sched: nextRunAt(schedule, lastRunAtMs)
            
            alt now >= nextRunAtMs
                Sched->>Lane: enqueueCommandInLane<br/>(CommandLane.Cron, fn)
                Note over Lane: maxConcurrent=1 (cron lane)
                
                Lane->>Sched: dequeue & start
                
                alt sessionTarget = "main"
                    Sched->>Runner: run with main session
                else sessionTarget = "isolated"
                    Sched->>Runner: run isolated session<br/>(new sessionKey)
                end
                
                alt payload.kind = "systemEvent"
                    Runner->>Runner: emit system event
                else payload.kind = "agentTurn"
                    Runner->>Runner: enqueue user-like message<br/>(prompt, instructions)
                end
                
                Note over Runner: cron 내부 LLM 호출 시<br/>cron-nested lane 사용<br/>(데드락 방지)
                
                Runner-->>Sched: result
                
                Sched->>Store: writeState({<br/>nextRunAtMs: 다음 시각,<br/>lastRunAtMs: now,<br/>lastRunStatus: "completed"})
            end
        end
    end
```

---

## 10. Approval Request

```mermaid
sequenceDiagram
    participant Runner as Agent Runner
    participant ApprMgr as ExecApprovalManager
    participant Bcast as Broadcast
    participant UI as Client UI
    participant User as 👤 User
    
    Runner->>Runner: 위험 작업 요청<br/>(rm -rf, install, ...)
    Runner->>ApprMgr: create(request, timeoutMs, id?)
    ApprMgr-->>Runner: ExecApprovalRecord
    
    Runner->>ApprMgr: register(record, timeoutMs)
    ApprMgr->>ApprMgr: setTimeout (timeoutMs)<br/>설정
    ApprMgr-->>Runner: Promise<decision>
    
    Runner->>Bcast: broadcast(<br/>"approval", record,<br/>{dropIfSlow: true})
    Bcast->>UI: approval event
    UI->>User: "위험 작업 허용?"
    
    alt User approves
        User->>UI: 승인
        UI->>ApprMgr: resolve(recordId, "approve", user)
        ApprMgr->>ApprMgr: clearTimeout<br/>resolvedAtMs = now
        ApprMgr->>Runner: resolve("approve")
        ApprMgr->>Bcast: broadcast resolved event
        ApprMgr->>ApprMgr: scheduleResolvedEntryCleanup<br/>(15s grace)
        Runner->>Runner: 작업 실행
    else User denies
        User->>UI: 거부
        UI->>ApprMgr: resolve(recordId, "deny")
        ApprMgr->>Runner: resolve("deny")
        Runner->>Runner: error → 다음 turn
    else Timeout
        ApprMgr->>ApprMgr: timer fires → expire(recordId)
        ApprMgr->>Runner: resolve(null)
        Runner->>Runner: treated as deny
    end
```

---

## 11. Memory Slot 전환

```mermaid
sequenceDiagram
    participant CLI as openclaw CLI
    participant Config as Config Layer
    participant OldMem as Active Memory<br/>(currently active)
    participant NewMem as Memory LanceDB<br/>(target)
    participant State as plugin-state.sqlite
    
    CLI->>Config: setMemorySlot("memory-lancedb")
    Config->>Config: validate plugin exists
    
    Config->>OldMem: graceful shutdown signal
    OldMem->>OldMem: drain pending requests
    OldMem->>State: persist state (if any)
    OldMem-->>Config: shutdown complete
    
    Config->>Config: update memory.active = "memory-lancedb"
    Config->>Config: write openclaw.json
    
    Config->>NewMem: init slot
    NewMem->>NewMem: check platform support<br/>(M1 Mac x64 unsupported)
    
    alt unsupported platform
        NewMem-->>Config: error → revert
        Config->>OldMem: re-init (rollback)
    end
    
    NewMem->>State: load previous state (if compatible)
    NewMem-->>Config: ready
    
    Config-->>CLI: ✅ memory slot changed
```

---

## 12. CLI command 흐름

```mermaid
sequenceDiagram
    participant Term as Terminal
    participant CLIBin as openclaw binary
    participant ProgMgr as src/cli/progress.ts
    participant Gateway as Gateway (locally)
    participant LLM as LLM API
    
    Term->>CLIBin: openclaw agent --message "..."
    CLIBin->>CLIBin: parse args, load config
    
    CLIBin->>Gateway: start (if not running) or connect
    Note over Gateway: gateway daemon이 별도 프로세스
    
    Gateway-->>CLIBin: WS connected, hello-ok
    
    CLIBin->>Gateway: chat.send RPC
    Gateway->>LLM: stream call
    
    loop streaming
        LLM-->>Gateway: deltas
        Gateway-->>CLIBin: chat event
        CLIBin->>ProgMgr: update progress display
        ProgMgr->>Term: render terminal output
    end
    
    Gateway-->>CLIBin: chat done event
    CLIBin->>CLIBin: print final result
    CLIBin->>Gateway: disconnect (or stay)
    CLIBin-->>Term: exit
```

---

## 데이터 흐름 요약

| 흐름 | 시작점 | 주요 경유 | 종착점 | 핵심 변환 |
|------|-------|----------|--------|----------|
| 인바운드 메시지 | 외부 채널 | Channel Plugin → Gateway → Lane → Runner | LLM API | InboundMessage normalization |
| 응답 메시지 | LLM stream | Runner → Outbound Adapter | 외부 채널 | text chunking, durability |
| Tool 호출 | LLM tool_use | Runner → Tool Registry / Subagent / Sandbox | Tool result back to LLM | schema normalization |
| 메모리 회상 | Runner | Active Memory → Separate LLM | System prompt | facts injection |
| OAuth refresh | Provider | OAuth → File lock → Provider API | Updated credentials | token rotation |
| WS handshake | Client | WSS → Auth → Pairing | Authenticated client | session establishment |
| Compaction | Runner | LLM (summary) → Store | Trimmed context | message replacement |
| Cron job | Scheduler | Lane → Runner | Side effects | scheduled trigger |
