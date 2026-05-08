# Deep Dive: 플러그인 로더 & SDK (실제 코드 분석)

> 실제 `.ts` 소스 기준. 모든 인용은 검증된 파일 경로.

## 1. 매니페스트 발견 알고리즘

### 1.1 매니페스트 파일 이름

`src/plugins/manifest.ts:34-36`:
```typescript
export const PLUGIN_MANIFEST_FILENAME = "openclaw.plugin.json";
export const PLUGIN_MANIFEST_FILENAMES = [PLUGIN_MANIFEST_FILENAME] as const;
export const MAX_PLUGIN_MANIFEST_BYTES = 256 * 1024;  // 256KB
```

### 1.2 캐시 — LRU 512

```typescript
const pluginManifestLoadCache = new PluginLruCache<PluginManifestLoadCacheEntry>(
  MAX_PLUGIN_MANIFEST_LOAD_CACHE_ENTRIES,  // = 512
);
```

### 1.3 파싱 — JSON5

매니페스트는 **JSON5**로 파싱됨 (주석 + unquoted key 허용). `zod`나 `ajv` 아닌 커스텀 타입 체크.

### 1.4 보안 검증

`src/plugins/discovery.ts:111-232`:

| 검증 | 목적 |
|------|------|
| `checkSourceEscapesRoot` | path traversal 방지 |
| 파일 권한 0o002 비트 검사 | 그룹/기타 쓰기 권한 거부 |
| POSIX 소유권 (UID) 검증 | bundled 외 플러그인의 신뢰 검증 |
| 심볼릭 링크 거부 | 우회 방지 |
| Hardlink 거부 | 우회 방지 |
| 256KB 크기 제한 | DoS 방지 |

`readRootStructuredFileSync` 사용 — 경계 파일 읽기로 path traversal 방어.

## 2. 매니페스트 타입

`src/plugins/manifest.ts:54+`:
```typescript
export type PluginManifest = {
  id: string;
  channels?: string[];
  providers?: string[];
  channelEnvVars?: Record<string, string[]>;
  configSchema?: JsonSchemaObject;
  channelConfigs?: Record<string, PluginManifestChannelConfig>;
  activation?: PluginManifestActivation;
  setup?: PluginManifestSetup;
  // ... 추가 필드
};

export type PluginManifestChannelConfig = {
  schema: JsonSchemaObject;
  uiHints?: Record<string, PluginConfigUiHint>;
  runtime?: ChannelConfigRuntimeSchema;
  label?: string;
  commands?: PluginManifestChannelCommandDefaults;
};
```

## 3. 레지스트리 구현

### 3.1 레지스트리 타입

`src/plugins/registry-types.ts`:
```typescript
export type PluginRegistry = {
  plugins?: PluginRecord[];
  channels?: PluginChannelRegistration[];
  providers?: PluginProviderRegistration[];
  commands?: PluginCommandRegistration[];
  [key: string]: unknown;
};

export type PluginRecord = {
  id: string;
  status?: "loaded" | "error";
  error?: string;
};
```

→ 순서 유지 배열. 정렬 안 됨 (manifest 활성화 순서 보존).

### 3.2 활성 런타임 레지스트리

`src/plugins/active-runtime-registry.ts:1-106`:
```typescript
export function getLoadedRuntimePluginRegistry(
  params?: {
    env?: NodeJS.ProcessEnv;
    loadOptions?: PluginLoadOptions;
    workspaceDir?: string;
    requiredPluginIds?: readonly string[];
    surface?: ActiveRuntimePluginRegistrySurface;
  }
): PluginRegistry | undefined
```

Hot path는 이 함수로 활성 레지스트리만 조회 (전체 스캔 X).

## 4. Lazy Loading 실제 구현

### 4.1 모듈 로더

`src/shared/lazy-runtime.ts`:
```typescript
export function createLazyRuntimeModule<TModule>(
  importer: () => Promise<TModule>,
): () => Promise<TModule> {
  return createLazyRuntimeSurface(importer, (module) => module);
}

export function createLazyRuntimeMethodBinder<TSurface>(load: () => Promise<TSurface>) {
  return function <TArgs extends unknown[], TResult>(
    select: (surface: TSurface) => (...args: TArgs) => TResult,
  ): (...args: TArgs) => Promise<Awaited<TResult>> {
    return createLazyRuntimeMethod(load, select);
  };
}
```

### 4.2 사용 예 — TTS 런타임

`src/plugins/runtime/index.ts`:
```typescript
const loadTtsRuntime = createLazyRuntimeModule(() => import("../../tts/tts.js"));
const bindTtsRuntime = createLazyRuntimeMethodBinder(loadTtsRuntime);

const tts = {
  textToSpeech: bindTtsRuntime((runtime) => runtime.textToSpeech),
  // ...
};
```

→ 첫 호출 시점에 import 실행. 모듈은 캐시됨.

## 5. Plugin SDK Surface

### 5.1 50+ 서브패스 export

`package.json` exports 일부:
```json
{
  "./plugin-sdk": "dist/plugin-sdk/index.d.ts",
  "./plugin-sdk/core": "dist/plugin-sdk/core.d.ts",
  "./plugin-sdk/plugin-entry": "dist/plugin-sdk/plugin-entry.d.ts",
  "./plugin-sdk/provider-entry": "dist/plugin-sdk/provider-entry.d.ts",
  "./plugin-sdk/channel-entry-contract": "dist/plugin-sdk/channel-entry-contract.d.ts",
  "./plugin-sdk/channel-core": "dist/plugin-sdk/channel-core.d.ts",
  "./plugin-sdk/provider-auth": "dist/plugin-sdk/provider-auth.d.ts",
  "./plugin-sdk/provider-model-shared": "dist/plugin-sdk/provider-model-shared.d.ts",
  "./plugin-sdk/provider-usage": "dist/plugin-sdk/provider-usage.d.ts"
  // ... 50+개
}
```

각 서브패스는 단일 책임. 플러그인이 core 내부 직접 접근 못 함.

### 5.2 Plugin Entry 헬퍼

`src/plugin-sdk/plugin-entry.ts`:
```typescript
export function definePluginEntry(
  definition: OpenClawPluginDefinition
): OpenClawPluginModule {
  return {
    default: {
      id: definition.id,
      name: definition.name,
      description: definition.description,
      register: definition.register,
    },
  };
}
```

### 5.3 Channel Entry 헬퍼

`src/plugin-sdk/channel-entry-contract.ts`:
```typescript
export function defineBundledChannelEntry(
  definition: BundledChannelPluginDefinition
): BundledChannelEntry { ... }
```

## 6. api.ts vs runtime-api.ts 패턴

### 6.1 분리 목적

- **`api.ts`** — 가벼움. 메타데이터/타입만. 즉시 로드 OK
- **`runtime-api.ts`** — 무거움. 외부 SDK import. lazy 로드 필요

### 6.2 Telegram 케이스

**`extensions/telegram/api.ts`** (1-185줄):
```typescript
export { telegramPlugin } from "./src/channel.js";
export { telegramSetupPlugin } from "./src/channel.setup.js";
export { inspectTelegramAccount, type TelegramCredentialStatus } from "./src/account-inspect.js";
export { resolveTelegramAccount, type ResolvedTelegramAccount } from "./src/accounts.js";
// ... 180+ exports (모두 타입/메타데이터/설정 셋업)
```

**`extensions/telegram/runtime-api.ts`** (1-97줄):
```typescript
export { auditTelegramGroupMembership } from "./src/audit.js";
export { probeTelegram } from "./src/probe.js";
export { monitorTelegramProvider } from "./src/monitor.js";
export { sendMessageTelegram } from "./src/send.js";
export { resolveTelegramFetch, resolveTelegramTransport } from "./src/fetch.js";
```

→ runtime-api는 grammy SDK가 필요한 함수들만.

## 7. Telegram 플러그인 구체 분석

### 7.1 매니페스트

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
    "additionalProperties": false,
    "properties": {}
  }
}
```

→ `activation.onStartup: false`로 lazy. 부팅 시 자동 로드되지 않음.

### 7.2 NPM 의존성

`extensions/telegram/package.json:7-12`:
```json
"dependencies": {
  "@grammyjs/runner": "^2.0.3",
  "@grammyjs/transformer-throttler": "^1.2.1",
  "grammy": "^1.42.0",
  "typebox": "1.1.37",
  "undici": "8.2.0"
}
```

| 패키지 | 역할 |
|--------|------|
| `grammy` | Telegram Bot API 클라이언트 (TypeScript-first) |
| `@grammyjs/runner` | 폴링/웹훅 러너 |
| `@grammyjs/transformer-throttler` | 레이트 제한 자동화 |
| `undici` | HTTP 클라이언트 (Node 표준) |
| `typebox` | JSON Schema 생성 |

### 7.3 Polling 설정

`extensions/telegram/src/monitor.ts:26-48`:
```typescript
export function createTelegramRunnerOptions(cfg: OpenClawConfig): RunOptions<unknown> {
  return {
    sink: {
      concurrency: resolveAgentMaxConcurrent(cfg),
    },
    runner: {
      fetch: {
        timeout: 30,                   // 30초 long polling
        allowed_updates: resolveTelegramAllowedUpdates(),
      },
      silent: true,
      maxRetryTime: 60 * 60 * 1000,    // 1시간 최대 재시도
      retryInterval: "exponential",
    },
  };
}
```

### 7.4 Outbound 송신 시그니처

`extensions/telegram/src/send.ts:68-107`:
```typescript
type TelegramSendOpts = {
  cfg: OpenClawConfig;
  token?: string;
  accountId?: string;
  verbose?: boolean;
  mediaUrl?: string;
  mediaLocalRoots?: readonly string[];
  mediaReadFile?: (filePath: string) => Promise<Buffer>;
  gatewayClientScopes?: readonly string[];
  maxBytes?: number;
  api?: TelegramApiOverride;
  retry?: RetryConfig;
  textMode?: "markdown" | "html";
  plainText?: string;
  asVoice?: boolean;
  asVideoNote?: boolean;
  silent?: boolean;
  replyToMessageId?: number;
  quoteText?: string;
  messageThreadId?: number;
  buttons?: TelegramInlineButtons;
  forceDocument?: boolean;
};
```

특이 옵션:
- `messageThreadId` — Telegram Forum topic 지원
- `forceDocument` — 압축 회피 (원본 품질 유지)
- `asVoice` / `asVideoNote` — 특수 메시지 타입

텍스트 청크 한도:
```typescript
export const TELEGRAM_TEXT_CHUNK_LIMIT = 4096;  // Telegram API 제한
```

## 8. Anthropic Provider 플러그인

### 8.1 NPM 의존성 (놀라운 부분!)

`extensions/anthropic/package.json:8`:
```json
"dependencies": {
  "@mariozechner/pi-ai": "^0.73.0"
}
```

→ **`@anthropic-ai/sdk` 사용 안 함!** 대신 `@mariozechner/pi-ai`라는 추상화 레이어 사용. 이는 OpenAI/Anthropic/Google 모두 단일 인터페이스로 다룰 수 있게 해주는 라이브러리.

### 8.2 SDK Imports

`extensions/anthropic/register.runtime.ts:1-45`:
```typescript
import type { OpenClawPluginApi, ProviderAuthContext } from "openclaw/plugin-sdk/plugin-entry";
import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth";
import { cloneFirstTemplateModel } from "openclaw/plugin-sdk/provider-model-shared";
import { fetchClaudeUsage } from "openclaw/plugin-sdk/provider-usage";
```

→ Plugin SDK가 인증/모델/사용량 헬퍼를 표준화. 각 Provider 플러그인이 자기 만의 헬퍼를 만들 필요 없음.

### 8.3 Stream Wrapper

`extensions/anthropic/stream-wrappers.ts`:
```typescript
export async function wrapAnthropicProviderStream(params: {
  stream: Stream<MessageStreamEvent>;
  originalPrompt: string;
  ctx: ProviderWrapStreamFnContext;
}): Promise<MessageStreamAdapter> {
  // 텍스트 청크 수집
  // usage 추적
  // thinking 블록 처리
  // tool_use 정규화
  return adaptStream(stream);
}
```

### 8.4 매니페스트

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

## 9. 플러그인 활성화 단계

`src/plugins/`의 단계별 처리:

| 단계 | 파일 | 동작 |
|------|------|------|
| 1. Discovery | `discovery.ts` | `extensions/*` 스캔, 보안 검증 |
| 2. Manifest load | `manifest.ts` | JSON5 파싱, 256KB 제한 |
| 3. Eligibility | `manifest-contract-eligibility.js` | 환경변수/설정 만족 여부 |
| 4. Activation plan | `activation-planner.ts` | 어떤 순서로 로드할지 |
| 5. Static load | `loader.ts` | `api.ts` import |
| 6. Registry update | `registry.ts` | 메타데이터 등록 |
| 7. Lazy runtime | (사용 시) | `runtime-api.ts` import |

## 10. 의존성 그래프

명시적 DAG 없음. 대신:
- 매니페스트에서 `onProviders`, `onChannels`, `onConfigPaths`로 활성화 의존성 선언
- 순환 import는 `pnpm check:import-cycles`로 검증
- 플러그인 간 직접 import 금지 (`pnpm check:architecture`)

## 11. 발견 사항 정리

| 항목 | 실제 |
|------|------|
| 매니페스트 파일명 | `openclaw.plugin.json` |
| 매니페스트 크기 한도 | 256KB |
| 매니페스트 캐시 | LRU 512 entries |
| 매니페스트 파싱 | JSON5 |
| 매니페스트 검증 | 커스텀 타입 체크 (Zod 아님) |
| 보안 | path traversal, POSIX 권한, symlink/hardlink 거부 |
| Lazy 로더 | `createLazyRuntimeModule`, `createLazyRuntimeMethodBinder` 실제 존재 |
| Telegram SDK | `grammy` 1.42 |
| Anthropic SDK | `@mariozechner/pi-ai` (NOT `@anthropic-ai/sdk`!) |
| SDK 서브패스 | 50+ |
| 플러그인 격리 | V8 isolate 공유 (격리 X) |

## 12. 한계 / 미확인

- **버전 호환성** — SDK ↔ 플러그인 버전 핸드셰이크 메커니즘 불명확
- **핫 리로드** — 플러그인 단위 hot reload 지원 여부 미확인
- **격리** — 모든 플러그인이 같은 프로세스/V8 isolate. 신뢰 모델은 manifest 검증 + POSIX 권한에 의존
