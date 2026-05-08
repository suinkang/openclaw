# 08. 클라이언트 앱 (macOS / iOS / Android / Web)

## 클라이언트 아키텍처 공통점

모든 클라이언트는 **Gateway WebSocket RPC**로만 OpenClaw와 통신합니다. 클라이언트 자체는 LLM API 직접 호출하지 않고, 모든 라우팅/실행/메모리는 Gateway가 담당.

```
[macOS / iOS / Android / Web UI]
            ↕ WebSocket (ws:// or wss://)
        [Gateway 프로세스]
            ↕
   [Channel plugins, Provider plugins, Memory, ...]
```

## macOS (`apps/macos/`)

### 기술 스택
- **언어**: Swift 5.9+
- **UI**: SwiftUI (Observation API)
- **빌드**: Xcode 프로젝트 (XcodeGen 또는 직접)
- **자동 업데이트**: Sparkle framework
- **Voice**: AVFoundation, on-device wake word

### 주요 기능

#### 1. 메뉴바 컨트롤
- 항상 메뉴바에 상주
- 클릭 → 컴팩트 컨트롤 패널
- 빠른 명령, 채널 상태, 에이전트 전환

#### 2. Voice Wake
- On-device 키워드 감지 (개인정보 보호)
- "Hey Claw" 등 사용자 정의 키워드
- 백그라운드 저전력 모드

#### 3. Push-to-Talk 오버레이
- 글로벌 hot key (예: `⌘⇧Space`)
- 누르면 마이크 시작, 떼면 종료
- 화면 어디서든 작동

#### 4. WebChat + 디버그 도구
- 풀 스크린 채팅 UI
- 프로토콜 인스펙터 (Gateway RPC 추적)
- 로그 뷰어

#### 5. Sparkle 자동 업데이트
- `appcast.xml` 모니터링
- 백그라운드 다운로드, 사용자 동의 후 적용

### 코드 패턴

`AGENTS.md:194`:
> SwiftUI: Observation (`@Observable`, `@Bindable`) over new `ObservableObject`.

새 코드는 Observation API 사용:

```swift
// ✅ 권장
@Observable
class ChatStore {
    var messages: [Message] = []
    var isStreaming = false
}

struct ChatView: View {
    @Bindable var store: ChatStore
    
    var body: some View {
        // ...
    }
}

// ❌ 비권장 (legacy)
class ChatStore: ObservableObject {
    @Published var messages: [Message] = []
}
```

### Gateway 페어링

- 첫 실행 시 사용자가 Gateway 실행 (CLI 또는 Docker)
- 페어링 코드 입력 또는 QR 스캔
- 정식 토큰 → Keychain 저장

### 빌드

```bash
cd apps/macos
xcodebuild -project OpenClaw.xcodeproj -scheme Release
```

## iOS (`apps/ios/`)

### 기술 스택
- **언어**: Swift 5.9+
- **UI**: SwiftUI
- **빌드**: Xcode (project.yml — XcodeGen)
- **버전 동기**: `apps/ios/version.json`

### 주요 기능

#### 1. Node 앱 모드
- iOS 기기를 Gateway "노드"로 동작
- Gateway에 페어링된 디바이스로 등록
- 음성/위치/알림을 Gateway로 포워딩

#### 2. 음성 트리거
- Wake word 감청 (백그라운드)
- Siri Shortcut 통합
- Apple Watch 빠른 명령

#### 3. Canvas 렌더링
- A2UI 컴포넌트 네이티브 렌더링
- 인터랙션 → Gateway action 이벤트

#### 4. Activity Widget
- 홈 화면 위젯 (현재 작업, 알림)
- Live Activity (진행 중인 도구 실행 표시)

#### 5. Share Extension
- iOS 공유 시트 통합
- 사파리/메일 등에서 → "Send to Claw"

#### 6. Watch App
- Apple Watch 컴플리케이션
- 빠른 음성 메모
- 컨텍스트 푸시 (현재 활동)

### 빌드

```bash
cd apps/ios
xcodegen generate
xcodebuild -project OpenClaw.xcodeproj -scheme Release
```

### 버전 동기화

`AGENTS.md:198`에 명시:
```
Version bump touches:
- package.json
- apps/android/app/build.gradle.kts
- apps/ios/version.json + pnpm ios:version:sync
- macOS Info.plist
- docs/install/updating.md
```

`pnpm ios:version:sync`가 `package.json` 버전 → iOS 프로젝트 동기화.

## Android (`apps/android/`)

### 기술 스택
- **언어**: Kotlin
- **UI**: Jetpack Compose
- **빌드**: Gradle (`build.gradle.kts`)
- **음성**: Speech Recognition API + 지속적 마이크 옵션

### 주요 기능

#### 1. Talk Mode
- Foreground Service로 지속적 마이크
- 사용자 동의 + 영구 알림
- 배터리 최적화 안내

#### 2. Canvas Renderer
- WebView 기반 (Canvas HTML 렌더링)
- A2UI 네이티브 변환

#### 3. Notification Bridge
- Android 알림 → Gateway로 포워드 (선택)
- 봇 응답 → 시스템 알림

#### 4. Tasker / Shortcuts 통합
- 자동화 트리거 가능
- Quick Settings tile

### 빌드

```bash
cd apps/android
./gradlew assembleRelease
```

## Web UI (`ui/`)

### 기술 스택
- **언어**: TypeScript
- **프레임워크**: React (추론)
- **번들러**: Vite (추론)

### 주요 화면

#### 1. WebChat
- 메인 대화 인터페이스
- 스트리밍 메시지 표시
- 도구 호출 가시화 (펼침/접힘)
- Canvas 임베드

#### 2. Settings Dashboard
- JSON Schema → UI 자동 생성 (`config.schema` RPC)
- 채널 활성화/비활성화
- 에이전트 편집
- 메모리 백엔드 선택

#### 3. Plugin Manager
- 설치된 플러그인 목록
- 매니페스트 메타데이터 표시
- 활성화/비활성화 토글

#### 4. Logs / Diagnostics
- 실시간 Gateway 로그 스트림
- RPC 트레이스
- 에러 인스펙터

### 원격 접근

- 원격에서 자체 Gateway에 접근하려면:
  - Tailscale / Wireguard / Cloudflare Tunnel
  - 또는 Fly.io에 Gateway 배포 + `wss://` + 토큰

## CLI (`src/cli/`, `openclaw.mjs`)

CLI도 Gateway 클라이언트:

```bash
# 직접 메시지
openclaw agent --message "What's on my calendar?"

# 인터랙티브 모드
openclaw

# 셋업
openclaw onboard

# 진단
openclaw doctor
openclaw doctor --fix

# Gateway 관리
openclaw gateway start
openclaw gateway status --deep
openclaw gateway restart
openclaw gateway stop
```

### Progress / Status 표시

- `src/cli/progress.ts` — 진행 표시
- `src/terminal/table.ts` — 상태 테이블

### Onboard 흐름

`pnpm openclaw onboard`:
1. Welcome
2. Workspace 디렉토리 선택
3. 첫 채널 선택 + 인증
4. 첫 프로바이더 선택 + auth
5. 첫 에이전트 생성
6. 테스트 메시지

## 페어링 모델

`AGENTS.md:197`:
> Mobile LAN pairing: plaintext `ws://` loopback-only. Private-network `ws://` needs `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1`; Tailscale/public use `wss://` or tunnel.

### 시나리오별 보안

| 시나리오 | 전송 | 인증 |
|---------|------|------|
| 같은 머신 | `ws://localhost` | 없음 (loopback) |
| 같은 LAN | `ws://192.168.1.x` | 페어링 토큰 + `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` |
| Tailscale | `ws://node.ts.net` | 페어링 토큰 (TS 자체 암호화) |
| 인터넷 | `wss://` only | 페어링 토큰 + TLS |

### Gateway watch (개발)

`AGENTS.md:198`:
> Mac gateway: dev watch = `pnpm gateway:watch` (tmux `openclaw-gateway-watch-main`, auto-attach).

개발자용 hot-reload Gateway:
```bash
pnpm gateway:watch
# tmux session: openclaw-gateway-watch-main
# 자동 attach
```

비대화형:
```bash
OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch
tmux attach -t openclaw-gateway-watch-main
tmux kill-session -t openclaw-gateway-watch-main
```

## Crabbox (라이브 시나리오 테스트)

`AGENTS.md:166`:
> Crabbox: preferred live scenario runner when available. It has Linux, Windows, and macOS workers/targets.

운영 환경 시뮬레이션 머신:
- macOS / Windows / Linux 풀
- 라이브 라이브 채널 검증 (실제 Telegram 봇, Discord 등)
- WebVNC로 화면 보기

특정 OS에서만 발생하는 버그 검증에 핵심.
