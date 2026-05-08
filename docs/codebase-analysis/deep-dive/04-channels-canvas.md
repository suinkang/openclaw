# Deep Dive: 채널 & Canvas/A2UI (실제 코드 분석)

> 실제 `.ts` 소스 기준. 이전 문서의 일부 추정치를 정정합니다.

## 0. 정정된 사실

| 항목 | 이전 (틀림) | 실제 |
|------|------------|------|
| Discord SDK | `@buape/carbon` | **`discord-api-types` + `ws` 직접 사용** |
| iMessage | "AppleScript / Messages.app 직접 접근" | **`imsg` CLI 서브프로세스 + JSON-RPC over stdio** |
| Canvas 번들러 | (불명확) | **Rolldown** |
| A2UI 프레임워크 | (불명확) | **Lit + `@a2ui/lit` 0.9.3** |

## 1. Channel 추상화 코어 (`src/channels/`)

### 1.1 규모

**192개 구현 파일** (테스트 제외).

### 1.2 디렉토리

```
src/channels/
├── plugins/                    # 채널 플러그인 인터페이스
│   ├── types.adapters.ts
│   ├── types.core.ts
│   ├── types.plugin.ts
│   ├── target-parsing.ts
│   └── target-parsing-loaded.ts
├── message/                   # 메시지 송수신
│   ├── types.ts               # MessageReceipt, MessageDurabilityPolicy
│   ├── send.ts
│   ├── receive.ts
│   ├── contracts.ts
│   ├── capabilities.ts
│   ├── live.ts                # Live message draft preview
│   ├── reply-pipeline.ts
│   └── runtime.ts
├── turn/                      # 채널 턴 처리
├── session-envelope.ts        # 세션 envelope
├── conversation-resolution.ts # 대화 해석
├── registry.ts                # 채널 레지스트리
└── ids.ts
```

## 2. 핵심 타입

### 2.1 OutboundReplyPayload

`src/plugin-sdk/reply-payload.ts:10-16`:
```typescript
export type OutboundReplyPayload = {
  text?: string;
  mediaUrls?: string[];
  mediaUrl?: string;          // 레거시 단일 미디어 fallback
  sensitiveMedia?: boolean;
  replyToId?: string;         // 스레드/답글 ID
};
```

### 2.2 MessageReceipt

`src/channels/message/types.ts:61-71`:
```typescript
export type MessageReceipt = {
  primaryPlatformMessageId?: string;
  platformMessageIds: string[];
  parts: MessageReceiptPart[];
  threadId?: string;
  replyToId?: string;
  editToken?: string;
  deleteToken?: string;
  sentAt: number;
  raw?: readonly MessageReceiptSourceResult[];
};
```

### 2.3 MessageSendContext

`src/channels/message/types.ts:118-136`:
```typescript
export type MessageSendContext<TPayload = unknown, TSendResult = unknown> = {
  id: string;
  channel: string;
  to: string;
  accountId?: string;
  durability: Exclude<MessageDurabilityPolicy, "disabled">;
  attempt: number;
  signal: AbortSignal;
  intent?: DurableMessageSendIntent;
  previousReceipt?: MessageReceipt;
  preview?: LiveMessageState<TPayload>;
  render(): Promise<RenderedMessageBatch<TPayload>>;
  previewUpdate(rendered: RenderedMessageBatch<TPayload>): Promise<LiveMessageState<TPayload>>;
  send(rendered: RenderedMessageBatch<TPayload>): Promise<TSendResult>;
  edit(receipt: MessageReceipt, rendered: RenderedMessageBatch<TPayload>): Promise<MessageReceipt>;
  delete(receipt: MessageReceipt): Promise<void>;
  commit(receipt: MessageReceipt): Promise<void>;
  fail(error: unknown): Promise<void>;
};
```

→ 단순 send 아니라 **render → preview → send → edit → commit / fail** 라이프사이클.

### 2.4 Durability 정책

`message/types.ts:7`:
```typescript
type MessageDurabilityPolicy = "required" | "best_effort" | "disabled";
```

| 정책 | 동작 |
|------|------|
| `required` | 메시지 전송이 반드시 기록되어야. 실패 시 retry/escalate |
| `best_effort` | 최선 시도, 실패는 로깅만 |
| `disabled` | 내구성 없음 (테스트/디버깅) |

### 2.5 DurableFinalDeliveryCapabilities (14개)

```typescript
export const durableFinalDeliveryCapabilities = [
  "text", "media", "payload", "silent", "replyTo", "thread",
  "nativeQuote", "messageSendingHooks", "batch",
  "reconcileUnknownSend", "afterSendSuccess", "afterCommit",
] as const;
```

각 채널은 자기 만의 capability subset 선언.

## 3. Conversation Resolution

`src/channels/conversation-resolution.ts:21-58`:
```typescript
type ConversationResolutionSource =
  | "command-provider"
  | "focused-binding"
  | "command-fallback"
  | "inbound-provider"
  | "inbound-bundled-artifact"
  | "inbound-bundled-plugin"
  | "inbound-fallback";

type ConversationResolution = {
  canonical: {
    channel: string;
    accountId: string;
    conversationId: string;
    parentConversationId?: string;
  };
  threadId?: string;
  placementHint?: "current" | "child";
  source: ConversationResolutionSource;
};
```

7가지 source — 어디에서 대화 ID가 결정됐는지 추적 (디버깅/감사용).

## 4. Target Parsing

`src/channels/plugins/target-parsing.ts:21-31`:
```typescript
function parseWithPlugin(
  getPlugin: (channel: string) => ReturnType<typeof getChannelPlugin>,
  rawChannel: string,
  rawTarget: string,
): ParsedChannelExplicitTarget | null {
  const channel = normalizeChatChannelId(rawChannel) ?? normalizeChannelId(rawChannel);
  if (!channel) {
    return null;
  }
  return getPlugin(channel)?.messaging?.parseExplicitTarget?.({ raw: rawTarget }) ?? null;
}
```

→ 각 채널 플러그인의 `messaging.parseExplicitTarget`을 호출. Core가 형식을 모름.

## 5. Telegram (`extensions/telegram/`)

### 5.1 의존성

`extensions/telegram/package.json:7-12`:
```json
{
  "@grammyjs/runner": "^2.0.3",
  "@grammyjs/transformer-throttler": "^1.2.1",
  "grammy": "^1.42.0",
  "typebox": "1.1.37",
  "undici": "8.2.0"
}
```

### 5.2 Polling

`extensions/telegram/src/monitor.ts:26-48`:
```typescript
runner: {
  fetch: {
    timeout: 30,
    allowed_updates: resolveTelegramAllowedUpdates(),
  },
  silent: true,
  maxRetryTime: 60 * 60 * 1000,    // 1시간
  retryInterval: "exponential",
}
```

### 5.3 Webhook 모드

`monitor.ts:140-155` `startTelegramWebhook` lazy 로드. HTTP POST 수신:
- 포트, 호스트, 시크릿, 인증서 설정 가능

### 5.4 텍스트 청크 한도

```typescript
export const TELEGRAM_TEXT_CHUNK_LIMIT = 4096;  // Telegram API 제한
```

## 6. Discord (`extensions/discord/`)

### 6.1 의존성 — 정정!

`extensions/discord/package.json:10-17`:
```json
{
  "@discordjs/voice": "^0.19.2",
  "discord-api-types": "^0.38.47",
  "https-proxy-agent": "^9.0.0",
  "opusscript": "^0.1.1",
  "typebox": "1.1.37",
  "undici": "8.2.0",
  "ws": "^8.20.0"
}
```

→ **`@buape/carbon` 사용 안 함!** 직접 `discord-api-types`로 타입만 가져오고, WebSocket은 `ws`로 직접 연결.

### 6.2 라이브 메시지 capability

`extensions/discord/src/channel.ts:78-88`:
```typescript
const discordMessageAdapter = createChannelMessageAdapterFromOutbound({
  id: "discord",
  outbound: discordOutbound,
  live: {
    capabilities: {
      draftPreview: true,           // 임시 메시지 미리보기
      previewFinalization: true,    // 최종화 지원
      progressUpdates: true,        // 진행률 업데이트
    },
    finalizer: {
      capabilities: {
        finalEdit: true,
        normalFallback: true,
        discardPending: true,
      },
    },
  },
});
```

→ Discord는 **draft preview** 지원 — 응답 생성 중에 임시 메시지 표시, 완료 시 편집.

### 6.3 Probe (헬스체크)

`extensions/discord/src/channel.ts:99-153`:
```typescript
function startDiscordStartupProbe(params: {...}): void {
  void (async () => {
    const probe = await (
      await loadDiscordProbeRuntime()
    ).probeDiscord(params.token, 2500, {
      includeApplication: true,
    });
    const messageContent = probe.application?.intents?.messageContent;
    if (messageContent === "disabled") {
      params.log?.warn?.(`Discord Message Content Intent is disabled...`);
    }
  })();
}
```

→ 봇 시작 시 `Message Content Intent` 활성 여부 검증 (Discord 정책상 별도 활성 필요).

### 6.4 음성 채널

`@discordjs/voice` + `opusscript` (Opus 인코딩) → 음성 채널 오디오 통신 가능.

## 7. iMessage (`extensions/imessage/`)

### 7.1 정정 — `imsg` CLI 사용

AppleScript 직접 사용 X. **`imsg` 외부 CLI 도구**를 spawn하고 stdio로 JSON-RPC 통신.

### 7.2 클라이언트 구현

`extensions/imessage/src/client.ts:49-150`:
```typescript
export class IMessageRpcClient {
  private readonly cliPath: string;          // imsg 바이너리 경로
  private readonly dbPath?: string;          // macOS Messages DB
  private readonly runtime?: RuntimeEnv;
  private readonly onNotification?: (msg: IMessageRpcNotification) => void;
  private readonly pending = new Map<string, PendingRequest>();
  private child: ChildProcessWithoutNullStreams | null = null;
  private reader: Interface | null = null;
  private nextId = 1;

  async start(): Promise<void> {
    if (this.child) return;
    if (isTestEnv()) {
      throw new Error("Refusing to start imsg rpc in test environment");
    }
    const args = ["rpc"];
    if (this.dbPath) {
      args.push("--db", this.dbPath);
    }
    const child = spawn(this.cliPath, args, {
      stdio: ["pipe", "pipe", "pipe"],
    });
    this.child = child;
    this.reader = createInterface({ input: child.stdout });

    this.reader.on("line", (line) => {
      const trimmed = line.trim();
      if (!trimmed) return;
      this.handleLine(trimmed);   // JSON-RPC 파싱
    });
  }
}
```

### 7.3 JSON-RPC 형식

`extensions/imessage/src/client.ts:8-26`:
```typescript
export type IMessageRpcResponse<T> = {
  jsonrpc?: string;
  id?: string | number | null;
  result?: T;
  error?: IMessageRpcError;
  method?: string;
  params?: unknown;
};

export type IMessageRpcNotification = {
  method: string;
  params?: unknown;
};
```

### 7.4 stderr 진단 로깅

`client.ts:96-104`:
```typescript
child.stderr?.on("data", (chunk) => {
  const lines = chunk.toString().split(/\r?\n/);
  for (const line of lines) {
    if (!line.trim()) continue;
    this.runtime?.error?.(`imsg rpc: ${line.trim()}`);
  }
});
```

### 7.5 CLI 옵션

`extensions/imessage/package.json:26-40`:
```json
"cliAddOptions": [
  { "flags": "--db-path <path>", "description": "iMessage database path" },
  { "flags": "--service <service>", "description": "iMessage service (imessage|sms|auto)" },
  { "flags": "--region <region>", "description": "iMessage region (for SMS)" }
]
```

## 8. Canvas (`extensions/canvas/`)

### 8.1 디렉토리

```
extensions/canvas/
├── src/
│   ├── host/
│   │   ├── a2ui.ts                # A2UI HTTP 핸들러
│   │   ├── a2ui-shared.ts         # 라이브 리로드 주입
│   │   ├── a2ui-app/              # A2UI 앱 소스 (Lit)
│   │   ├── a2ui/.bundle.hash      # 번들 해시
│   │   ├── server.ts              # HTTP/WS 서버
│   │   └── file-resolver.ts       # 파일 서빙
│   ├── tool.ts                    # Canvas 도구
│   ├── a2ui-jsonl.ts
│   ├── documents.ts
│   ├── config.ts
│   └── capability.ts
├── scripts/
│   ├── bundle-a2ui.mjs            # 번들 스크립트
│   └── copy-a2ui.mjs
└── package.json
```

### 8.2 의존성

`extensions/canvas/package.json:11-16`:
```json
{
  "@a2ui/lit": "0.9.3",
  "@lit/context": "^1.1.6",
  "chokidar": "^5.0.0",
  "lit": "^3.3.2",
  "typebox": "1.1.37",
  "ws": "^8.20.0"
}
```

→ **Lit (Web Components) + `@a2ui/lit`**. React/Vue/Svelte 아님.

### 8.3 Canvas 도구 액션 (7개)

`tool.ts:20`:
```typescript
const CANVAS_ACTIONS = [
  "present",      // Canvas 표시
  "hide",         // 숨김
  "navigate",     // URL 이동
  "eval",         // JavaScript 평가
  "snapshot",     // 스크린샷
  "a2ui_push",    // A2UI 상태 푸시
  "a2ui_reset",   // 리셋
] as const;
```

### 8.4 도구 스키마

`tool.ts:119-138`:
```typescript
const CanvasToolSchema = Type.Object({
  action: stringEnum(CANVAS_ACTIONS),
  gatewayUrl: Type.Optional(Type.String()),
  gatewayToken: Type.Optional(Type.String()),
  timeoutMs: Type.Optional(Type.Number()),
  node: Type.Optional(Type.String()),
  target: Type.Optional(Type.String()),
  x: Type.Optional(Type.Number()),
  y: Type.Optional(Type.Number()),
  width: Type.Optional(Type.Number()),
  height: Type.Optional(Type.Number()),
  url: Type.Optional(Type.String()),
  javaScript: Type.Optional(Type.String()),
  outputFormat: optionalStringEnum(CANVAS_SNAPSHOT_FORMATS),
  maxWidth: Type.Optional(Type.Number()),
  quality: Type.Optional(Type.Number()),
  delayMs: Type.Optional(Type.Number()),
  jsonl: Type.Optional(Type.String()),
  jsonlPath: Type.Optional(Type.String()),
});
```

→ 위치(x, y, width, height) + 콘텐츠(url, javaScript, jsonl) + 출력(snapshot 형식, quality).

## 9. A2UI HTTP 호스트 서버

### 9.1 경로

`extensions/canvas/src/host/a2ui-shared.ts:3-7`:
```typescript
export const A2UI_PATH = "/__openclaw__/a2ui";
export const CANVAS_HOST_PATH = "/__openclaw__/canvas";
export const CANVAS_WS_PATH = "/__openclaw__/ws";
```

### 9.2 요청 핸들러

`a2ui.ts:77-111`:
```typescript
export async function handleA2uiHttpRequest(
  req: IncomingMessage,
  res: ServerResponse,
): Promise<boolean> {
  const url = new URL(req.url, "http://localhost");
  const basePath = isA2uiPath(url.pathname) ? A2UI_PATH : undefined;
  if (!basePath) return false;
  if (req.method !== "GET" && req.method !== "HEAD") {
    res.statusCode = 405;
    res.end("Method Not Allowed");
    return true;
  }
  const a2uiRootReal = await resolveA2uiRootReal();
  if (!a2uiRootReal) {
    res.statusCode = 503;
    res.end("A2UI assets not found");
    return true;
  }
  const rel = url.pathname.slice(basePath.length);
  const result = await resolveFileWithinRoot(a2uiRootReal, rel || "/");
  // ... 파일 서빙
}
```

캐시: `Cache-Control: no-store` (라이브 리로드 위해).

### 9.3 라이브 리로드 주입

`a2ui-shared.ts:54-62` 클라이언트에 주입되는 JS:
```javascript
const cap = new URLSearchParams(location.search).get("oc_cap");
const proto = location.protocol === "https:" ? "wss" : "ws";
const capQuery = cap ? "?oc_cap=" + encodeURIComponent(cap) : "";
const ws = new WebSocket(proto + "://" + location.host + "/__openclaw__/ws" + capQuery);
ws.onmessage = (ev) => {
  if (String(ev.data || "") === "reload") location.reload();
};
```

→ 단순한 메커니즘. 서버가 `"reload"` 문자열만 보내면 페이지 전체 reload.

### 9.4 모바일 브릿지 (iOS/Android)

`a2ui-shared.ts:14-52`:
```javascript
const handlerNames = ["openclawCanvasA2UIAction"];
function postToNode(payload) {
  const raw = typeof payload === "string" ? payload : JSON.stringify(payload);
  // iOS: WebKit message handlers
  const iosHandler = globalThis.webkit?.messageHandlers?.[name];
  if (iosHandler && typeof iosHandler.postMessage === "function") {
    iosHandler.postMessage(raw);
    return true;
  }
  // Android: native bridge
  const androidHandler = globalThis[name];
  if (androidHandler && typeof androidHandler.postMessage === "function") {
    androidHandler.postMessage(raw);
    return true;
  }
}
```

| 플랫폼 | 브릿지 |
|--------|--------|
| iOS | `window.webkit.messageHandlers.openclawCanvasA2UIAction.postMessage(...)` |
| Android | `window.openclawCanvasA2UIAction.postMessage(...)` |
| Web | WebSocket fallback |

## 10. A2UI 번들 (Rolldown!)

### 10.1 번들 스크립트

`extensions/canvas/scripts/bundle-a2ui.mjs`:

`computeHash()` (라인 15-16, 193-200) — SHA256 해시 생성:
```javascript
const hashFile = path.join(pluginDir, "src", "host", "a2ui", ".bundle.hash");
const outputFile = path.join(pluginDir, "src", "host", "a2ui", "a2ui.bundle.js");

async function computeHash() {
  let files = listTrackedInputFiles();   // package.json, pnpm-lock.yaml, a2ui-app/
  const hash = createHash("sha256");
  for (const filePath of files) {
    hash.update(normalizePath(path.relative(rootDir, filePath)));
    hash.update("\0");
    hash.update(await fs.readFile(filePath));
    hash.update("\0");
  }
  return hash.digest("hex");
}

const currentHash = await computeHash();
if (await pathExists(hashFile)) {
  const previousHash = (await fs.readFile(hashFile, "utf8")).trim();
  if (previousHash === currentHash && hasOutputFile) {
    console.log("A2UI bundle up to date; skipping.");
    return;
  }
}
```

해시 입력:
- `package.json`
- `pnpm-lock.yaml`
- `a2ui-app/` 전체

→ 의존성이나 소스 변경되지 않으면 번들 스킵. 빌드 시간 절약.

### 10.2 Rolldown 사용

`bundle-a2ui.mjs:49-63`:
```javascript
export function getLocalRolldownCliCandidates(repoRoot = rootDir) {
  return [
    path.join(repoRoot, "node_modules", "rolldown", "bin", "cli.mjs"),
    path.join(repoRoot, "node_modules", ".pnpm", "node_modules", "rolldown", "bin", "cli.mjs"),
  ];
}
```

→ **Rolldown** (Rust 기반 Rollup 후속). esbuild/webpack/vite 아님.

### 10.3 호출

`extensions/canvas/package.json:22-24`:
```json
"assetScripts": {
  "build": "node scripts/bundle-a2ui.mjs",
  "copy": "node scripts/copy-a2ui.mjs"
}
```

`pnpm canvas:a2ui:bundle` 호출 시 실행.

## 11. Skills (`skills/`)

### 11.1 매니페스트 형식 — `SKILL.md` (Markdown + YAML frontmatter)

`skills/nano-pdf/SKILL.md`:
```yaml
---
name: nano-pdf
description: Edit PDFs with natural-language instructions using the nano-pdf CLI.
homepage: https://pypi.org/project/nano-pdf/
metadata:
  openclaw:
    emoji: "📄"
    requires:
      bins: ["nano-pdf"]
    install:
      - id: uv
        kind: uv
        package: nano-pdf
        bins: ["nano-pdf"]
        label: "Install nano-pdf (uv)"
---
```

본문은 사용 설명서 (Markdown).

### 11.2 설치 백엔드

`metadata.openclaw.install[].kind`:
- `uv` (Python via uv)
- `npm`
- `brew`
- 기타

→ 온보딩 시 자동 설치 가능.

### 11.3 검증

`requires.bins` — 필수 CLI 바이너리.
`requires.env` — 필수 환경변수.

스킬 활성화 시 OpenClaw가 검증.

## 12. MCP 통합

### 12.1 설정 타입

`src/config/types.mcp.ts`:
```typescript
export type McpServerConfig = {
  // Stdio 전송
  command?: string;
  args?: string[];
  env?: Record<string, string | number | boolean>;
  cwd?: string;
  workingDirectory?: string;

  // HTTP 전송
  url?: string;
  transport?: "sse" | "streamable-http";
  headers?: Record<string, string | number | boolean>;

  connectionTimeoutMs?: number;
  [key: string]: unknown;
};

export type McpConfig = {
  servers?: Record<string, McpServerConfig>;
  sessionIdleTtlMs?: number;     // 기본: 10분
};
```

### 12.2 도구 타입

`src/tools/types.ts`:
```typescript
| { readonly kind: "mcp"; readonly serverId: string };
| { readonly kind: "mcp"; readonly serverId: string; readonly toolName: string };
```

### 12.3 흐름

```
User config → ~/.openclaw/config.json (channels.mcp.servers)
  ↓
/mcp 명령으로 활성화 (commands.mcp: true)
  ↓
Stdio: spawn(command, args)  /  HTTP: connect via SSE/streamable
  ↓
list_tools() RPC → 사용 가능 도구 목록
  ↓
LLM이 도구 호출 → call_tool(name, args)
  ↓
세션별 번들 MCP 런타임, 10분 idle 후 cleanup
```

## 13. Update Offset Tracking (Telegram)

`update-offset-runtime-api.js`:
- 마지막 처리된 update ID 저장
- 재시작 시 다음 ID부터 `getUpdates(offset=N)`
- 중복 처리 방지

레이스 조건: `TelegramPollingSession` 리스 메커니즘으로 동시 polling 방지.

## 14. Capability vs Implementation

### 14.1 채널 어댑터 인터페이스

```typescript
export type ChannelOutboundAdapter = {
  sendText?: (ctx: ChannelMessageSendTextContext) => Promise<MessageReceipt>;
  sendMedia?: (ctx: ChannelMessageSendMediaContext) => Promise<MessageReceipt>;
  sendPayload?: (ctx: ChannelMessageSendPayloadContext) => Promise<MessageReceipt>;
  // ... 기타
};
```

각 채널은 **자기가 지원하는 메서드만 구현**. Core는 capability flag로 감지.

### 14.2 메시지 어댑터 (확장)

```typescript
export type ChannelMessageAdapter = ChannelOutboundAdapter & {
  receive?: ChannelMessageReceiveAdapterShape;
  live?: {
    capabilities: { draftPreview?: boolean; ... };
    finalizer?: { capabilities: { finalEdit?: boolean; ... } };
  };
};
```

## 15. 정리

| 항목 | 실제 |
|------|------|
| 채널 코어 파일 수 | 192 (테스트 제외) |
| 메시지 라이프사이클 | render → preview → send → edit → commit/fail |
| Durability 정책 | required / best_effort / disabled |
| Telegram | grammy, undici, runner, throttler |
| Discord | **discord-api-types 직접** (carbon 안 씀) + ws |
| iMessage | **`imsg` CLI + JSON-RPC over stdio** |
| Canvas 번들러 | **Rolldown** |
| Canvas 프레임워크 | **Lit + @a2ui/lit 0.9.3** |
| HTTP 경로 | `/__openclaw__/{a2ui,canvas,ws}` |
| 캐시 | `no-store` (live reload) |
| 모바일 브릿지 | iOS WebKit + Android JS |
| A2UI 번들 캐싱 | SHA256 해시, package.json/pnpm-lock/a2ui-app 입력 |
| Skills 형식 | `SKILL.md` (Markdown + YAML frontmatter) |
| MCP 전송 | stdio + HTTP(sse/streamable-http) |
| MCP idle TTL | 기본 10분 |

## 16. 흥미로운 디자인 결정

### 16.1 Discord가 Carbon을 안 쓰는 이유 (추측)
- 직접 제어 (음성 채널, intent 검증 등)
- 의존성 최소화
- License 이슈?

### 16.2 iMessage CLI 분리
- macOS API는 Swift/ObjC 필요
- TypeScript에서 직접 접근 불가
- → 별도 `imsg` CLI 도구로 추상화
- 결과: macOS 의존성 격리, 다른 OS에서 mock 가능

### 16.3 Lit + Web Components 선택
- React/Vue 대비 번들 크기 작음
- iOS/Android WebView 호환성 (브릿지 단순)
- Standards-based (장기 안정성)

### 16.4 Rolldown
- Rolldown은 Vite팀이 개발 중인 차세대 번들러
- esbuild보다 코드분할 우수, Rollup 호환
- A2UI bundle은 한 파일이라 esbuild로 충분할 수도 있지만, 차세대 도구로 검증

## 17. 미확인 / 한계

- **Channel hot-reload** — 채널 플러그인 단위 reload 지원 미확인
- **iMessage 양방향** — 보내기는 분명, 받기 (notification 수신) 메커니즘 추가 분석 필요
- **A2UI 컴포넌트 스펙** — `@a2ui/lit` 패키지 자체 분석 필요 (`a2ui-app/` 깊이 탐색 미수행)
- **Canvas WebSocket 인증** — `oc_cap` query 파라미터의 검증 방식 미확인
