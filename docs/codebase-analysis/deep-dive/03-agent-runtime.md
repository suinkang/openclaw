# Deep Dive: Agent 런타임 (실제 코드 분석)

> 실제 `.ts` 소스 기준. 이전 문서의 일부 추정치를 정정합니다.

## 0. 정정된 사실

이전 분석에서 부정확했던 부분:

| 항목 | 이전 (틀림) | 실제 |
|------|------------|------|
| Active memory timeout | 5000ms | **15000ms (DEFAULT_TIMEOUT_MS)** |
| promptStyle 값 | 3개 (balanced/strict/recall-heavy) | **6개** |
| Memory subagent | "sub-agent spawn" | **별도 LLM 호출 (실제로는 spawn 안 함)** |

## 1. 디렉토리 구조

`src/agents/` 하위:

| 폴더 | 역할 |
|------|------|
| `runtime-plan/` | Plan 빌드 (`build.ts`, `types.ts`, `auth.ts`, `tools.ts`) |
| `pi-embedded-runner/` | 실행 엔진. 124개 `.ts` 파일 |
| `sandbox/` | 샌드박스 (`docker.ts`, `tool-policy.ts` 등 20개) |
| `tools/` | 65+ 내장 도구 (`web-search`, `sessions-spawn` 등) |
| `auth-profiles/` | 인증 프로필 관리 (69개 파일) |
| `harness/` | 모델별 출력 정규화 |

## 2. AgentRuntimePlan — 실제 인터페이스

`src/agents/runtime-plan/types.ts:342-368`:

```typescript
export type AgentRuntimePlan = {
  resolvedRef: AgentRuntimeResolvedRef;
  providerRuntimeHandle?: AgentRuntimeProviderHandle;
  auth: AgentRuntimeAuthPlan;
  prompt: AgentRuntimePromptPlan;
  tools: AgentRuntimeToolPlan;
  transcript: {
    policy: AgentRuntimeTranscriptPolicy;
    resolvePolicy(params?: {
      workspaceDir?: string;
      modelApi?: string;
      model?: AgentRuntimeModel;
    }): AgentRuntimeTranscriptPolicy;
  };
  delivery: AgentRuntimeDeliveryPlan;
  outcome: AgentRuntimeOutcomePlan;
  transport: AgentRuntimeTransportPlan;
  observability: {
    resolvedRef: string;
    provider: string;
    modelId: string;
    modelApi?: string;
    harnessId?: string;
    authProfileId?: string;
    transport?: AgentRuntimeTransport;
  };
};
```

### 2.1 9개 영역

1. **`resolvedRef`** — 모델 참조 해석 결과
2. **`providerRuntimeHandle`** — 프로바이더 플러그인 핸들
3. **`auth`** — 인증 프로필 순서, 위임 전략
4. **`prompt`** — System prompt 빌드, 텍스트 변환
5. **`tools`** — 도구 정규화, 메타데이터 스냅샷 lazy 로드
6. **`transcript`** — 메시지 정규화 정책 (Anthropic/Google/OpenAI 호환)
7. **`delivery`** — 응답 라우팅, 팔로우업 결정
8. **`outcome`** — 결과 처리
9. **`transport`** — Extra params (thinking level, temperature 등)

### 2.2 System Prompt Contribution 타입

`types.ts:188-192`:
```typescript
export type AgentRuntimeSystemPromptContribution = {
  stablePrefix?: string;          // 캐시 친화적 안정 prefix
  dynamicSuffix?: string;         // 매 요청 변경 가능
  sectionOverrides?: Partial<Record<
    "interaction_style" | "tool_call_style" | "execution_bias",
    string
  >>;
};
```

## 3. buildAgentRuntimePlan — 실제 흐름

`src/agents/runtime-plan/build.ts:134-336`:

```typescript
export function buildAgentRuntimePlan(params: BuildAgentRuntimePlanParams): AgentRuntimePlan {
  // 1. Config 정규화
  const config = asOpenClawConfig(params.config);
  const model = asProviderRuntimeModel(params.model);

  // 2. Provider 핸들 해석
  const providerRuntimeHandleForPlugins = resolveProviderRuntimeHandleForPlugins({
    provider: params.provider,
    config,
    workspaceDir: params.workspaceDir,
    runtimeHandle: params.providerRuntimeHandle,
    resolveWhenMissing: true,
  });

  // 3. Auth plan
  const auth = buildAgentRuntimeAuthPlan({...});

  // 4. Prompt plan
  prompt: {
    resolveSystemPromptContribution(context) {
      return resolveProviderSystemPromptContribution({...});
    },
    transformSystemPrompt(context) {
      return transformProviderSystemPrompt({...});
    }
  }

  // 5. Tools plan (lazy metadata snapshot)
  tools: {
    preparedPlanning: {
      loadMetadataSnapshot: loadToolPlanningMetadataSnapshot,
    },
    normalize<TSchemaType extends TSchema = TSchema, TResult = unknown>(
      tools: AgentTool<TSchemaType, TResult>[],
      overrides?: {...}
    ): AgentTool<TSchemaType, TResult>[] {
      return normalizeProviderToolSchemas({...tools});
    }
  }

  // 6. Transcript policy (lazy getter!)
  transcript: {
    get policy() {
      return resolveDefaultTranscriptPolicy();
    },
    resolvePolicy: resolveTranscriptRuntimePolicy,
  }

  // 7. Transport (lazy getter!)
  transport: {
    get extraParams() {
      return resolveDefaultTransportExtraParams();
    },
    resolveExtraParams: resolveTransportExtraParams,
  }
}
```

**중요**: `get policy`와 `get extraParams`는 **getter** — 접근 시점에 계산. Plan 빌드 시 비싼 일 안 함.

## 4. 메인 실행 함수: runEmbeddedPiAgent

`src/agents/pi-embedded-runner/run.ts:337+` 흐름:

```typescript
export async function runEmbeddedPiAgent(params: RunEmbeddedPiAgentParams) {
  // (라인 342-350) sessionKey 백필
  // (라인 351-361) Lane 해석
  //   sessionLane = resolveSessionLane(sessionKey);
  //   globalLane = resolveGlobalLane(params.lane);

  // (라인 414-433) 워크스페이스 해석
  //   resolveRunWorkspaceDir()

  // (라인 434-439) 런타임 플러그인 로드
  //   ensureRuntimePluginsLoaded()

  // (라인 441-530) 모델 & 프로바이더 선택
  //   - resolveHookModelSelection() — 사용자 정의 hook
  //   - selectAgentHarness() — 모델별 harness
  //   - resolveModelAsync()

  // RuntimePlan 빌드
  // System Prompt 빌드 (provider contribution + section overrides)
  // Tool 정규화

  // Message Attempt Loop (재시도 가능)
  //   runEmbeddedAttemptWithBackend():
  //     - tool call 파싱
  //     - tool 실행 또는 dispatch:
  //       - sessions_spawn → spawnSubagentDirect()
  //       - self-aware tools → in-process
  //       - external tool → Process execution
  //     - tool result 응답
  //     - streaming/completion 처리
  //
  //   Empty/Overflow → 재시도/Compaction
  //   Failover → Auth/Model fallback
  //   Success → 루프 종료

  // Compaction (필요 시)
  //   - Context overflow 감지
  //   - Before/After hooks

  // Delivery route 결정
  //   delivery.resolveFollowupRoute()

  // EmbeddedPiRunResult 반환
}
```

## 5. Subagent Spawn — 진짜 구현

`src/agents/subagent-spawn.ts:675+` `spawnSubagentDirect`:

```typescript
export async function spawnSubagentDirect(
  params: SpawnSubagentParams,
  ctx: SpawnSubagentContext,
): Promise<SpawnSubagentResult> {
  // 1. Depth 검증
  const callerDepth = getSubagentDepthFromSessionStore(requesterInternalKey, { cfg });
  if (callerDepth >= maxSpawnDepth) {
    return { status: "forbidden", error: "depth exceeded" };
  }

  // 2. 활성 자식 수 검증
  const activeChildren = countActiveRunsForSession(requesterInternalKey);
  if (activeChildren >= maxChildren) {
    return { status: "forbidden", error: "max children exceeded" };
  }

  // 3. 자식 세션 키 생성
  const childSessionKey = `agent:${targetAgentId}:subagent:${crypto.randomUUID()}`;

  // 4. Context mode 결정 (격리 정도)
  const contextMode = resolveSubagentContextMode({
    requestedContext: params.context,    // "full" | "partial" | "empty"
    threadRequested: params.thread === true,
    cfg,
    requester: { channel: ctx.agentChannel, accountId: ctx.agentAccountId }
  });

  // 5. 자식 세션 초기화
  const initialChildSessionPatch: Record<string, unknown> = {
    spawnDepth: childDepth,
    subagentRole: childCapabilities.role,
    subagentControlScope: childCapabilities.controlScope,
    model, thinkingLevel, ...
  };

  // 6. Gateway 호출 (sessions.patch 또는 agent)
  const gatewayCfg = callSubagentGateway({
    method: spawnMode === "session" ? "sessions.patch" : "agent",
    params: { sessionKey: childSessionKey, task: params.task, model: plan.resolvedModel, ... },
    scopes: [ADMIN_SCOPE],
    timeoutMs: resolveSubagentAgentGatewayTimeoutMs(runTimeoutSeconds),
  });

  // 7. 결과 반환
  return {
    status: "accepted",
    childSessionKey,
    runId: readGatewayRunId(gatewayCfg),
    modelApplied,
    ...
  };
}
```

### 5.1 Context Mode

`subagent-spawn.ts:729-737`:
```typescript
const contextMode = resolveSubagentContextMode({...});
// 결과:
// - "full"    → 모든 컨텍스트 전달 (transcript, files, tools)
// - "partial" → 최근 메시지만
// - "empty"   → Task 텍스트만
```

### 5.2 Subagent Role Capabilities

`subagent-spawn.ts:846-849`:
```typescript
const childCapabilities = resolveSubagentCapabilities({
  depth: childDepth,
  maxSpawnDepth,
});
// role: "main" | "orchestrator" | "leaf"
// controlScope: "children" | "none"
// Leaf는 자식 spawn 불가
```

→ 트리 구조 제한. Leaf 노드는 더 이상 분기 못 함. 무한 재귀 방어.

### 5.3 인증 스코프

`callSubagentGateway`에 `scopes: [ADMIN_SCOPE]`. Subagent는 admin 권한으로 Gateway에 RPC. 외부 클라이언트와 다른 권한 컨텍스트.

## 6. Active Memory — 실제 구현

`extensions/active-memory/index.ts`

### 6.1 설정 타입

```typescript
type ActiveRecallPluginConfig = {
  enabled?: boolean;
  agents?: string[];
  model?: string;                  // 별도 recall 모델
  timeoutMs?: number;              // 기본: 15,000ms!
  queryMode?: "message" | "recent" | "full";
  promptStyle?:
    | "balanced"
    | "strict"
    | "contextual"
    | "recall-heavy"
    | "precision-heavy"
    | "preference-only";
  promptOverride?: string;
  thinking?: ActiveMemoryThinkingLevel;
  // ...
};
```

### 6.2 상수들

```typescript
const DEFAULT_TIMEOUT_MS = 15_000;                    // 15초!
const TIMEOUT_PARTIAL_DATA_GRACE_MS = 500;            // 라인 50
const MAX_ACTIVE_MEMORY_SEARCH_QUERY_CHARS = 480;
const activeRecallCache = new Map<string, CachedActiveRecallResult>();
const timeoutCircuitBreaker = new Map<string, CircuitBreakerEntry>();
```

### 6.3 중요한 사실

> Memory extension은 **내부적으로 sub-agent를 spawn하지 않습니다.**

대신:
- **별도의 LLM 호출** — `model` 옵션으로 지정된 모델 사용
- 부모 에이전트의 transcript에서 직접 recall query 생성
- Timeout: 15초 기본

이전 분석에서 "sub-agent spawn"이라 했던 부분은 부정확. 단순 inline LLM call.

### 6.4 Circuit Breaker

`active-memory/index.ts:318-349`:
```typescript
function isCircuitBreakerOpen(key: string, maxTimeouts: number, cooldownMs: number): boolean {
  const entry = timeoutCircuitBreaker.get(key);
  if (!entry || entry.consecutiveTimeouts < maxTimeouts) {
    return false;
  }
  if (Date.now() - entry.lastTimeoutAt >= cooldownMs) {
    timeoutCircuitBreaker.delete(key);
    return false;
  }
  return true;
}
```

연속 timeout 발생 → circuit open → 일정 시간 동안 recall 시도 안 함. 사용자 응답 지연 방지.

### 6.5 promptStyle 6가지 — 자동 선택 규칙

`active-memory/index.ts:911-930`:
```typescript
function resolvePromptStyle(
  promptStyle: unknown,
  queryMode: ActiveRecallPluginConfig["queryMode"],
): ActiveMemoryPromptStyle {
  if (promptStyle === "balanced" || promptStyle === "strict" ||
      promptStyle === "contextual" || promptStyle === "recall-heavy" ||
      promptStyle === "precision-heavy" || promptStyle === "preference-only") {
    return promptStyle;
  }
  if (queryMode === "message") {
    return "preference-only";       // message 모드 기본
  }
  if (queryMode === "full") {
    return "recall-heavy";           // full 모드 기본
  }
  return "balanced";
}
```

### 6.6 queryMode 효과

`active-memory/index.ts:2053-2056`:
- `"message"` — 최신 user 메시지만 recall query에 사용
- `"recent"` — 최근 2 user + 1 assistant turns
- `"full"` — 전체 transcript 요약

### 6.7 System Prompt 주입

`active-memory/index.ts:999-1000`:
```typescript
systemPromptLines.push(
  `Prompt style: ${params.config.promptStyle}.`,
  ...buildPromptStyleLines(params.config.promptStyle),
);
```

## 7. LanceDB Memory Backend

### 7.1 모듈 로드

`extensions/memory-lancedb/lancedb-runtime.ts`:
```typescript
type LanceDbModule = typeof import("@lancedb/lancedb");

export async function loadLanceDbModule(
  logger?: LanceDbRuntimeLogger
): Promise<LanceDbModule> {
  return await defaultLoader.load(logger);
}
```

### 7.2 플랫폼 제약

`lancedb-runtime.ts:22-27`:
```typescript
function isUnsupportedNativePlatform(params: {
  platform: NodeJS.Platform;
  arch: NodeJS.Architecture;
}): boolean {
  return params.platform === "darwin" && params.arch === "x64";
}
```

→ **Intel Mac (x64)는 미지원**. Apple Silicon (arm64) Mac과 Linux/Windows만.

### 7.3 임베딩 모델

코드에서 직접 정의되지 않음. `@lancedb/lancedb` 패키지의 기본 임베딩 모델 사용 (런타임 결정).

## 8. Memory Wiki — 4개 도구

`extensions/memory-wiki/src/tool.ts`:

### 8.1 wiki_search (라인 118-165)

```typescript
{
  name: "wiki_search",
  parameters: {
    query: string,
    maxResults?: number,
    backend?: "lancedb" | "obsidian",
    corpus?: "wiki" | "memory",
    mode?: "inherit" | "search" | "vsearch" | "query"
  },
  execute: async () => {
    const results = await searchMemoryWiki({
      config, appConfig, agentId, agentSessionKey, sandboxed,
      query, maxResults, searchBackend, searchCorpus, mode
    });
  }
}
```

### 8.2 wiki_get (라인 235-250)
페이지 ID 또는 경로로 위키 페이지 읽기.

### 8.3 wiki_apply (라인 203-232)
새 synthesis 생성 또는 메타데이터 업데이트.

### 8.4 wiki_status (라인 95-115)
Vault 상태 + Obsidian CLI 가용성 확인.

→ Memory Wiki는 Obsidian 호환 (markdown vault) + LanceDB 벡터 검색 두 백엔드 지원.

## 9. Sandbox 시스템

`src/agents/sandbox/`:

### 9.1 모드

- **Docker** — 컨테이너 격리 (기본)
- **SSH** — 원격 머신 (`ssh.ts`)
- **직접** — Host 명령 (개발/디버깅)

### 9.2 Tool Policy

`src/agents/sandbox/tool-policy.ts`에서 결정:
- `bash` — 샌드박스에서만
- `web_search` — 항상 허용
- `image_generate` — 모델 API 액세스 필요
- `sessions_send` — 채널 권한 필요

## 10. Auto-Response 결정

`src/auto-reply/reply/`:
- `get-reply.js`
- `directives.js`

응답 여부 결정 요인:
1. **Trigger** — `"user"`, `"cron"`, `"heartbeat"`, `"memory"`, `"overflow"`
2. **Directives** — `@elevated`, `@reasoning`, `@think`, `@verbose`, `@exec`
3. **Channel** — Messaging vs notification
4. **Session state** — Active run? Suspended?

## 11. 메시지 처리 — 진짜 시퀀스

```
사용자 메시지 도착 (채널/API)
  ↓
Gateway (라우팅, 권한)
  ↓
runEmbeddedPiAgent() [pi-embedded-runner/run.ts:337]
  ├─ sessionKey 백필
  ├─ Lane 해석
  ├─ 워크스페이스 해석
  ├─ 런타임 플러그인 로드
  ├─ 모델 & 프로바이더 선택 (hook 가능)
  │
  ↓
buildAgentRuntimePlan() [runtime-plan/build.ts:134]
  ├─ Auth plan
  ├─ Prompt plan
  ├─ Tools plan (lazy metadata)
  ├─ Transcript policy (getter, lazy)
  ├─ Delivery plan
  └─ Transport plan (getter, lazy)
  │
  ↓
System Prompt 빌드
  ├─ Provider contribution
  ├─ Section overrides (interaction_style 등)
  └─ Text transform 적용
  │
  ↓
Active Memory? → 별도 LLM call (15s timeout, circuit breaker)
  ├─ queryMode으로 입력 결정
  ├─ promptStyle로 prompt 결정
  └─ Recall 결과 → System prompt에 주입
  │
  ↓
Tool 정규화 (provider별)
  │
  ↓
runEmbeddedAttemptWithBackend() [Loop]
  ├─ LLM 호출 (stream)
  ├─ Tool call 파싱
  │   ├─ sessions_spawn → spawnSubagentDirect()
  │   │   ├─ depth 검증
  │   │   ├─ children count 검증
  │   │   ├─ context mode 결정
  │   │   └─ Gateway RPC
  │   ├─ web_search
  │   ├─ bash → sandbox
  │   └─ ...
  ├─ Tool result → 다음 turn
  ├─ Empty/Overflow → 재시도 또는 Compaction
  └─ Failover (auth/rate_limit/timeout)
  │
  ↓
Delivery route 결정 [delivery.resolveFollowupRoute()]
  ├─ 원래 채널?
  ├─ Dispatcher?
  └─ Drop?
  │
  ↓
ReplyPayload 빌드 [run/payloads.ts]
  ├─ text
  ├─ mediaUrls
  ├─ presentation (tone, urgency)
  ├─ interactive (buttons, menu)
  └─ delivery (pin, silent)
  │
  ↓
응답 전송 (Channel API)
  │
  ↓
EmbeddedPiRunResult 반환 { payloads[], meta{durationMs, agentMeta, usage} }
```

## 12. 정리

| 항목 | 실제 |
|------|------|
| Plan 빌드 함수 | `buildAgentRuntimePlan` (build.ts:134) |
| 메인 runner | `runEmbeddedPiAgent` (run.ts:337) |
| Subagent spawn | `spawnSubagentDirect` (subagent-spawn.ts:675) |
| Active memory timeout | **15000ms** (5초 아님!) |
| promptStyle 옵션 | **6개** |
| Memory subagent | 진짜 spawn 아님, 별도 inline LLM 호출 |
| LanceDB Intel Mac | **미지원** |
| Memory wiki 도구 | 4개 (search/get/apply/status) |
| 샌드박스 | Docker/SSH/직접 |
| Subagent 격리 | full / partial / empty 3단계 |
| Subagent 트리 | role: main/orchestrator/leaf, leaf는 분기 불가 |
