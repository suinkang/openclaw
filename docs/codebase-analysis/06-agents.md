# 06. Agent 런타임과 메모리 시스템

## Agent란?

OpenClaw의 **Agent**는 단순 LLM 프롬프트가 아니라, 다음을 묶은 실행 단위입니다:

- 사용 모델 (`claude-opus-4-7` 등)
- 시스템 프롬프트 + 페르소나
- 활성 도구 목록 (skills)
- 메모리 백엔드 연결
- 라우팅 정책 (어떤 채널에서 응답?)
- Auth profile (어떤 자격증명?)
- 샌드박스 모드

여러 에이전트를 동시 운영 가능. 예:
- `main` — 개인 비서 (전체 권한)
- `team-assistant` — 팀 채널 (제한 권한)
- `coder` — 코딩 전용 (긴 컨텍스트, code 도구만)

## 디렉토리

```
src/agents/
├── runtime-plan/         # Plan 빌더
│   ├── build.ts          # buildAgentRuntimePlan
│   └── types.ts          # AgentRuntimePlan, ProviderRuntimePluginHandle
├── pi-embedded-runner/   # 인-프로세스 실행기
│   ├── tool-schema-runtime.ts
│   └── ...
├── pi-hooks/             # 후킹 시스템
└── ...
```

## AgentRuntimePlan

핵심 prepared fact 객체. 시작 시 한 번 빌드, 모든 요청에서 재사용.

`src/agents/runtime-plan/types.ts`:
```typescript
type AgentRuntimePlan = {
  agentId: string;
  provider: string;                              // "anthropic"
  modelRef: string;                              // "claude-opus-4-7"
  providerRuntimeHandle: ProviderRuntimePluginHandle;
  
  // 모델 메타
  contextWindow: number;
  maxTokens: number;
  
  // 도구
  tools: PreparedTool[];
  toolFailoverPolicy: ToolFailoverPolicy;
  
  // 사고 (thinking)
  thinkLevel: AgentRuntimeThinkLevel;
  
  // 인증
  authPlan: AuthPlan;
  
  // Failover
  failoverChain: AgentRuntimeFailover[];
  
  // 전송
  transport: AgentRuntimeTransport;              // "sse" | "websocket" | "auto"
};

type AgentRuntimeThinkLevel = 
  | "off" | "minimal" | "low" | "medium" 
  | "high" | "xhigh" | "adaptive" | "max";

type AgentRuntimeFailoverReason =
  | "auth" | "rate_limit" | "timeout" 
  | "provider_error" | "context_overflow";
```

## Plan 빌드

`src/agents/runtime-plan/build.ts`:
```typescript
export async function buildAgentRuntimePlan(
  params: BuildAgentRuntimePlanParams
): Promise<AgentRuntimePlan> {
  // 1. 모델 → 프로바이더 해석
  const provider = resolveProviderFromModel(params.model);
  
  // 2. 프로바이더 런타임 핸들 (lazy 로드)
  const providerRuntimeHandle = await loadProviderRuntime(provider);
  
  // 3. Auth plan 구축
  const authPlan = await buildAuthPlan(provider, params.authProfileId);
  
  // 4. 도구 로드 + 정규화
  const tools = await loadTools(params.skills);
  const normalizedTools = tools.map(t => 
    providerRuntimeHandle.normalizeToolSchema(t.schema)
  );
  
  // 5. Failover chain
  const failoverChain = buildFailoverChain(params.failover);
  
  return { /* ... */ };
}
```

## 에이전트 구성 파일

```yaml
# ~/.openclaw/agents.yaml
agents:
  main:
    model: claude-opus-4-7
    persona:
      name: "Claude"
      style: "concise, direct"
    skills:
      - github
      - notion
      - canvas
    memory:
      backend: active-memory
      timeoutMs: 5000
    sandbox:
      mode: main                    # 또는 "non-main" (Docker)
    thinking: high
    transport: auto
    failover:
      - claude-sonnet-4-6
      - gpt-5.5
      
  coder:
    model: claude-opus-4-7
    contextWindow: 1000000          # 1M context 활용
    skills:
      - github
      - shell
      - filesystem
    sandbox:
      mode: non-main                # 격리 실행
```

## Sub-agent / Multi-agent

메인 에이전트가 다른 에이전트를 호출 가능:

```typescript
// 메인 에이전트 hook
const result = await runtime.subagent.run({
  agentId: "memory-recall",
  message: "What do I know about X?",
  timeoutMs: 5000,
  context: "isolated",              // 부모와 분리된 컨텍스트
});
```

### Sub-agent 종류

1. **Memory subagent** — 회상 작업 (active-memory)
2. **Coder subagent** — 메인 → 코딩 위임
3. **Planner subagent** — 복잡한 멀티스텝 작업 분해
4. **Tool-specific subagent** — 특정 도구 전용 (예: 브라우저)

### 격리

`AGENTS.md:148`:
> Thread-bound subagent tests that do not create a requester transcript should set `context: "isolated"` so fork-context validation does not hide lifecycle cleanup paths.

`context: "isolated"` 명시 시 부모 에이전트 transcript 격리, 별도 lifecycle.

## Memory 시스템

### 슬롯 기반 (single-active)

한 번에 **하나의 메모리 플러그인만 활성**:
- `extensions/active-memory/`
- `extensions/memory-lancedb/`
- `extensions/memory-wiki/`

### Active Memory

**Active Memory** (`extensions/active-memory/`)는 OpenClaw의 시그니처 메모리 패턴:

`extensions/active-memory/openclaw.plugin.json` 설정 옵션:
```json
{
  "configSchema": {
    "properties": {
      "enabled": { "type": "boolean" },
      "model": { "type": "string" },
      "queryMode": { 
        "enum": ["message", "recent", "full"]
      },
      "promptStyle": {
        "enum": ["balanced", "strict", "recall-heavy"]
      },
      "timeoutMs": { "type": "integer", "default": 5000 }
    }
  }
}
```

**흐름**:
```
사용자 메시지 도착
    ↓
메인 에이전트 hook 트리거
    ↓
Active memory sub-agent 호출 (timeout: 5s)
    ↓
서브에이전트:
  - 메모리 저장소 검색
  - 관련 사실 요약
  - "User likes X. Working on Y. Avoid Z." 형식 응답
    ↓
메인 에이전트 프롬프트에 주입
    ↓
응답 생성
```

**queryMode 의미**:
| 값 | 의미 |
|----|------|
| `message` | 현재 메시지만 컨텍스트로 사용 |
| `recent` | 최근 N개 메시지 |
| `full` | 전체 대화 |

**promptStyle 의미**:
| 값 | 의미 |
|----|------|
| `balanced` | 정확성/회상량 균형 |
| `strict` | 확실한 사실만 (false positive ↓) |
| `recall-heavy` | 가능한 모든 관련 사실 (recall ↑) |

### Memory LanceDB

벡터 데이터베이스 기반 (`extensions/memory-lancedb/`):
- 임베딩 모델로 메모리 인덱싱
- 시맨틱 검색
- 자동 청킹

### Memory Wiki

구조화된 위키 (`extensions/memory-wiki/`):
- 마크다운 파일 트리
- People directory: `reports/person-agent-directory.md`
- 검색 모드: `find-person`, `route-question`, `source-evidence`, `raw-claim`
- 인용/출처 검증 강제

`AGENTS.md:213`:
> Memory wiki: keep prompt digest tiny. The prompt should only say the wiki exists, prefer `wiki_search` / `wiki_get`, ...

위키 자체 내용은 프롬프트에 넣지 않고, 도구 호출로 동적 조회. 토큰 절약.

## Tool 시스템

### Tool 정의

```typescript
type Tool = {
  id: string;
  name: string;
  description: string;
  inputSchema: JSONSchema;
  execute(input: unknown, context: ToolContext): Promise<ToolResult>;
  capabilities?: {
    streaming?: boolean;
    canCancel?: boolean;
    longRunning?: boolean;
  };
};
```

### 도구 종류

- **내장**: `src/tools/` (filesystem, shell, web fetch 등)
- **스킬 기반**: `skills/<skill-id>/` (GitHub, Notion 등)
- **MCP**: `extensions/mcp/`로 외부 MCP 서버 통합
- **플러그인 도구**: `extensions/canvas/`, `extensions/browser/` 등

### Tool 실행 흐름

```
LLM이 도구 호출 결정 (tool_use chunk)
    ↓
Gateway가 호출 검증 (스키마, 권한)
    ↓
Tool execute() 실행
    ↓
결과 반환 (text/JSON/media/error)
    ↓
LLM 컨텍스트에 추가
    ↓
다음 응답 사이클
```

### 도구 실패 정책

`ToolFailoverPolicy`:
- **strict**: 도구 실패 시 즉시 사용자에게 에러
- **graceful**: 에러를 LLM 컨텍스트에 주입, LLM이 복구 시도
- **silent-skip**: 결과 없음으로 처리 (특정 도구만)

## Auto-response

`src/auto-response/`:
- 자동으로 메시지에 응답할지 결정
- "presence" 정책 (사용자가 idle / active / sleeping)
- DM vs group 차별
- Cooldown / rate limiting

예: 그룹 채널에서 봇이 매 메시지에 응답하지 않고, 멘션/질문 패턴 감지 시에만 응답.

## Session 관리

`src/session/`:
- 세션 = 하나의 대화 인스턴스
- 채널 conversationId ↔ session ID 매핑
- 세션별 transcript, 메모리 포인터, 진행 중 도구 상태
- 만료 정책 (idle timeout, 최대 길이)

## Talk Mode (음성)

### macOS
- On-device wake word detection
- Push-to-talk 오버레이
- 마이크 권한 + Audio Tap

### iOS/Android
- 지속적 wake word 감청
- 모바일 → Gateway 오디오 스트림
- TTS 응답 → 디바이스 재생

서버 측: `src/tts/`, `src/stt/` (Whisper, Apple Speech, Azure 등 플러그인)

## Streaming

`AGENTS.md:204`:
> External messaging: no token-delta channel messages. Follow `docs/concepts/streaming.md`; preview/block streaming uses edits/chunks and preserves final/fallback delivery.

채널로 매 토큰을 송신하지 않음 (Telegram 메시지 길이 폭발). 대신:
- **Preview/block streaming** — 의미 있는 청크 단위로 메시지 편집
- **Fallback** — 편집 미지원 채널은 최종 메시지만 전송
- **Final delivery 보장** — 스트림 중단 시에도 최종 응답 도착
