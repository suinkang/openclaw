# 04. 채널 통합 (Channels)

> ⚠️ **일부 추정 부정확.** 실제 코드 기반 분석은 [deep-dive/04-channels-canvas.md](./deep-dive/04-channels-canvas.md). 정정: Discord는 `@buape/carbon` **사용 안 함** (`discord-api-types` + `ws` 직접). iMessage는 AppleScript가 아니라 **`imsg` CLI + JSON-RPC over stdio**.

## 개요

OpenClaw는 사용자가 평소 사용하는 메시징 서비스를 통해 AI 어시스턴트에 접근하게 합니다. 각 메시징 서비스는 **채널 플러그인**으로 추상화되며, Core는 일반화된 인터페이스만 인지합니다.

## 지원 채널 (README.md:26)

```
WhatsApp, Telegram, Slack, Discord, Google Chat, Signal, iMessage,
IRC, Microsoft Teams, Matrix, Feishu, LINE, Mattermost,
Nextcloud Talk, Nostr, Synology Chat, Tlon, Twitch,
Zalo, Zalo Personal, WeChat, QQ, WebChat
```

각 채널은 `extensions/<channel-id>/` 디렉토리에 독립 플러그인으로 구현.

## 디렉토리 구조

### Core 측 (`src/channels/`)
```
src/channels/
├── plugins/              # 플러그인 계약 정의
│   ├── types.ts
│   ├── registry.ts
│   └── ...
├── conversation-resolution.ts   # 채널별 대화 ID 매핑
├── session-envelope.ts          # 세션 컨텍스트 추출
├── target-parsing.ts            # outbound 대상 파싱
└── ...
```

### Plugin 측 (`extensions/<channel-id>/`)
```
extensions/telegram/
├── api.ts                # 정적 진입점
├── runtime-api.ts        # 런타임 훅 (lazy)
├── setup-entry.ts        # 온보딩 흐름
├── openclaw.plugin.json  # 매니페스트
├── package.json
└── src/
    ├── inbound/          # 메시지 수신
    ├── outbound/         # 메시지 전송
    ├── auth/             # 인증
    └── types/            # 채널별 타입
```

## 핵심 데이터 타입

### Inbound (사용자 → Gateway)

```typescript
type InboundMessage = {
  channelId: string;        // "telegram", "discord", ...
  senderId: string;         // 채널 내부 사용자 ID
  senderLabel?: string;     // 표시 이름 (선택)
  text?: string;            // 텍스트 내용
  mediaUrls?: string[];     // 미디어 첨부 URL
  conversationId: string;   // 스레드/채팅 ID
  timestamp: number;
  raw?: unknown;            // 채널별 원본 (디버깅용)
};
```

### Outbound (Gateway → 사용자)

```typescript
type OutboundReply = {
  target: string;           // "telegram:12345" 또는 "discord:guild#channel"
  text?: string;
  mediaUrl?: string;
  replyTo?: string;         // 답장 대상 메시지 ID
  capabilities?: {          // 채널별 기능 명시
    canEdit?: boolean;
    canReact?: boolean;
    canStream?: boolean;
  };
};
```

## 채널 추상화 패턴

### 공통 인터페이스
```typescript
interface ChannelPlugin {
  id: string;
  start(): Promise<void>;
  stop(): Promise<void>;
  send(reply: OutboundReply): Promise<void>;
  onMessage(callback: (msg: InboundMessage) => void): void;
  status(): ChannelStatus;
}
```

### 채널별 차이 흡수

| 항목 | 차이 예시 |
|------|----------|
| 메시지 형식 | 텍스트만 / Markdown / HTML / Discord embed / Slack blocks |
| 미디어 | 단일 첨부 / 다중 첨부 / 인라인 / 파일 업로드 |
| 스레드 | 없음 (IRC) / 답글 (Telegram) / 명시적 스레드 (Slack) |
| 반응(reactions) | 없음 / 이모지 / 커스텀 |
| 편집 | 불가 / 시간 제한 / 자유 |
| 타이핑 표시 | 없음 / "...입력 중" / 이벤트 기반 |

각 채널 플러그인이 위 차이를 자체 흡수하여 표준 인터페이스를 구현.

## Target 파싱 (`src/channels/target-parsing.ts`)

Outbound 대상은 단일 문자열로 인코딩:

```
telegram:12345                   # Telegram chat ID 12345
discord:guild_id#channel_id      # Discord 길드/채널
slack:T0123#C0456                # Slack 워크스페이스/채널
slack:T0123#C0456@thread_ts      # Slack 스레드
imessage:+15551234567            # iMessage 전화번호
matrix:!room:server.org          # Matrix room
```

파싱 → `{ channelId, conversationId, threadId? }`

## Conversation Resolution

채널별로 "대화"의 정의가 다르므로 매핑 필요:

```typescript
// src/channels/conversation-resolution.ts (개념적)
function resolveConversation(channel: string, raw: unknown): ConversationKey {
  switch (channel) {
    case "telegram":
      return { id: raw.chat.id, threadId: raw.message_thread_id };
    case "discord":
      return { id: `${raw.guildId}#${raw.channelId}`, threadId: raw.threadId };
    case "slack":
      return { id: `${raw.team}#${raw.channel}`, threadId: raw.thread_ts };
    // ...
  }
}
```

## Session Envelope

각 인바운드 메시지는 **세션 envelope**으로 감싸져 Gateway로 전달:

```typescript
type SessionEnvelope = {
  message: InboundMessage;
  channel: ChannelMetadata;
  account: AccountIdentity;       // 어느 OpenClaw 사용자 계정?
  agent: AgentBinding;            // 어느 에이전트로 라우팅?
  permissions: PermissionGrant;
  trace: TraceContext;
};
```

라우팅 규칙은 `src/routing/`에서 결정:
- 발신자 ID → 사용자 매핑
- 채널/그룹 → 에이전트 매핑
- 권한 검증 (DM 정책, allowFrom 등)

## 권한 / 보안

`README.md:132-144`에 명시된 채널 보안 모델:

```yaml
channels:
  telegram:
    dmPolicy: "pairing"       # 기본: 미지인은 페어링 코드 요구
    allowFrom: ["user_id_1", "group_id_1"]
    
  discord:
    dmPolicy: "open"          # 명시 옵트인 시 모든 DM 허용
    allowFrom: ["*"]
```

| 정책 | 의미 |
|------|------|
| `dmPolicy: "pairing"` | 미지인 발신자는 일회성 페어링 코드로 인증 |
| `dmPolicy: "open"` | 모든 DM 허용 (스팸 위험, 명시 동의 필요) |
| `allowFrom: [...]` | ID 화이트리스트 |
| `allowFrom: ["*"]` | 전체 허용 |

## 채널 설정 흐름

`onboard` CLI가 단계별 안내:

1. **Channel 선택** — 활성화할 채널 선택
2. **Auth** — 봇 토큰 / OAuth / QR 코드 등 채널별 흐름
3. **Account 매핑** — 봇이 사용자 계정에 매핑
4. **권한 설정** — `dmPolicy`, `allowFrom`
5. **Test message** — 양방향 작동 검증
6. **저장** — `~/.openclaw/credentials/<channel>.json`

## 읽기 전용 / 단방향 채널

일부 채널은 메시지 전송만 가능 (RSS-like):

```json
// 매니페스트
{
  "channels": ["webhook-out"],
  "capabilities": {
    "inbound": false,
    "outbound": true
  }
}
```

알림/리포트 전송 전용으로 사용 (예: 스케줄러가 매일 요약을 Discord 채널에 푸시).

## 샌드박싱

`README.md:160`:
> agents.defaults.sandbox.mode: "non-main"  # 채널/그룹은 Docker 샌드박스

채널/그룹 메시지 처리 시 Docker 샌드박스에서 에이전트 실행 가능. 메인 사용자 DM과 분리하여 권한 격리.

## 새 채널 추가 가이드

1. `extensions/<channel-id>/` 생성
2. 매니페스트에 `channels: ["<id>"]` 명시
3. `setup-entry.ts`에서 인증 흐름 구현
4. `runtime-api.ts`에서 inbound/outbound 핸들러 export
5. 채널별 capability를 매니페스트에 명시
6. `.github/labeler.yml` 업데이트
7. 통합 테스트 작성 (`extensions/<id>/test/`)
