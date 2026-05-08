# 02. Gateway — 중앙 제어 평면

## Gateway란?

**Gateway**는 OpenClaw의 핵심으로, 로컬에서 동작하는 WebSocket RPC 서버입니다. 모든 클라이언트(macOS 앱, iOS/Android 노드, 웹 UI, CLI)는 Gateway와 WebSocket으로 통신하며, Gateway가 다음을 담당합니다:

- 세션 라이프사이클 관리
- 모든 메시징 채널의 중앙 진입점
- 에이전트 런타임 오케스트레이션
- LLM 프로바이더 라우팅
- 설정/credential 관리
- 클라이언트 간 이벤트 브로드캐스트

핵심 디렉토리: `src/gateway/`

```
src/gateway/
├── server/                # WebSocket 서버
├── protocol/              # RPC 프로토콜 정의
│   ├── index.ts           # 메서드/이벤트 타입
│   ├── schema/            # JSON Schema (AJV 검증)
│   └── version.ts         # 프로토콜 버저닝
├── session/               # 세션 관리
└── auth/                  # 인증/페어링
```

## 전송 계층

기본 전송: **WebSocket** (`ws://` 또는 `wss://`)
폴백: **SSE** (Server-Sent Events) — 일부 환경에서 WebSocket 차단 시

| 시나리오 | URL | 인증 |
|---------|-----|------|
| 로컬 (같은 머신) | `ws://localhost:18789` | 없음 (loopback) |
| LAN (Tailscale 등) | `ws://gateway.local:18789` | 환경변수 `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` 명시 필요 |
| 원격 | `wss://gateway.example.com` | TLS + 토큰 필수 |
| 모바일 페어링 | QR 코드 → 단기 토큰 | 페어링 코드 |

## RPC 메서드 카테고리

`src/gateway/protocol/index.ts`에 정의된 메서드들:

### Agent 메서드
| 메서드 | 설명 |
|--------|------|
| `agent.create` | 새 에이전트 인스턴스 생성 |
| `agent.update` | 에이전트 설정 업데이트 |
| `agent.delete` | 에이전트 삭제 |
| `agent.list` | 에이전트 목록 |
| `agent.wait` | 에이전트 작업 완료 대기 |

### Chat 메서드
| 메서드 | 설명 |
|--------|------|
| `chat.send` | 메시지 전송 → ChatEvent 스트림 응답 |
| `chat.abort` | 진행 중인 응답 중단 |
| `chat.history` | 대화 기록 조회 |

### Channel 메서드
| 메서드 | 설명 |
|--------|------|
| `channels.start` | 채널 활성화 (e.g. Telegram 봇 시작) |
| `channels.status` | 채널별 상태 조회 |
| `channels.logout` | 채널 자격증명 제거 |

### Config 메서드
| 메서드 | 설명 |
|--------|------|
| `config.get` | 전체 설정 트리 |
| `config.set` | 경로 기반 부분 업데이트 |
| `config.schema` | JSON Schema 응답 (UI 렌더링용) |

### Talk / Voice 메서드
| 메서드 | 설명 |
|--------|------|
| `talk.start` | 음성 세션 시작 |
| `talk.stop` | 음성 세션 종료 |
| `talk.audio` | 오디오 청크 스트림 |

### Commands 메서드
| 메서드 | 설명 |
|--------|------|
| `commands.run` | 슬래시 커맨드 실행 (`/help` 등) |
| `commands.list` | 사용 가능한 커맨드 |

## 이벤트 (서버 → 클라이언트)

서버는 클라이언트에 다음 이벤트를 푸시:

- `agentEvent` — 에이전트 생성/업데이트/삭제 알림
- `chatEvent` — 메시지 청크, 도구 호출, 완료 신호
- `channelEvent` — 채널 상태 변경 (연결/끊김)
- `configEvent` — 설정 변경 알림
- `notificationEvent` — 시스템 알림

## 프로토콜 스키마 검증

런타임 검증: **AJV** (JSON Schema)
- `src/gateway/protocol/schema/` 에 모든 메서드/이벤트의 JSON Schema 정의
- 클라이언트 호출 시 inbound payload 검증
- 응답도 outbound 검증 (개발 모드)

## 버저닝 정책

`AGENTS.md:45`:
> Gateway protocol changes: additive first; incompatible needs versioning/docs/client follow-through.

### Additive (호환)
- 선택적 필드 추가 OK
- 새 메서드 추가 OK
- 새 이벤트 추가 OK (구 클라이언트는 무시)
- 신규 enum 값 추가 OK (기본값/폴백 정의 시)

### Breaking (호환 불가)
- 필드 제거/이름 변경
- 필수 필드 추가
- 메서드 시그니처 변경
- 이벤트 페이로드 구조 변경

Breaking 변경은:
1. 새 메서드/이벤트 이름으로 추가 (`chat.sendV2`)
2. 구 메서드는 deprecation 마커 + 한동안 유지
3. `version.ts`에서 프로토콜 메이저 버전 증가
4. 클라이언트 SDK 버전 동기화

## 연결 흐름

```
클라이언트                  Gateway
   │                          │
   │ WebSocket connect        │
   ├─────────────────────────►│
   │                          │
   │ ConnectParams            │
   │   {clientType, version}  │
   ├─────────────────────────►│
   │                          │ 핸드셰이크
   │ ConnectResult            │
   │   {protocolVersion,      │
   │    capabilities,         │
   │    serverInfo}           │
   │◄─────────────────────────┤
   │                          │
   │ rpc: chat.send           │
   ├─────────────────────────►│
   │                          │ 처리
   │ event: chatEvent (chunk) │
   │◄─────────────────────────┤
   │ event: chatEvent (chunk) │
   │◄─────────────────────────┤
   │ event: chatEvent (done)  │
   │◄─────────────────────────┤
   │                          │
```

## 페어링 (모바일)

1. **데스크탑 Gateway**가 임시 페어링 코드 생성 (QR로 표시)
2. **모바일 앱**이 QR 스캔 → `pair.start` RPC
3. Gateway가 코드 검증 → 단기 페어링 토큰 발급
4. 페어링 토큰으로 정식 인증 토큰 교환 (`pair.complete`)
5. 정식 토큰을 모바일에 안전하게 저장 (Keychain/Keystore)

LAN pairing은 plaintext `ws://` loopback만 기본 허용. private network는 위에 언급한 환경변수 필요. 원격은 TLS 필수.

## CLI ↔ Gateway

CLI(`pnpm openclaw`)도 Gateway 클라이언트입니다:

```bash
openclaw agent --message "..."   # → chat.send RPC
openclaw doctor                  # → config 검증 + 진단
openclaw onboard                 # → 대화형 셋업
openclaw gateway start           # Gateway 데몬 시작
openclaw gateway status --deep   # Gateway 헬스체크
```

CLI는 자체 프로세스로 실행되지 않고, 항상 Gateway에 연결해서 작동.
