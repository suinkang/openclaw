# 05. LLM Provider 통합

## 개요

OpenClaw는 다양한 LLM 프로바이더를 플러그인으로 지원합니다. 각 프로바이더는 독립된 `extensions/<id>/` 패키지로 구현되며, Core는 통일된 인터페이스만 알고 프로바이더별 세부 구현은 모릅니다.

## 지원 프로바이더

`extensions/`에서 발견 가능한 프로바이더 플러그인:

### 호스티드 클라우드
- **Anthropic** (`extensions/anthropic/`) — Claude 모델
- **OpenAI** (`extensions/openai/`) — GPT-4, GPT-5 등
- **Google** (`extensions/google/`) — Gemini
- **Amazon Bedrock** (`extensions/amazon-bedrock/`)
- **Azure OpenAI**
- **Deepseek**, **Fireworks**, **Perplexity**, **Together**, **DeepInfra**

### 로컬 / 자체 호스팅
- **Ollama** (`extensions/ollama/`)
- **LM Studio** (`extensions/lmstudio/`)
- **vLLM** (호환 서버)

### 전문화
- **Copilot Proxy** — GitHub Copilot 백엔드
- **Azure Speech** — TTS/STT 전용

## 3계층 분리

`AGENTS.md:39`:
> providers: core owns generic loop; provider plugins own auth/catalog/runtime hooks.

각 프로바이더 플러그인은 세 영역을 소유:

### Layer 1: Auth
- API key, OAuth 토큰, refresh 흐름
- 자격증명 저장 위치:
  - 채널 자격증명: `~/.openclaw/credentials/`
  - 모델 auth profiles: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- 환경변수 fallback: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` 등

### Layer 2: Catalog
- 사용 가능한 모델 목록 (`provider-discovery.ts`)
- 모델별 메타데이터:
  - 컨텍스트 윈도우
  - 최대 출력 토큰
  - 비전/오디오/도구 지원 여부
  - 가격 (선택)

`src/model-catalog/`가 모든 프로바이더의 카탈로그를 수집.

### Layer 3: Runtime Hooks
- 스트림 wrapper — 프로바이더별 SSE 형식을 통일된 이벤트로 변환
- 재시도 정책 — rate limit, 5xx, auth error 처리
- 도구 스키마 정규화 — JSON Schema 변환
- 프롬프트 캐싱 — 결정적 ordering 보장

## Provider 매니페스트

`extensions/anthropic/openclaw.plugin.json`:
```json
{
  "id": "anthropic",
  "providers": ["anthropic"],
  "providerDiscoveryEntry": "./provider-discovery.ts",
  "modelSupport": {
    "modelPrefixes": ["claude-"]
  },
  "providerAuthEnvVars": {
    "anthropic": ["ANTHROPIC_OAUTH_TOKEN", "ANTHROPIC_API_KEY"]
  },
  "contracts": {
    "mediaUnderstandingProviders": ["anthropic"]
  }
}
```

`modelPrefixes`로 어떤 모델 ID가 이 프로바이더에 속하는지 결정. 예: `claude-opus-4-7`, `claude-sonnet-4-6`은 Anthropic 플러그인이 처리.

## Provider Runtime 핸들

```typescript
type ProviderRuntimePluginHandle = {
  id: string;
  
  // 인증
  resolveAuth(scope: AuthScope): Promise<AuthCredential>;
  
  // 모델 호출 (스트리밍)
  stream(params: StreamParams): AsyncIterable<StreamEvent>;
  
  // 도구 스키마 정규화
  normalizeToolSchema(schema: ToolSchema): ProviderToolSchema;
  
  // 재시도 정책
  shouldRetry(error: Error, attempt: number): RetryDecision;
  
  // 미디어 처리
  processMedia?(media: Media[]): Promise<NormalizedMedia[]>;
};
```

## 도구 스키마 정규화

각 프로바이더의 도구(function calling) 스키마는 미묘하게 다릅니다:

### OpenAI
```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "parameters": { "type": "object", "properties": {...} }
  }
}
```

### Anthropic
```json
{
  "name": "get_weather",
  "input_schema": { "type": "object", "properties": {...} }
}
```

### Google Gemini
```json
{
  "name": "get_weather",
  "parameters": { "type": "OBJECT", "properties": {...} }
}
```

OpenClaw는 통일된 내부 표현을 받아 프로바이더별로 변환.

### Provider 호환성 워닝

`AGENTS.md:202`:
> Provider tool schemas: prefer flat string enum helpers over `Type.Union([Type.Literal(...)])`; some providers reject `anyOf`.

Gemini는 `anyOf` 거부하므로 enum은 평탄한 string 배열로 변환:

```typescript
// ✅ 호환
{ type: "string", enum: ["a", "b", "c"] }

// ❌ Gemini 거부
{ anyOf: [{ const: "a" }, { const: "b" }, { const: "c" }] }
```

## 스트림 처리

각 프로바이더는 자체 스트림 형식 (SSE 기반이지만 이벤트 종류 다름):

### Anthropic
```
event: message_start
event: content_block_start
event: content_block_delta  (text/tool_use)
event: content_block_stop
event: message_delta
event: message_stop
```

### OpenAI
```
data: {"choices":[{"delta":{"content":"..."}}]}
data: {"choices":[{"delta":{"tool_calls":[...]}}]}
data: [DONE]
```

각 프로바이더 플러그인이 `stream-wrappers.ts`에서 통일된 `StreamEvent`로 변환:

```typescript
type StreamEvent =
  | { type: "text_delta"; text: string }
  | { type: "tool_call_start"; id: string; name: string }
  | { type: "tool_call_delta"; id: string; arguments: string }
  | { type: "tool_call_end"; id: string }
  | { type: "thinking_delta"; text: string }     // Claude 확장
  | { type: "stop"; reason: StopReason }
  | { type: "usage"; tokens: TokenUsage };
```

## 재시도 정책

`extensions/anthropic/replay-policy.ts` 등에서 정의:

```typescript
function shouldRetry(error: Error, attempt: number): RetryDecision {
  // Rate limit (429)
  if (error.code === 429) {
    return { retry: true, delayMs: parseRetryAfter(error) };
  }
  // Auth (401) — refresh token 시도
  if (error.code === 401 && attempt === 0) {
    return { retry: true, refreshAuth: true };
  }
  // 5xx — 지수 백오프
  if (error.code >= 500 && attempt < 3) {
    return { retry: true, delayMs: 1000 * 2 ** attempt };
  }
  return { retry: false };
}
```

## Failover

복수 프로바이더에 걸친 failover 가능:

```yaml
agents:
  main:
    model: claude-opus-4-7
    failover:
      - claude-sonnet-4-6        # Claude 다운 시
      - gpt-5.5                  # Anthropic 전체 다운 시
```

`AgentRuntimePlan.failoverReason`이 트리거:
- `auth` — 인증 실패
- `rate_limit` — 빈도 제한
- `timeout` — 응답 타임아웃
- `provider_error` — 5xx
- 기타

## 프롬프트 캐싱 결정성

`AGENTS.md:48`:
> Prompt cache: deterministic ordering for maps/sets/registries/plugin lists/files/network results before model/tool payloads. Preserve old transcript bytes when possible.

Claude/OpenAI 프롬프트 캐싱은 **바이트 정확도**로 작동. 이전과 동일한 프리픽스를 유지해야 캐시 히트.

### 위반 예 (캐시 미스)
```typescript
const tools = Object.values(toolRegistry);  // 객체 순서 비결정적!
const prompt = JSON.stringify({ tools });    // 매번 다름 → 캐시 미스
```

### 준수 예
```typescript
const tools = Object.values(toolRegistry).sort((a, b) => a.id.localeCompare(b.id));
const plugins = Array.from(pluginsSet).sort();
const prompt = JSON.stringify({ tools, plugins });  // 결정적 → 캐시 히트
```

## Auth Profiles

복수 자격증명 동시 보유 가능:

`~/.openclaw/agents/main/agent/auth-profiles.json`:
```json
{
  "profiles": [
    {
      "id": "personal-anthropic",
      "provider": "anthropic",
      "type": "oauth",
      "token": "...",
      "refreshToken": "...",
      "expiresAt": 1735000000
    },
    {
      "id": "work-anthropic",
      "provider": "anthropic",
      "type": "api_key",
      "key": "..."
    }
  ],
  "active": "personal-anthropic"
}
```

UI/CLI에서 프로파일 전환 가능. 보안: 개인 vs 회사 결제 분리.

## 모델 선택 흐름

```
agent config → model: "claude-opus-4-7"
  → catalog 조회 → provider: "anthropic"
  → provider plugin 로드
  → auth profile 해석
  → runtime handle 빌드
  → AgentRuntimePlan 캐시
```

이후 매 요청에서 캐시된 plan 재사용 (prepared facts).

## Carbon 핀 (오너 전용)

`AGENTS.md:189`:
> Carbon pins owner-only: do not change `@buape/carbon` unless Shadow asks.

Discord 플러그인이 사용하는 `@buape/carbon` 라이브러리는 메인테이너(Shadow) 승인 없이 버전 변경 금지. 의존성 패치/오버라이드는 명시 승인 필요.
