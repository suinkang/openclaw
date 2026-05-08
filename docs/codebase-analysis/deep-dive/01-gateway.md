# Deep Dive: Gateway 서브시스템 (실제 코드 분석)

> 이 문서는 `AGENTS.md` 가이드가 아닌 **실제 `.ts` 소스 코드**를 직접 읽어 정리한 것입니다. 모든 인용은 실제 파일 경로와 라인 번호 기준.

## 1. 진입점 & Lazy Boot

### 1.1 진입점

`src/gateway/server.ts:24-29`:
```typescript
export async function startGatewayServer(
  port = 18789,
  opts: GatewayServerOptions = {},
): Promise<GatewayServer>
```

`server.ts`는 매우 경량입니다. 실제 로직은 `server.impl.ts`에 있고, **동적 import**로 로드됩니다 (`server.ts:13-21`). 이는 "startup trace 측정"을 가능하게 하기 위함.

### 1.2 기본 포트
- **18789** (`startGatewayServer` 기본값)

## 2. WebSocket 라이브러리

`src/gateway/server/ws-connection.ts:3`:
```typescript
import type { RawData, WebSocket, WebSocketServer } from "ws";
```

→ **`ws` 라이브러리** (Node.js 표준). `socket.io`나 `µWS` 아님.

## 3. 프로토콜 스키마: TypeBox + AJV

### 3.1 TypeBox로 스키마 선언, AJV로 컴파일된 검증

`src/gateway/protocol/index.ts`:
```typescript
const ajv = new (AjvPkg as unknown as new (opts?: object) => import("ajv").default)({
  allErrors: true,
  strict: false,
  removeAdditional: false,
});
```

검증 함수 종류:
- `validateRequestFrame`
- `validateConnectParams`
- `validateResponseFrame`
- `validateEventFrame`

각 함수는 AJV `compile()`로 미리 컴파일됨 → 핫 패스 빠름.

### 3.2 3가지 프레임 타입

`src/gateway/protocol/schema/frames.ts`:
```typescript
export const RequestFrameSchema = Type.Object({
  type: Type.Literal("req"),
  id: NonEmptyString,           // 응답 매칭용 client-generated ID
  method: NonEmptyString,       // RPC 메서드명
  params: Type.Optional(Type.Unknown()),
});

export const ResponseFrameSchema = Type.Object({
  type: Type.Literal("res"),
  id: NonEmptyString,
  ok: Type.Boolean(),
  payload: Type.Optional(Type.Unknown()),
  error: Type.Optional(ErrorShapeSchema),
});

export const EventFrameSchema = Type.Object({
  type: Type.Literal("event"),
  event: NonEmptyString,
  payload: Type.Optional(Type.Unknown()),
  seq: Type.Optional(Type.Integer({ minimum: 0 })),
  stateVersion: Type.Optional(StateVersionSchema),
});
```

### 3.3 상태 버전 추적

`stateVersion`은 presence와 health 상태의 단조 증가 카운터:
```typescript
type GatewayBroadcastStateVersion = {
  presence?: number;
  health?: number;
};
```

클라이언트가 stale 이벤트 무시 가능.

## 4. 인증 — 6가지 모드

`src/gateway/auth.ts:35-51`:
```typescript
export type GatewayAuthResult = {
  ok: boolean;
  method?:
    | "none"
    | "token"
    | "password"
    | "tailscale"
    | "device-token"
    | "bootstrap-token"
    | "trusted-proxy";
  user?: string;
  reason?: string;
  rateLimited?: boolean;
  retryAfterMs?: number;
};
```

### 4.1 인증 결정 흐름

`src/gateway/auth.ts:388-421` `authorizeGatewayConnect`:

1. **Rate limit 체크** — IP별 토큰 시도 횟수, 실패 시 `retryAfterMs`
2. **Tailscale auth** — WS Control UI에서만, `resolveVerifiedTailscaleUser`로 확인
3. **Token/password auth** — `safeEqualSecret` (constant-time 비교)로 타이밍 공격 방어
4. **Device token auth** — `verifyDeviceToken`, 디바이스 공개키 기반
5. **Bootstrap token auth** — `verifyDeviceBootstrapToken`, 페어링 단계 전용
6. **Trusted proxy** — `X-Forwarded-*` 헤더 검증 (proxy 설정 필요)

### 4.2 페어링 함수

`src/gateway/server/ws-connection/message-handler.ts` 임포트:
```typescript
import {
  approveDevicePairing,
  ensureDeviceToken,
  getPairedDevice,
  listDevicePairing,
  requestDevicePairing,
} from "../../../infra/device-pairing.js";
import { reconcileNodePairingOnConnect } from "../../node-connect-reconcile.js";
```

## 5. 연결 라이프사이클

### 5.1 연결 즉시 — Challenge 발송

`src/gateway/server/ws-connection.ts:196-494`:
```typescript
wss.on("connection", (socket, upgradeReq) => {
  const connId = randomUUID();
  // ...
  const connectNonce = randomUUID();
  send({
    type: "event",
    event: "connect.challenge",
    payload: { nonce: connectNonce, ts: Date.now() },
  });
```

### 5.2 Preauth Budget — DoS 방어

인증 전 최대 연결 수 제한. `preauthConnectionBudget.release(preauthBudgetKey)` 인증 성공 시 릴리즈.

### 5.3 핸드셰이크 타임아웃

```typescript
const handshakeTimeoutMs = resolvePreauthHandshakeTimeoutMs({...});
const handshakeTimer = setTimeout(() => {
  if (!client) {
    setCloseCause("handshake-timeout", {...});
    close();
  }
}, handshakeTimeoutMs);
```

### 5.4 메시지 핸들러 온디맨드 로딩

```typescript
attachGatewayWsMessageHandlerOnDemand({...});

const queued: RawData[] = [];
const queueMessage = (data: RawData) => {
  if (queued.length >= MAX_QUEUED_MESSAGE_HANDLER_FRAMES) {  // = 16
    params.close(1008, "gateway message handler loading");
  }
  queued.push(data);
};
```

핸들러 로딩 중 도착한 메시지는 큐에 (최대 16). 초과 시 1008 close.

### 5.5 Ping/Pong 25초

```typescript
pingTimer = setInterval(() => {
  socket.ping();
}, 25_000);
```

## 6. 메시지 처리 — 단계별 trace

`src/gateway/server/ws-connection/message-handler.ts:341-495`

### Step 1: 페이로드 크기 검증
```typescript
const preauthPayloadBytes = !getClient() ? getRawDataByteLength(data) : undefined;
if (preauthPayloadBytes > MAX_PREAUTH_PAYLOAD_BYTES) {
  setCloseCause("preauth-payload-too-large", {...});
  close(1009, "preauth payload too large");
  return;
}
```

### Step 2: JSON 파싱 및 프레임 추출
```typescript
const text = rawDataToString(data);
const parsed = JSON.parse(text);
const frameType = typeof parsed.type === "string" ? String(parsed.type) : undefined;
const frameMethod = typeof parsed.method === "string" ? String(parsed.method) : undefined;
```

### Step 3: 핸드셰이크 (첫 메시지)

```typescript
if (!client) {
  // 반드시: { type:"req", method:"connect", params: ConnectParams }
  if (!isRequestFrame || parsed.method !== "connect" ||
      !validateConnectParams(parsed.params)) {
    sendHandshakeErrorResponse(
      ErrorCodes.INVALID_REQUEST,
      `invalid connect params: ${formatValidationErrors(validateConnectParams.errors)}`
    );
    close(1008, "invalid handshake");
    return;
  }
```

### Step 4: 프로토콜 버전 협상

`message-handler.ts:480-495`:
```typescript
const { minProtocol, maxProtocol } = connectParams;
if (maxProtocol < PROTOCOL_VERSION || minProtocol > PROTOCOL_VERSION) {
  markHandshakeFailure("protocol-mismatch", {
    minProtocol,
    maxProtocol,
    expectedProtocol: PROTOCOL_VERSION,
  });
  sendHandshakeErrorResponse(ErrorCodes.INVALID_REQUEST, "protocol mismatch");
}
```

클라이언트는 자신이 지원하는 `[minProtocol, maxProtocol]` 범위를 보내고, 서버의 `PROTOCOL_VERSION`이 그 범위 내에 있어야 함.

### Step 5: Hello-OK 응답

성공 시 서버는 다음 응답 송신:
```typescript
send({
  type: "res",
  id: frame.id,
  ok: true,
  payload: {
    type: "hello-ok",
    protocol: PROTOCOL_VERSION,
    server: { version, connId },
    features: { methods, events },
    snapshot: buildGatewaySnapshot(),
    auth: { deviceToken?, role, scopes },
    policy: { maxPayload, maxBufferedBytes, tickIntervalMs },
  },
});
```

`features.methods`로 클라이언트가 사용 가능한 RPC 메서드 목록 알림 (플러그인이 추가한 메서드 포함).

### Step 6: 일반 RPC 처리

```typescript
if (client && isRequestFrame) {
  const method = parsed.method;
  if (!gatewayMethods.includes(method)) {
    respond(false, null, errorShape(ErrorCodes.UNKNOWN_METHOD, `unknown method: ${method}`));
    return;
  }
  const handler = extraHandlers[method];
  if (!handler) {
    respond(false, null, errorShape(ErrorCodes.INTERNAL, "no handler"));
    return;
  }
  await handler({
    req: parsed,
    params: parsed.params ?? {},
    client,
    isWebchatConnect: (p) => isWebchatClient(p?.client),
    respond,
    context: gatewayRequestContext,
  });
}
```

## 7. 클라이언트 트래킹

`src/gateway/server/ws-types.ts`:
```typescript
export type GatewayWsClient = PluginNodeCapabilityClient & {
  socket: WebSocket;
  connect: ConnectParams;
  connId: string;
  isDeviceTokenAuth?: boolean;
  usesSharedGatewayAuth: boolean;
  sharedGatewaySessionGeneration?: string;
  presenceKey?: string;
  clientIp?: string;
};
```

전체 클라이언트 집합:
```typescript
const clients: Set<GatewayWsClient> = new Set();
```

## 8. 요청 컨텍스트 (Handler 인자)

`src/gateway/server-methods/shared-types.ts:42-120` `GatewayRequestContext`:
```typescript
export type GatewayRequestContext = {
  deps: CliDeps;
  cron: CronServiceContract;
  cronStorePath: string;
  getRuntimeConfig: () => OpenClawConfig;
  execApprovalManager?: ExecApprovalManager;
  pluginApprovalManager?: ExecApprovalManager<PluginApprovalRequestPayload>;
  loadGatewayModelCatalog: (params?: { readOnly?: boolean }) => Promise<ModelCatalogEntry[]>;
  getHealthCache: () => HealthSummary | null;
  refreshHealthSnapshot: (opts?: {...}) => Promise<HealthSummary>;

  // Broadcast
  broadcast: GatewayBroadcastFn;
  broadcastToConnIds: GatewayBroadcastToConnIdsFn;

  // Session events
  subscribeSessionEvents: (connId: string) => void;
  unsubscribeSessionEvents: (connId: string) => void;
  subscribeSessionMessageEvents: (connId: string, sessionKey: string) => void;

  // Node (remote device) communication
  nodeRegistry: NodeRegistry;
  nodeSendToSession: (sessionKey: string, event: string, payload: unknown) => void;
  nodeSubscribe: (nodeId: string, sessionKey: string) => void;

  // 세션 상태 추적 맵
  dedupe: Map<string, DedupeEntry>;
  agentRunSeq: Map<string, number>;
  chatAbortControllers: Map<string, ChatAbortControllerEntry>;
  chatRunBuffers: Map<string, string>;

  // Plugin/Wizard
  wizardSessions: Map<string, WizardSession>;
  findRunningWizard: () => string | null;
};
```

이 컨텍스트가 모든 RPC 핸들러에 주입됨.

## 9. Broadcast 메커니즘

`src/gateway/server-broadcast-types.ts`:
```typescript
export type GatewayBroadcastFn = (
  event: string,
  payload: unknown,
  opts?: GatewayBroadcastOpts,
) => void;

export type GatewayBroadcastOpts = {
  dropIfSlow?: boolean;
  stateVersion?: GatewayBroadcastStateVersion;
};
```

### Presence broadcast 예

`src/gateway/server/presence-events.ts`:
```typescript
export function broadcastPresenceSnapshot(params: {
  broadcast: GatewayBroadcastFn;
  incrementPresenceVersion: () => number;
  getHealthVersion: () => number;
}): number {
  const presenceVersion = params.incrementPresenceVersion();
  params.broadcast(
    "presence",
    { presence: listSystemPresence() },
    {
      dropIfSlow: true,    // 느린 클라이언트 스킵
      stateVersion: { presence: presenceVersion, health: params.getHealthVersion() },
    },
  );
}
```

`dropIfSlow: true` — 백프레셔 처리. 느린 클라이언트가 큐 막히면 이 이벤트 그냥 누락.

## 10. 세션 라이프사이클 상태 머신

`src/gateway/session-lifecycle-state.ts:6-50`:
```typescript
type SessionRunStatus = "running" | "done" | "failed" | "killed" | "timeout";
type LifecyclePhase = "start" | "end" | "error";

function resolveTerminalStatus(event: LifecycleEventLike): SessionRunStatus {
  const phase = resolveLifecyclePhase(event);
  if (phase === "error") return "failed";
  const stopReason = typeof event.data?.stopReason === "string" ? event.data.stopReason : "";
  if (stopReason === "aborted") return "killed";
  return event.data?.aborted === true ? "timeout" : "done";
}
```

### 스냅샷 derive

`session-lifecycle-state.ts:92-130`:
```typescript
export function deriveGatewaySessionLifecycleSnapshot(params: {
  session?: Partial<LifecycleSessionShape> | null;
  event: LifecycleEventLike;
}): GatewaySessionLifecycleSnapshot {
  const phase = resolveLifecyclePhase(params.event);

  if (phase === "start") {
    return {
      updatedAt: startedAt,
      status: "running",
      startedAt,
      endedAt: undefined,
      runtimeMs: undefined,
      abortedLastRun: false,
    };
  }
  // "end" 또는 "error"
  const endedAt = resolveLifecycleEndedAt(params.event);
  return {
    updatedAt: endedAt,
    status: resolveTerminalStatus(params.event),
    startedAt,
    endedAt,
    runtimeMs: resolveRuntimeMs({ startedAt, endedAt }),
    abortedLastRun: resolveTerminalStatus(params.event) === "killed",
  };
}
```

## 11. Startup Flow (server.impl.ts)

`src/gateway/server.impl.ts:513-700` `startGatewayServer`. 순서:

### 11.1 Network runtime bootstrap (517-518)
```typescript
const { bootstrapGatewayNetworkRuntime } = await import("./server-network-runtime.js");
bootstrapGatewayNetworkRuntime();
```

### 11.2 Startup trace 생성 (533)
```typescript
const startupTrace = createGatewayStartupTrace();
```

### 11.3 Config snapshot 로드 (543-552)
```typescript
const startupConfigLoad = await startupTrace.measure("config.snapshot", () =>
  loadGatewayStartupConfigSnapshot({...}),
);
```

### 11.4 Secrets 활성화 (566-570)
```typescript
const activateRuntimeSecrets = createRuntimeSecretsActivator({
  logSecrets,
  emitStateEvent: emitSecretsStateEvent,
});
```

### 11.5 Auth 부트스트랩 (579-588)
```typescript
const authBootstrap = await startupTrace.measure("config.auth", () =>
  prepareGatewayStartupConfig({
    configSnapshot,
    authOverride: opts.auth,
    tailscaleOverride: opts.tailscale,
    activateRuntimeSecrets,
  }),
);
cfgAtStart = authBootstrap.cfg;
if (authBootstrap.generatedToken) {
  log.warn(formatRuntimeGatewayAuthTokenWarning());
}
```

### 11.6 Plugin 부트스트랩 (630-650)
```typescript
const pluginBootstrap = await startupTrace.measure("plugins.bootstrap", () =>
  prepareGatewayPluginBootstrap({...}),
);
```

### 11.7 Runtime config (684-697)
```typescript
const runtimeConfig = await startupTrace.measure("runtime.config", () =>
  resolveGatewayRuntimeConfig({...}),
);
```

### 11.8 TLS 런타임 (774-779)
```typescript
const gatewayTls = await startupTrace.measure("tls.runtime", () =>
  loadGatewayTlsRuntime(cfgAtStart.gateway?.tls, log.child("tls")),
);
```

### 11.9 WebSocket 핸들러 attach (1319-1347)
```typescript
attachGatewayWsHandlers({
  wss,
  clients,
  preauthConnectionBudget,
  port,
  getPluginNodeCapabilities: () => listPluginNodeCapabilities(pluginRegistry),
  rateLimiter: authRateLimiter,
  browserRateLimiter: browserAuthRateLimiter,
  gatewayMethods: runtimeState.gatewayMethods,
  events: GATEWAY_EVENTS,
  extraHandlers: attachedGatewayExtraHandlers,
  broadcast,
  context: gatewayRequestContext,
});
```

## 12. Startup Trace 상세

`src/gateway/server.impl.ts:200-394`:
```typescript
function createGatewayStartupTrace() {
  const logEnabled = isTruthyEnvValue(process.env.OPENCLAW_GATEWAY_STARTUP_TRACE);
  const eventLoopDelay = monitorEventLoopDelay({ resolution: 10 });

  return {
    mark(name: string) { /* emit timeline events */ },
    detail(name: string, metrics: ...) { /* detailed metrics */ },
    measure<T>(name: string, run: () => Promise<T> | T): Promise<T> {
      // span start, run, span end
    },
  };
}
```

활성화: `OPENCLAW_GATEWAY_STARTUP_TRACE=1` 환경변수.

측정되는 단계:
- `config.snapshot`
- `config.auth`
- `plugins.bootstrap`
- `runtime.config`
- `tls.runtime`
- `control-ui.root`
- `http.bound`
- `runtime.post-attach`

P50/P95/P99/Max 이벤트 루프 지연 샘플링도 함께.

## 13. 플러그인 핫리로드

`src/gateway/server.impl.ts:1060-1083`:
```typescript
const attachedGatewayExtraHandlers: GatewayRequestHandlers = {
  ...pluginRegistry.gatewayHandlers,
  ...extraHandlers,
};

const replaceAttachedPluginRuntime = (loaded: {
  pluginRegistry: typeof pluginRegistry;
  gatewayMethods: string[];
}) => {
  pluginRegistry = loaded.pluginRegistry;
  baseGatewayMethods = loaded.gatewayMethods;

  runtimeState.gatewayMethods.splice(
    0,
    runtimeState.gatewayMethods.length,
    ...listActiveGatewayMethods(baseGatewayMethods),
  );

  for (const key of attachedPluginGatewayHandlerKeys) {
    delete attachedGatewayExtraHandlers[key];
  }
  Object.assign(attachedGatewayExtraHandlers, pluginRegistry.gatewayHandlers);
  attachedPluginGatewayHandlerKeys = new Set(Object.keys(pluginRegistry.gatewayHandlers));
};
```

런타임 중 플러그인 등록/해제 가능. 메서드 목록과 핸들러 맵을 in-place 갱신.

## 14. 발견 사항 정리

| 항목 | 실제 |
|------|------|
| WebSocket 라이브러리 | `ws` (Node 표준) |
| 스키마 | TypeBox + AJV (compile됨) |
| 프레임 타입 | req / res / event 3가지 |
| 인증 모드 | none / token / password / tailscale / device-token / bootstrap-token / trusted-proxy |
| 기본 포트 | 18789 |
| Ping 간격 | 25초 |
| 큐 한도 | 16 frame (handler loading 중) |
| 보안 비교 | `safeEqualSecret` (constant-time) |
| Hot reload | `replaceAttachedPluginRuntime` 함수 |
| Startup trace | env `OPENCLAW_GATEWAY_STARTUP_TRACE=1` |
| 백프레셔 | `dropIfSlow` 옵션, stateVersion으로 stale 무시 |

## 15. 미확인 / 한계

- **RPC timeout** — 명시적 서버측 타임아웃 없음 (클라이언트 책임)
- **Connection pooling** — 클라이언트당 1 WebSocket
- **Backpressure** — `dropIfSlow` 외 대안 없음
- **Graceful shutdown** — pending 요청 완료 대기 정책 미확인
- **Plugin isolation** — 모든 플러그인 같은 V8 isolate
