# 01. 전체 아키텍처와 모노레포 구조

## pnpm Workspace

OpenClaw는 pnpm workspace 기반 모노레포입니다.

`pnpm-workspace.yaml`:
```yaml
packages:
  - .                  # 루트 (메인 패키지)
  - ui                 # 웹 UI
  - packages/*         # 공개 SDK 패키지
  - extensions/*       # 130+ 플러그인
```

## 디렉토리 레이아웃

```
openclaw/
├── src/                    # Core TypeScript (105+ 모듈)
│   ├── gateway/            # WebSocket RPC 서버 + 프로토콜
│   ├── channels/           # 채널 추상화 (인바운드/아웃바운드)
│   ├── agents/             # 에이전트 런타임 + plan 빌더
│   ├── plugin-sdk/         # 플러그인 공개 계약
│   ├── plugins/            # 플러그인 발견/로더/레지스트리
│   ├── model-catalog/      # 모델 메타데이터 인덱싱
│   ├── memory/             # 메모리 호스트
│   ├── routing/            # 채널 → 에이전트 라우팅
│   ├── tools/              # 내장 도구
│   ├── auto-response/      # 자동 응답 정책
│   ├── session/            # 세션 라이프사이클
│   └── cli/                # CLI 진입점
│
├── extensions/             # 130+ 플러그인
│   ├── telegram/           # 채널 플러그인
│   ├── discord/
│   ├── slack/
│   ├── whatsapp/
│   ├── anthropic/          # 프로바이더 플러그인
│   ├── openai/
│   ├── google/
│   ├── canvas/             # 도구 플러그인
│   ├── active-memory/      # 메모리 플러그인
│   └── ... (총 130+)
│
├── packages/               # 공개 SDK 패키지
│   ├── plugin-sdk/         # 플러그인 작성자용 SDK
│   ├── sdk/                # 클라이언트 SDK
│   ├── memory-host-sdk/    # 메모리 호스트 SDK
│   └── plugin-package-contract/
│
├── apps/                   # 클라이언트 앱
│   ├── macos/              # SwiftUI (Sparkle 자동 업데이트)
│   ├── ios/                # iOS (Watch Kit + Share Extension 포함)
│   ├── android/            # Kotlin
│   └── shared/             # 공유 로직
│
├── ui/                     # 웹 UI (TypeScript/React)
│
├── skills/                 # 55+ 번들 스킬
│   ├── github/
│   ├── notion/
│   ├── 1password/
│   ├── canvas/
│   └── ... (총 55+)
│
├── docs/                   # 사용자/개발자 문서
├── deploy/                 # 배포 매니페스트
├── git-hooks/              # Git pre-commit/pre-push 훅
├── qa/                     # QA 시나리오
├── security/               # 보안 정책
├── patches/                # pnpm 의존성 패치
│
├── Dockerfile              # 멀티스테이지 Docker 빌드
├── docker-compose.yml
├── fly.toml                # Fly.io 배포
├── render.yaml             # Render 배포
├── appcast.xml             # macOS Sparkle appcast
├── openclaw.mjs            # 진입 스크립트
├── package.json
├── pnpm-workspace.yaml
├── pnpm-lock.yaml
│
├── AGENTS.md               # 에이전트(Claude) 행동 규칙
├── CLAUDE.md               # AGENTS.md 심볼릭 링크
├── CONTRIBUTING.md
├── SECURITY.md
├── VISION.md
├── CHANGELOG.md
└── README.md
```

## 데이터 흐름

사용자 메시지 한 통이 처리되는 전체 흐름:

```
┌─────────────────┐
│  사용자 (모바일)  │
└────────┬────────┘
         │ 메시지 전송
         ▼
┌─────────────────────────────────┐
│  메시징 서비스 (Telegram, etc.)  │
└────────┬────────────────────────┘
         │ 웹훅 / 폴링
         ▼
┌──────────────────────────────────┐
│  채널 플러그인                    │
│  (extensions/telegram/)           │
│  - 인바운드 메시지 정규화          │
└────────┬─────────────────────────┘
         │ InboundMessage
         ▼
┌──────────────────────────────────┐
│  Gateway                          │
│  (src/gateway/)                   │
│  - 세션 관리                      │
│  - 라우팅                         │
└────────┬─────────────────────────┘
         │ AgentRuntimePlan + InboundMessage
         ▼
┌──────────────────────────────────┐
│  Agent Runtime                    │
│  (src/agents/)                    │
│  - 메모리 회상 sub-agent          │
│  - 도구 결정                      │
└────────┬─────────────────────────┘
         │ Provider 요청
         ▼
┌──────────────────────────────────┐
│  Provider 플러그인                │
│  (extensions/anthropic/)          │
│  - Auth                           │
│  - 스트림 래퍼                    │
│  - 재시도 정책                    │
└────────┬─────────────────────────┘
         │ HTTPS / SSE
         ▼
┌──────────────────────────────────┐
│  LLM API (Anthropic/OpenAI 등)    │
└────────┬─────────────────────────┘
         │ 응답 스트림
         ▼
   (역방향으로 채널까지 전달)
```

## 4계층 아키텍처

OpenClaw는 명확한 계층 경계를 강제합니다 (`AGENTS.md:26-42`):

### Layer 1: Core (`src/`)
- 확장에 무관(extension-agnostic)
- 게이트웨이 프로토콜, 세션, 라우팅의 일반 메커니즘만 담당
- **bundled ID 하드코딩 금지** — 모든 채널/프로바이더 ID는 매니페스트로 들어와야 함

### Layer 2: Plugin SDK (`packages/plugin-sdk/`, `src/plugin-sdk/`)
- Core ↔ 플러그인의 **유일한 공개 경계**
- `api.ts` (정적 메타데이터), `runtime-api.ts` (런타임 훅)
- 서드파티 플러그인이 안전하게 확장 가능한 진입점

### Layer 3: Bundled Plugins (`extensions/`)
- 130+ 플러그인 (채널, 프로바이더, 메모리, 도구)
- 각자 `openclaw.plugin.json` 매니페스트 보유
- Core 내부(`src/**`)에 직접 import 금지

### Layer 4: Apps & UI (`apps/`, `ui/`)
- WebSocket 프로토콜로만 Gateway와 통신
- Gateway 프로토콜 변경은 additive-first

## 경계 규칙

`AGENTS.md`에 명시된 강제 규칙:

```
- 확장 prod 코드는 core src/**, src/plugin-sdk-internal/**, 다른 확장 src/**, 
  패키지 외부 상대 경로 import 금지
- core/tests는 깊은 플러그인 내부(extensions/*/src/**) import 금지
- 채널 src/channels/**는 구현; 플러그인 작성자는 SDK seam 사용
- providers: core가 generic loop 소유; 플러그인이 auth/catalog/runtime hooks 소유
```

이 경계는 단순 컨벤션이 아니라 `pnpm check:architecture`, `pnpm check:import-cycles`, madge 등으로 검증됩니다.

## 버저닝

- 메인 패키지 버전: `package.json:version` (예: `2026.5.6`)
- **날짜 기반 버저닝** (CalVer: `YYYY.M.D[-beta.N]`)
- npm tag: 안정판 `latest`, 베타 `beta`
- macOS는 별도 `Info.plist`, Android는 `apps/android/app/build.gradle.kts`, iOS는 `apps/ios/version.json`에서 동기화
