# 07. Canvas, A2UI, Skills

> ⚠️ **구현 디테일 정정.** 실제 코드 기반 분석은 [deep-dive/04-channels-canvas.md](./deep-dive/04-channels-canvas.md). 정정 사항: Canvas 번들러는 **Rolldown** (Vite팀 차세대 번들러). A2UI 프레임워크는 **Lit + `@a2ui/lit` 0.9.3** (React 아님). 라이브 리로드는 `/__openclaw__/ws` WebSocket으로 `"reload"` 문자열 1개만 전송 후 `location.reload()`.

## Canvas — 라이브 사용자 인터페이스

**Canvas**는 OpenClaw의 시그니처 기능 중 하나로, 에이전트가 사용자에게 **라이브 GUI를 렌더링**할 수 있게 합니다. 단순 텍스트 응답을 넘어 인터랙티브한 위젯, 대시보드, 폼 등을 직접 그릴 수 있습니다.

플러그인 위치: `extensions/canvas/`

### 매니페스트

`extensions/canvas/openclaw.plugin.json`:
```json
{
  "id": "canvas",
  "activation": { "onStartup": true },
  "enabledByDefault": true,
  "contracts": { "tools": ["canvas"] },
  "configSchema": {
    "properties": {
      "host": {
        "properties": {
          "enabled": { "type": "boolean" },
          "root": { "type": "string" },
          "port": { "type": "integer" },
          "liveReload": { "type": "boolean" }
        }
      }
    }
  }
}
```

### 사용 예 (에이전트 관점)

```typescript
// 도구 호출
await runtime.tools.canvas.render({
  html: `
    <div class="weather-widget">
      <h2>${city}</h2>
      <div class="temp">${temp}°C</div>
      <button onclick="refresh()">Refresh</button>
    </div>
  `,
  css: "...",
  js: "function refresh() { /* ... */ }"
});
```

렌더링 위치:
- macOS: 메뉴바 앱 내 패널
- iOS/Android: Canvas 뷰
- 웹 UI: 메인 영역

### Live Reload

`liveReload: true` 시 에이전트가 동일 Canvas를 재호출하면 DOM 패치 (전체 리렌더링 없이 부분 업데이트). 실시간 데이터 표시(주식, 날씨, 진행 중인 작업) 등에 유용.

## A2UI — Agent-to-User Interface

**A2UI**는 Canvas 위에 구축된 더 높은 수준의 추상화입니다. 에이전트가 raw HTML이 아니라 **선언적 UI 컴포넌트**로 사용자 인터페이스를 표현.

### 번들

위치: `extensions/canvas/src/host/a2ui/`

빌드:
```bash
pnpm canvas:a2ui:bundle
# → extensions/canvas/src/host/a2ui/.bundle.hash
```

생성되는 `.bundle.hash`는 캐시 무효화용으로 git에 커밋 (별도 commit 권장, `AGENTS.md:198`).

### A2UI vs Raw Canvas

| 항목 | Raw Canvas | A2UI |
|------|-----------|------|
| 인터페이스 | HTML/CSS/JS | 컴포넌트 트리 |
| 상태 관리 | 수동 | 자동 reactive |
| 보안 | iframe 격리 | 검증된 프리미티브만 |
| 토큰 비용 | 높음 (전체 HTML) | 낮음 (컴포넌트 ID + props) |
| 일관성 | 디자인 분산 | OpenClaw 디자인 시스템 |

### A2UI 컴포넌트 (개념적)

```typescript
await runtime.tools.canvas.a2ui({
  components: [
    { type: "Heading", level: 2, text: "Today's Tasks" },
    { type: "List", items: [
      { type: "Task", title: "Review PR", done: false },
      { type: "Task", title: "Reply email", done: true }
    ]},
    { type: "Button", label: "Add Task", action: "task.create" }
  ]
});
```

### 작용 (action) 핸들링

A2UI 컴포넌트의 `action`은 Gateway 이벤트로 라우팅:
```
사용자가 "Add Task" 클릭
  → A2UI runtime에서 action 캡처
  → Gateway WebSocket 이벤트 송신
  → 에이전트 hook 트리거
  → 에이전트가 새 Canvas 렌더링 (폼 표시)
```

## Skills — 번들 자동화

**Skills**는 OpenClaw에 번들된 도구/자동화 모음입니다 (`skills/` 디렉토리, 55+개).

### 위치
```
skills/
├── 1password/
├── apple-reminders/
├── apple-shortcuts/
├── browser/
├── canvas/
├── discord/
├── github/
├── google-drive/
├── google-calendar/
├── jira/
├── linear/
├── notion/
├── slack/
├── spotify/
├── telegram/
├── todoist/
└── ... (총 55+)
```

### Skills vs Plugins

| 항목 | Skills | Plugins |
|------|--------|---------|
| 위치 | `skills/` | `extensions/` |
| 활성화 | 설정만 (코드 실행 X) | 매니페스트 + 코드 |
| 형식 | MCP / ACPX / 스크립트 | npm 패키지 |
| 마켓플레이스 | 가능 (ClawHub) | 가능 |
| 커스터마이징 | 사용자 작성 가능 | 개발자 친화 |

### Skill 형식

스킬은 다음 중 하나:

1. **MCP (Model Context Protocol)** — Anthropic의 외부 도구 표준
2. **ACPX** — OpenClaw 자체 스킬 정의 형식
3. **Cluster** — 여러 스킬 묶음
4. **Native script** — Node.js / Python 스크립트

### 스킬 예: GitHub

```
skills/github/
├── manifest.json          # 메타데이터
├── tools/                 # 개별 도구 정의
│   ├── list-repos.ts
│   ├── create-pr.ts
│   ├── review-pr.ts
│   └── ...
├── prompts/               # 시스템 프롬프트 추가
└── README.md
```

### 활성화

에이전트 설정에서 스킬 명시:
```yaml
agents:
  main:
    skills:
      - github
      - notion
      - canvas
```

### ClawHub

`VISION.md:78-82`에 명시된 미래 방향:
- 공개 스킬 마켓플레이스
- 커뮤니티 기여 스킬 검색/설치
- 평가, 평점, 보안 검토

## 비교: Skill / Tool / Plugin

용어 혼동을 피하기 위해:

| 단어 | 정의 |
|------|------|
| **Tool** | LLM이 호출하는 함수 (function calling 단위) |
| **Skill** | Tool들의 묶음 + 프롬프트 추가 (사용자 활성화 단위) |
| **Plugin** | 코드를 실행하는 확장 (`extensions/`, 매니페스트 보유) |

스킬은 plugins로 구현되거나, plugins 없이 manifest만으로 활성화 가능.

## Memory Wiki와 People Directory

`AGENTS.md:213`에 나오는 wiki는 일종의 메모리 + 검색 엔진:

```
~/.openclaw/wiki/
├── reports/
│   ├── person-agent-directory.md   # 사람 → 에이전트 라우팅
│   └── ...
├── people/                         # 개인 정보
│   ├── alice.md
│   └── bob.md
└── ...
```

검색 모드:
- `find-person` — 사람 검색
- `route-question` — 질문 → 적임자 라우팅
- `source-evidence` — 출처 검증
- `raw-claim` — 인용 추출

`AGENTS.md:215`:
> People wiki provenance: generated identity, social, contact, and "fun detail" notes need explicit source class/confidence.

생성된 인물 정보는 출처 명시 필요 (`maintainer-whois`, GitHub 프로필, Discord 샘플 등). 추론된 사실을 사실로 승격 금지.

## Voice Wake / Talk Mode

### macOS
- On-device wake word ("Hey Claw")
- 백그라운드 audio capture (저전력)
- Push-to-talk hot key

### iOS
- 지속적 마이크 (배터리 영향)
- BackgroundTasks framework

### Android
- Foreground service + 알림
- AccessibilityService (선택)

### Watch (iOS)
- Apple Watch에서 빠른 음성 메모
- 핸드폰 리프트 없이 명령

## Streaming UX

채널/Canvas 모두에서 스트림은 사용자 경험 핵심:

`AGENTS.md:204`:
> preview/block streaming uses edits/chunks and preserves final/fallback delivery.

전략:
1. **Block streaming** — 의미 단위(문단/코드 블록)로 묶어서 편집
2. **Preview** — 첫 응답은 placeholder, 이후 편집
3. **Fallback** — 편집 미지원 채널은 최종 1회만 전송
4. **Final guarantee** — 네트워크 단절 시에도 최종 응답 보장

## Sandbox 모드

채널/그룹 메시지는 격리 실행 가능:

```yaml
agents:
  defaults:
    sandbox:
      mode: non-main         # Docker
```

- `main` — 메인 프로세스 (개인 DM)
- `non-main` — Docker 컨테이너 (그룹/공개 채널)
- 권한 격리: 파일 시스템, 네트워크, 자격증명
