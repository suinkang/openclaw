# 03. 플러그인 / 익스텐션 시스템

> ℹ️ 실제 매니페스트 발견 알고리즘, 보안 검증(POSIX 권한, symlink 거부), LRU 512 캐시, Telegram(grammy)/Anthropic(`@mariozechner/pi-ai`) 의존성 등 코드 기반 정밀 분석은 [deep-dive/02-plugin-loader.md](./deep-dive/02-plugin-loader.md).

## 개요

OpenClaw의 거의 모든 옵션 기능은 **플러그인**으로 구현됩니다. `extensions/` 디렉토리에 130개 이상의 플러그인이 있으며, 각각은 독립된 npm 패키지처럼 동작합니다. Core(`src/`)는 의도적으로 어떤 플러그인도 hardcode하지 않고, 매니페스트와 계약(contract)을 통해서만 인지합니다.

## 플러그인 종류

| 종류 | 매니페스트 키 | 예시 |
|------|--------------|------|
| **Channel** | `channels: [...]` | `telegram`, `discord`, `slack`, `whatsapp`, `signal`, `imessage` |
| **Provider** | `providers: [...]` | `anthropic`, `openai`, `google`, `bedrock`, `ollama` |
| **Memory** (슬롯) | `contracts.memory` | `active-memory`, `memory-lancedb`, `memory-wiki` |
| **Tool** | `contracts.tools` | `canvas`, `browser`, `mcp`, `image-generation` |
| **Capability** | `contracts.*` | `mediaUnderstandingProviders` 등 특수 능력 |

## 매니페스트 (`openclaw.plugin.json`)

각 플러그인은 정적 JSON 매니페스트를 보유합니다.

### Channel 플러그인 예 (Telegram)

`extensions/telegram/openclaw.plugin.json`:
```json
{
  "id": "telegram",
  "activation": { "onStartup": false },
  "channels": ["telegram"],
  "channelEnvVars": {
    "telegram": ["TELEGRAM_BOT_TOKEN"]
  },
  "configSchema": {
    "type": "object",
    "properties": {
      "botToken": { "type": "string", "format": "secret" },
      "allowedUsers": { "type": "array", "items": { "type": "string" } }
    }
  }
}
```

### Provider 플러그인 예 (Anthropic)

`extensions/anthropic/openclaw.plugin.json`:
```json
{
  "id": "anthropic",
  "activation": { "onStartup": false },
  "enabledByDefault": true,
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

매니페스트의 핵심 필드:

- **`id`** — 전역 고유 ID
- **`activation.onStartup`** — true면 부팅 시 즉시 로드, false면 lazy
- **`enabledByDefault`** — 기본 활성화 여부
- **`channels` / `providers`** — 이 플러그인이 제공하는 채널/프로바이더 ID 목록
- **`*EnvVars`** — 필요한 환경변수 (UI/도큐멘테이션용)
- **`configSchema`** — JSON Schema (UI 자동 생성)
- **`contracts`** — 어떤 계약(capability)을 충족하는지

## 두 진입점 패턴

플러그인은 **두 개의 export 진입점**을 가집니다:

### `api.ts` — 정적 메타데이터

```typescript
// extensions/anthropic/api.ts
export { anthropicSetupEntry } from "./setup-entry";
export type { AnthropicConfig } from "./types";
```

특징:
- 즉시 로드됨 (코드 실행 비용 거의 0)
- 매니페스트 메타데이터, 타입, 설정 스키마 노출
- onboarding/setup 흐름 호스트

### `runtime-api.ts` — 런타임 훅

```typescript
// extensions/anthropic/runtime-api.ts
export { registerAnthropicProvider } from "./register.runtime";
export { anthropicStreamWrapper } from "./stream-wrappers";
```

특징:
- **lazy import** — 실제 사용 직전에만 로드
- 무거운 코드 (SDK, 스트림 처리, 재시도 정책 등) 호스트
- Hot path 진입점

## `.runtime.ts` lazy boundary 패턴

`AGENTS.md:130`의 핵심 규칙:
> Dynamic import: no static+dynamic import for same prod module. Use `*.runtime.ts` lazy boundary.

### ❌ 금지 패턴
```typescript
// foo.ts
import { heavy } from "./heavy-module";  // 정적

async function lazyPath() {
  const dyn = await import("./heavy-module");  // 동적 — 같은 모듈!
}
// → 번들러가 트리쉐이킹 실패, 앞단에서 비싼 import 발생
```

### ✅ 권장 패턴
```typescript
// foo.ts (정적 진입점, 빠르게 로드)
export { fooMeta } from "./foo-meta";

// foo.runtime.ts (lazy 진입점, 무거운 의존성)
export async function runFoo() {
  const sdk = await import("anthropic-sdk");
  // ...
}
```

빌드 후 `pnpm build`로 검증되며, `[INEFFECTIVE_DYNAMIC_IMPORT]` 경고 발생 시 패턴 위반.

## 플러그인 로더 (`src/plugins/`)

```
src/plugins/
├── runtime/
│   ├── index.ts                  # PluginRuntime 인터페이스 빌드
│   ├── metadata-registry-loader.ts
│   └── ...
├── contracts/
│   ├── registry.ts                # 계약 정의
│   └── ...
├── manifest-contract-eligibility.js
└── plugin-discovery/
```

### 로딩 단계

1. **Discovery** — `extensions/*` 스캔, 매니페스트 파싱
2. **Eligibility check** — 환경변수 / 설정 만족하는지
3. **Static load** — `api.ts` import (가벼움)
4. **Registration** — Core registry에 메타데이터 등록
5. **Lazy runtime load** — 실제 호출 시 `runtime-api.ts` import
6. **Caching** — 동일 요청에서 재사용 (LRU, 결정적 ordering)

### Lazy method binder 예

`src/plugins/runtime/index.ts` 패턴:
```typescript
const loadTtsRuntime = createLazyRuntimeModule(
  () => import("../../tts/tts.js")
);

function createRuntimeTts(): PluginRuntime["tts"] {
  const bindTtsRuntime = createLazyRuntimeMethodBinder(loadTtsRuntime);
  return {
    textToSpeech: bindTtsRuntime((runtime) => runtime.textToSpeech),
    // 첫 호출 시에만 실제 모듈 로드
  };
}
```

## Plugin SDK (`packages/plugin-sdk/`)

서드파티 플러그인 작성자에게 노출되는 공개 인터페이스:

```typescript
import {
  defineChannelPlugin,
  defineProviderPlugin,
  type InboundMessage,
  type OutboundReply,
  type ProviderRuntimeHook,
} from "openclaw/plugin-sdk";
```

### 노출 영역

- 매니페스트 빌더 (타입 안전)
- 채널/프로바이더/도구 인터페이스
- 런타임 헬퍼 (로깅, 텔레메트리, 설정 접근)
- 테스트 헬퍼 (`packages/plugin-sdk/test/`)

### Versioning
- SDK는 시맨틱 버저닝
- Breaking 변경은 메이저 bump
- Deprecated API는 한 메이저 동안 유지 + 경고

## 플러그인 경계 강제

다음 규칙은 정적 분석으로 강제됩니다 (`pnpm check:architecture`):

```
- 확장 prod 코드 → core src/** import 금지
- 확장 prod 코드 → src/plugin-sdk-internal/** import 금지
- 확장 prod 코드 → 다른 확장 src/** import 금지
- 확장 prod 코드 → 패키지 외부 상대 경로 import 금지
- core/tests → 깊은 플러그인 내부(extensions/*/src/**) import 금지
- core 테스트가 확장 specific 동작 assert 금지 (계약 테스트로 옮길 것)
```

위반은 madge / 커스텀 architecture 체크로 빌드 실패.

## 메모리 슬롯 (Single-active 패턴)

메모리 플러그인은 **슬롯 기반**: 한 번에 하나만 활성.

```
extensions/
├── active-memory/      # 옵션 1
├── memory-lancedb/     # 옵션 2
└── memory-wiki/        # 옵션 3
```

설정 (`config.memory.active`)에서 선택. Slot 변경 시:
1. 기존 메모리 플러그인 graceful shutdown
2. 새 플러그인 init
3. 호환성 마이그레이션 (필요 시)

## 매니페스트 vs 런타임 사실

`AGENTS.md:40-42`:
> Request-time runtime resolution: when a path already knows the provider id, model ref, channel id, outbound target, capability family, or attachment class, carry that as a prepared runtime fact instead of rediscovering it later.

### Bad (요청 시 재발견)
```typescript
async function send(message: string, channelId: string) {
  const allChannels = await loadOpenClawPlugins();      // 비쌈!
  const channel = allChannels.find(c => c.id === channelId);
  // ...
}
```

### Good (prepared fact)
```typescript
type PreparedChannel = { id: string; runtime: ChannelRuntimePluginHandle };
// 시작 시 한 번만 빌드
const prepared = await prepareChannelRuntime(channelId);
// hot path
async function send(message: string, prepared: PreparedChannel) {
  await prepared.runtime.send(message);
}
```

이는 hot reply / tool / outbound / media 경로의 핵심 최적화 원칙입니다.

## 새 플러그인 추가 흐름

1. `extensions/<id>/` 디렉토리 생성
2. `package.json`, `openclaw.plugin.json`, `api.ts`, `runtime-api.ts` 작성
3. `pnpm install`
4. `.github/labeler.yml`에 라벨 추가
5. GitHub repo settings에 라벨 추가
6. `AGENTS.md` 추가 시 sibling `CLAUDE.md` symlink
7. `pnpm test extensions/<id>` 통과
8. PR 생성 (라벨러가 자동 분류)
