# 06. Concurrency Model

OpenClaw의 동시성 / 큐잉 / 디스패처 / 백프레셔 모델.

## 1. 전체 동시성 토폴로지

```mermaid
flowchart TB
    subgraph External["외부 입력"]
        TG[Telegram Server]
        DC[Discord Gateway]
        Browser[웹 클라이언트]
        CLI[CLI]
    end
    
    subgraph ChIn["Channel Plugin (Inbound)"]
        TGRunner["grammy Runner<br/>sink.concurrency=4"]
        DCWS["Discord WS<br/>(events)"]
    end
    
    subgraph WSGate["Gateway WebSocket"]
        WSConn["GatewayWsClient<br/>per connection"]
        RateLim["AuthRateLimiter<br/>10/min, 5min lockout"]
        BcastFn["broadcast<br/>(per client buffered check)"]
    end
    
    subgraph Lanes["Command Lanes"]
        MainL["main lane<br/>maxConcurrent=4"]
        SubL["subagent lane<br/>maxConcurrent=8"]
        CronL["cron lane<br/>maxConcurrent=1"]
        CronNL["cron-nested lane<br/>maxConcurrent=1"]
        SessL1["session:abc<br/>maxConcurrent=1"]
        SessL2["session:def<br/>maxConcurrent=1"]
        SessLN["session:...<br/>maxConcurrent=1"]
    end
    
    subgraph Runner["Agent Runtime"]
        Run1["Runner #1"]
        Run2["Runner #2"]
        AbortMap["chatAbortControllers<br/>Map"]
        RunSeqMap["agentRunSeq<br/>Map"]
    end
    
    subgraph PluginLoad["Plugin Loading"]
        Pinned["pinned PluginRegistry"]
        ManCache["Manifest LRU 512"]
        LazyCache["lazy runtime modules"]
    end
    
    External --> ChIn
    ChIn --> WSGate
    Browser --> WSGate
    CLI --> WSGate
    WSGate --> Lanes
    Lanes --> Runner
    Runner --> PluginLoad
    
    style External fill:#FFB6C1
    style ChIn fill:#87CEEB
    style WSGate fill:#FFE4B5
    style Lanes fill:#FFFACD
    style Runner fill:#F0FFF0
    style PluginLoad fill:#E0FFFF
```

---

## 2. Lane 시스템 — 핵심 추상화

### 2.1 Lane Enum

`src/process/lanes.ts`:
```typescript
export const enum CommandLane {
  Main = "main",
  Cron = "cron",
  CronNested = "cron-nested",
  Subagent = "subagent",
  Nested = "nested",
}
```

### 2.2 Lane 종류와 maxConcurrent

```mermaid
flowchart LR
    subgraph LaneKinds["Lane 종류"]
        Main["main<br/>(global)<br/>maxConcurrent=4"]
        Cron["cron<br/>(global)<br/>maxConcurrent=1"]
        CronN["cron-nested<br/>(global)<br/>maxConcurrent=1"]
        Sub["subagent<br/>(global)<br/>maxConcurrent=8"]
        Nested["nested<br/>(global)"]
        SessId["session:{id}<br/>(per session)<br/>maxConcurrent=1"]
    end
    
    SessId -.->|"resolveSessionLane('abc')"| LaneA[session:abc]
    SessId -.->|"resolveSessionLane('def')"| LaneB[session:def]
    
    style Main fill:#FFE4B5
    style Cron fill:#FFB6C1
    style Sub fill:#87CEEB
    style SessId fill:#90EE90
```

### 2.3 Lane 결정 로직

`src/agents/pi-embedded-runner/lanes.ts`:

```typescript
export function resolveSessionLane(key: string) {
  const cleaned = key.trim() || CommandLane.Main;
  return cleaned.startsWith("session:") ? cleaned : `session:${cleaned}`;
}

export function resolveGlobalLane(lane?: string) {
  const cleaned = lane?.trim();
  // 크론 작업은 cron-nested로 리매핑 (데드락 방지)
  if (cleaned === CommandLane.Cron) {
    return CommandLane.CronNested;
  }
  return cleaned ? cleaned : CommandLane.Main;
}
```

### 2.4 Cron 데드락 회피

```mermaid
sequenceDiagram
    participant Sched as Cron Scheduler
    participant CronL as cron lane (max=1)
    participant CronNL as cron-nested lane
    participant LLM
    
    Sched->>CronL: enqueue cron job
    CronL->>CronL: pump → start running
    
    Note over CronL: 활성 1/1 (full)
    
    CronL->>LLM: agent turn (LLM 호출)
    LLM-->>CronL: tool_use (sub task)
    
    Note over CronL: cron lane은 가득 참
    
    CronL->>CronNL: enqueue sub task<br/>(resolveGlobalLane mapping)
    Note over CronL,CronNL: 데드락 회피!<br/>서로 다른 lane이라 처리 가능
    
    CronNL->>CronNL: pump → start
    CronNL-->>LLM: result
    LLM-->>CronL: continue
    CronL-->>Sched: complete
```

---

## 3. Command Queue 동작

### 3.1 Lane State 구조

`src/process/command-queue.ts`:

```typescript
type LaneState = {
  lane: string;
  queue: QueueEntry[];
  activeTaskIds: Set<number>;
  maxConcurrent: number;
  draining: boolean;
  generation: number;          // lane 초기화 감지
};

type QueueEntry = {
  task: () => Promise<unknown>;
  resolve: (v: unknown) => void;
  reject: (e: unknown) => void;
  opts?: EnqueueOpts;
};
```

### 3.2 Drain 알고리즘

```mermaid
flowchart TD
    Start[enqueueCommandInLane] --> Push[queue.push]
    Push --> Drain{drain in progress?}
    Drain -->|no| StartDrain[draining = true]
    Drain -->|yes| Wait[기존 drain이 처리]
    
    StartDrain --> Pump[pump function]
    Pump --> Check{queue.length > 0<br/>AND<br/>activeTaskIds.size < maxConcurrent}
    
    Check -->|yes| Dequeue[entry = queue.shift]
    Dequeue --> AssignId[taskId = nextTaskId++]
    AssignId --> AddActive[activeTaskIds.add]
    AddActive --> Async[비동기 실행 시작]
    Async --> Pump
    
    Check -->|no| EndPump[draining = false]
    EndPump --> Wait
    
    Async --> Done{작업 완료}
    Done -->|성공| Complete[completeTask]
    Done -->|실패| CompleteFail[completeTask + reject]
    Complete --> RemoveActive[activeTaskIds.delete]
    CompleteFail --> RemoveActive
    RemoveActive --> Notify[notifyActiveTaskWaiters]
    Notify --> Pump
    
    style Pump fill:#FFFACD
    style Async fill:#87CEEB
```

### 3.3 enqueue API

```typescript
export function enqueueCommandInLane<T>(
  lane: string,
  task: () => Promise<T>,
  opts?: {
    warnAfterMs?: number;       // 대기 시간 경고 임계값
    taskTimeoutMs?: number;     // 작업 타임아웃
    onWait?: (waitMs: number, queuedAhead: number) => void;
  },
): Promise<T>
```

---

## 4. 세션별 순차성 보장

### 4.1 왜 session lane?

- 한 사용자 메시지의 응답이 끝나기 전에 다음 메시지 도착
- 동시에 처리하면 race condition (transcript 충돌, 컨텍스트 오염)
- `session:{id}` lane은 maxConcurrent=1 → 자동 직렬화

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant Ch as Channel
    participant GW as Gateway
    participant SL as session:user_xyz<br/>maxConcurrent=1
    participant R as Runner
    
    U->>Ch: 메시지 1
    Ch->>GW: chat.send #1
    GW->>SL: enqueue task1
    SL->>R: start task1 (active 1/1)
    
    U->>Ch: 메시지 2 (즉시)
    Ch->>GW: chat.send #2
    GW->>SL: enqueue task2 (queued)
    
    U->>Ch: 메시지 3
    Ch->>GW: chat.send #3
    GW->>SL: enqueue task3 (queued)
    
    R-->>SL: task1 done
    SL->>R: start task2 (active 1/1)
    
    R-->>SL: task2 done
    SL->>R: start task3
    
    Note over SL: 항상 1개씩만 처리<br/>순서 보장
```

### 4.2 AgentRunSeq — broadcast 순서

`agentRunSeq: Map<string, number>`는 **broadcast 이벤트의 순서**를 추적:

```typescript
// chat-abort.ts:139 (개념)
broadcast("chat", {
  runId,
  sessionKey,
  seq: (ops.agentRunSeq.get(runId) ?? 0) + 1,
  state: "running",
  // ...
});
```

→ 클라이언트는 `seq` 단조증가로 stale event 무시 가능.

---

## 5. AbortController 시스템

### 5.1 등록과 추적

`src/gateway/chat-abort.ts:70-108`:

```typescript
export type ChatAbortControllerEntry = {
  controller: AbortController;
  sessionId: string;
  sessionKey: string;
  startedAtMs: number;
  expiresAtMs: number;
  ownerConnId?: string;
  ownerDeviceId?: string;
  kind?: "chat-send" | "agent";
};

// chatAbortControllers: Map<runId, ChatAbortControllerEntry>
```

### 5.2 Abort 흐름

```mermaid
sequenceDiagram
    participant Client
    participant GW as Gateway
    participant Map as chatAbortControllers Map
    participant Runner as Agent Runner
    participant LLM
    participant Tools
    
    Client->>GW: chat.send (runId=X)
    GW->>Map: registerChatAbortController(X)
    Map->>Map: AbortController 생성
    GW->>Runner: run with signal
    
    Runner->>LLM: fetch(url, { signal })
    Runner->>Tools: execute(input, { signal })
    
    Note over Client,Map: 사용자가 중단 클릭
    Client->>GW: chat.abort (runId=X)
    GW->>Map: abortChatRunById(X)
    
    Map->>Map: entry.controller.abort()
    Map->>Map: chatAbortControllers.delete(X)
    Map->>Map: chatRunBuffers.delete(X)
    Map->>Map: chatDeltaSentAt.delete(X)
    
    LLM-->>Runner: AbortError
    Tools-->>Runner: AbortError
    Runner-->>GW: throw
    
    GW->>Client: broadcast {<br/>state: "aborted", partialText<br/>}
```

### 5.3 Cross-session 보호

```typescript
// chat-abort.ts (개념)
export function abortChatRunById(ops, params) {
  const active = ops.chatAbortControllers.get(runId);
  if (!active) return { aborted: false };
  if (active.sessionKey !== sessionKey) {
    return { aborted: false };  // 다른 세션의 abort 거부
  }
  active.controller.abort();
  // ...
}
```

---

## 6. Subagent 동시성 제한

### 6.1 두 가지 제약

```mermaid
flowchart TB
    Spawn[spawnSubagentDirect] --> Check1{depth >= MAX_SPAWN_DEPTH?}
    Check1 -->|yes (1)| Reject1[forbidden: depth exceeded]
    Check1 -->|no| Check2{children >= MAX_CHILDREN?}
    Check2 -->|yes (5)| Reject2[forbidden: max children]
    Check2 -->|no| Allowed[허용]
    
    Allowed --> CalcRole[role calculation]
    CalcRole --> Leaf{depth >= MAX-1?}
    Leaf -->|yes| LeafRole[role=leaf<br/>controlScope=none]
    Leaf -->|no| Orchestrator[role=orchestrator<br/>controlScope=children]
    
    LeafRole --> Done
    Orchestrator --> Done
    
    style Reject1 fill:#FFB6C1
    style Reject2 fill:#FFB6C1
    style Done fill:#90EE90
```

### 6.2 트리 구조 제한

```mermaid
flowchart TD
    Root["Root Agent<br/>(depth 0, role=main)"]
    
    Root --> C1["Child 1<br/>(depth 1, role=leaf)"]
    Root --> C2["Child 2<br/>(depth 1, role=leaf)"]
    Root --> C3["Child 3<br/>(depth 1, role=leaf)"]
    Root --> C4["Child 4<br/>(depth 1, role=leaf)"]
    Root --> C5["Child 5<br/>(depth 1, role=leaf)"]
    
    Root -.->|❌ children=5 이미 활성<br/>거부| Excess["Child 6"]
    
    C1 -.->|❌ leaf 역할은<br/>spawn 불가| GC["Grandchild"]
    
    style Excess fill:#FFB6C1
    style GC fill:#FFB6C1
```

기본값:
- `DEFAULT_SUBAGENT_MAX_CHILDREN_PER_AGENT = 5`
- `DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH = 1`

→ 1단계 분기, 5명까지. 무한 재귀 방어.

### 6.3 Subagent Registry 쿼리

```typescript
import {
  countActiveDescendantRunsFromRuns,
  countActiveRunsForSessionFromRuns,
  isSubagentSessionRunActiveFromRuns,
  // ...
} from "./subagent-registry-queries.js";
```

- `countActiveRunsForSession(parentSessionKey)` — 직계 자식 활성 수
- `countActiveDescendantRunsFromRuns(parentSessionKey)` — 모든 후손 활성 수
- `isSubagentSessionRunActiveFromRuns(runId)` — runId 활성 여부

---

## 7. Channel 동시성

### 7.1 Telegram (grammy)

`extensions/telegram/src/monitor.ts:28-48`:
```typescript
export function createTelegramRunnerOptions(cfg: OpenClawConfig) {
  return {
    sink: {
      concurrency: resolveAgentMaxConcurrent(cfg),  // = 4
    },
    runner: {
      fetch: { timeout: 30 },
      maxRetryTime: 60 * 60 * 1000,    // 1h
      retryInterval: "exponential",
    },
  };
}
```

→ Telegram 봇은 max 4개 메시지를 동시 처리 (gateway concurrency 상속).

### 7.2 인바운드 흐름

```mermaid
flowchart LR
    TG[Telegram Server] -->|long polling| Bot[grammy Bot]
    Bot --> Sink["sink<br/>(concurrency=4)"]
    Sink -->|max 4 동시| Handlers[Update Handlers]
    Handlers --> RPC[chat.send / agent.run RPC]
    RPC --> Lane["session:{id} lane<br/>(maxConcurrent=1)"]
    Lane --> Runner[Agent Runner]
    
    style Sink fill:#FFFACD
    style Lane fill:#FFE4B5
```

여러 사용자가 동시 메시지 → grammy sink에서 4개씩 받음 → 각 사용자별 session lane으로 분배 → 사용자별 1:1 직렬 처리.

---

## 8. Plugin Registry Pinning

### 8.1 Pin/Release 메커니즘

`src/plugins/runtime.ts:229-263`:

```typescript
type RegistrySurfaceState = {
  registry: PluginRegistry | null;
  pinned: boolean;
  version: number;
};

const state = {
  activeRegistry: null,
  activeVersion: 0,
  httpRoute: { registry: null, pinned: false, version: 0 },
  channel: { registry: null, pinned: false, version: 0 },
};

export function pinActivePluginChannelRegistry(registry: PluginRegistry) {
  installSurfaceRegistry(state.channel, registry, true);
}

export function releasePinnedPluginChannelRegistry(registry?: PluginRegistry) {
  if (registry && state.channel.registry !== registry) {
    return;
  }
  installSurfaceRegistry(state.channel, state.activeRegistry, false);
}
```

### 8.2 왜 pinning?

```mermaid
sequenceDiagram
    participant Boot as Gateway Boot
    participant Pin as PinnedRegistry
    participant Runtime as Runtime Operations
    participant SchemaRead as ConfigSchema Read
    
    Boot->>Pin: pinActivePluginChannelRegistry(registry)
    Note over Pin: ✅ Registry 잠김
    
    par 동시 작업
        Runtime->>Pin: getActivePluginChannelRegistry
        Pin-->>Runtime: pinned registry (안전)
    and
        SchemaRead->>SchemaRead: load schema (가벼운 작업)
        Note over SchemaRead: ✅ Pin과 무관<br/>channel registry 영향 없음
    end
    
    Boot->>Pin: releasePinnedPluginChannelRegistry
    Pin->>Pin: state.channel.registry = activeRegistry
```

→ **config schema 읽기 같은 가벼운 작업이 channel plugin을 evict하지 않도록 보장.**

---

## 9. Backpressure & Slow Client

### 9.1 Buffered Amount 모니터링

`src/gateway/server-broadcast.ts:87-181`:

```typescript
const slow = c.socket.bufferedAmount > MAX_BUFFERED_BYTES;

if (slow && opts?.dropIfSlow) {
  // 메시지 드롭, 연결 유지
  continue;
}

if (slow) {
  // 연결 강제 종료
  c.socket.close(1008, "slow consumer");
  continue;
}

// 정상: send
c.socket.send(frame);
```

### 9.2 두 가지 정책

```mermaid
flowchart TB
    Broadcast[broadcast event] --> Check{각 client.bufferedAmount<br/>> MAX_BUFFERED_BYTES?}
    
    Check -->|no| Send[정상 송신]
    Check -->|yes| Policy{dropIfSlow?}
    
    Policy -->|true| Drop[메시지 드롭<br/>연결 유지<br/>seq만 증가]
    Policy -->|false| Close[ws.close 1008<br/>slow consumer]
    
    Drop --> RecordSlow[reportedSlowPayloadClients.add]
    Close --> RemoveClient[clients.delete]
    
    Send --> Done
    Drop --> Done
    Close --> Done
    
    style Drop fill:#FFFACD
    style Close fill:#FFB6C1
    style Send fill:#90EE90
```

### 9.3 사용 예

```typescript
// 빈번 이벤트 (presence) — drop OK
broadcast("presence", { /* ... */ }, { dropIfSlow: true });

// 중요 이벤트 (chat done) — drop X, 연결 종료
broadcast("chat", { state: "done", /* ... */ });
// dropIfSlow 미설정 → false → 연결 종료
```

---

## 10. Auth Rate Limiter

### 10.1 슬라이딩 윈도우 + 잠금

`src/gateway/auth-rate-limit.ts:99-236`:

```typescript
// 기본값
const maxAttempts = 10;
const windowMs = 60_000;       // 1분 슬라이딩 윈도우
const lockoutMs = 300_000;     // 5분 잠금
```

### 10.2 동작

```mermaid
stateDiagram-v2
    [*] --> open: 정상
    
    open --> open: check() success<br/>(remaining > 0)
    
    open --> recordFailed: 인증 실패<br/>action: attempts.push(now)
    
    recordFailed --> open: attempts < maxAttempts
    recordFailed --> locked: attempts >= maxAttempts<br/>action: lockedUntil = now + lockoutMs
    
    locked --> locked: 시도<br/>(retryAfterMs 응답)
    
    locked --> open: now >= lockedUntil<br/>action: clear attempts
    
    open --> open: pruneTimer (60s)<br/>action: 만료 entry 제거
```

### 10.3 Scope별 분리

```typescript
export const AUTH_RATE_LIMIT_SCOPE_DEFAULT = "default";
export const AUTH_RATE_LIMIT_SCOPE_SHARED_SECRET = "shared-secret";
export const AUTH_RATE_LIMIT_SCOPE_DEVICE_TOKEN = "device-token";
export const AUTH_RATE_LIMIT_SCOPE_HOOK_AUTH = "hook-auth";
```

→ scope별 독립적 카운터. shared-secret 시도 한도가 device-token에 영향 없음.

### 10.4 Loopback 예외

```typescript
const exemptLoopback = config?.exemptLoopback ?? true;
// 127.0.0.1, ::1 등은 rate limit 면제 (CLI 사용 보호)
```

---

## 11. 외부 API Rate Limiting

```mermaid
flowchart LR
    Code[OpenClaw Code] --> Provider[Provider Plugin]
    Provider --> Retry{rate_limit?}
    Retry -->|yes 429| BackOff[exponential backoff]
    BackOff --> RetryAfter{retry-after header?}
    RetryAfter -->|yes| WaitHeader[해당 시간 대기]
    RetryAfter -->|no| WaitExp[지수 백오프 + jitter]
    WaitHeader --> Retry2[재시도]
    WaitExp --> Retry2
    Retry2 --> Code
    
    Retry -->|no, 다른 에러| Failover[FailoverError]
    Failover --> AltProvider[다른 모델/profile 시도]
    
    style BackOff fill:#FFE4B5
    style Failover fill:#FFB6C1
```

`src/infra/retry.ts:69-137`:
```typescript
const baseDelay = hasRetryAfter
  ? Math.max(retryAfterMs, minDelayMs)
  : minDelayMs * 2 ** (attempt - 1);  // exponential
let delay = Math.min(baseDelay, maxDelayMs);
delay = applyJitter(delay, jitter);    // thundering herd 방지
```

기본 정책: **3 attempts, 400ms-30s, 10% jitter**.

---

## 12. Test Parallelism 제어

### 12.1 Vitest 워커 결정

`test/vitest/vitest.shared.config.ts:75-116`:

```typescript
// 환경변수 우선
if (hasWorkerOverride(env)) {
  return { /* OPENCLAW_VITEST_MAX_WORKERS 사용 */ };
}

// CI
if (isCI) {
  return {
    fileParallelism: true,
    maxWorkers: isWindows ? 2 : 3,
  };
}

// 로컬: CPU-적응
return localScheduling;
```

### 12.2 Module Cache Race 방어

`test/vitest/vitest.performance-config.ts`:

```typescript
if (env.OPENCLAW_VITEST_FS_MODULE_CACHE_PATH?.trim()) {
  experimental.fsModuleCachePath = env.OPENCLAW_VITEST_FS_MODULE_CACHE_PATH;
}
```

여러 Vitest 프로세스가 같은 worktree에서 돌면 `node_modules/.experimental-vitest-cache`에서 race condition (`ENOTEMPTY`).

해결:
```bash
# 1. 단일 프로세스
OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test

# 2. 분리된 캐시 (PID 기반)
OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/cache-${PID} pnpm test
```

---

## 13. 동시성 안전성 매트릭스

| 영역 | 메커니즘 | 동시성 모델 |
|------|---------|------------|
| 세션 메시지 | session:{id} lane (maxConcurrent=1) | 사용자별 직렬 |
| Subagent | subagent lane + tree 제약 | 5 children, depth 1 |
| Cron | cron + cron-nested lane | 1 job, 1 nested |
| 메인 처리 | main lane | 4 동시 |
| Channel inbound | grammy sink concurrency | 4 |
| LLM 호출 | retry policy (provider별) | 3 attempts, exp backoff |
| Auth 시도 | rate limiter (scope별) | 10/min, 5min lock |
| WebSocket | per-client buffered check | dropIfSlow / close |
| Plugin loading | pinned registry | snapshot 안정 |
| Manifest cache | LRU 512 | 비결정적 갱신 |
| OAuth refresh | file lock | 단일 갱신 보장 |
| Session store | exclusive write lock | atomic write |
| Plugin state | SQLite | DB 자체 트랜잭션 |
| Test workers | vitest config | CPU-adaptive |

---

## 14. 잠재적 병목

```mermaid
flowchart TD
    Bot["Telegram bot<br/>concurrency=4"] -.->|많은 사용자 시| Bottleneck1[봇 처리 한계]
    
    SessionLane["session:{id} lane<br/>maxConcurrent=1"] -.->|사용자가 많은 메시지 빠르게 보낼 때| Bottleneck2[큐 적체]
    
    LLM["LLM API"] -.->|rate_limit 발생 시| Bottleneck3[Failover or backoff]
    
    Storage["Session store"] -.->|많은 세션 쓰기| Bottleneck4[exclusive lock 대기]
    
    style Bottleneck1 fill:#FFB6C1
    style Bottleneck2 fill:#FFB6C1
    style Bottleneck3 fill:#FFB6C1
    style Bottleneck4 fill:#FFB6C1
```

### 완화 방법

| 병목 | 완화 |
|------|------|
| Telegram concurrency=4 | 더 많은 동시성 가능하나 API rate limit 고려 |
| Session lane=1 | 사용자별 분산 (자연스러운 분할) |
| LLM rate_limit | 다중 auth profile + failover chain |
| Session store lock | LRU + serialized cache로 read 빠름 |

---

## 15. 모니터링 / 진단

```mermaid
flowchart LR
    Metrics[Metrics] --> Lane[Lane queue depth]
    Metrics --> Active[Active task count]
    Metrics --> Wait[Wait time histogram]
    Metrics --> Buffer[Per-client bufferedAmount]
    Metrics --> RateLim[Rate limiter rejections]
    Metrics --> Abort[Active abort controllers]
    
    Lane --> Health[Health snapshot]
    Active --> Health
    Wait --> Health
    
    Health -.->|broadcast| Client[Client UI]
    
    style Health fill:#90EE90
```

`OPENCLAW_GATEWAY_STARTUP_TRACE=1` 활성화 시 startup 단계별 P50/P95/P99 측정.
