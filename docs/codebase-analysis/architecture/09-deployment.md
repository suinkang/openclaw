# 09. Deployment View

OpenClaw의 배포 토폴로지 — UML Deployment Diagram.

## 1. 기본 배포 시나리오 — 로컬 단일 머신

가장 일반적인 배포: 사용자 데스크탑에서 모든 컴포넌트 실행.

```mermaid
flowchart TB
    subgraph Mobile["📱 Mobile Devices (옵션)"]
        iOS["iOS App<br/>(node mode)"]
        Android["Android App<br/>(node mode)"]
    end
    
    subgraph Desktop["💻 사용자 데스크탑/Mac/Linux"]
        subgraph Apps["Native Apps"]
            macOSApp["macOS 앱<br/>(SwiftUI)"]
            CLI["openclaw CLI"]
            WebBrowser["Browser →<br/>Web UI"]
        end
        
        subgraph GW["Gateway Process (Node 22+)"]
            GWMain["Gateway Server<br/>:18789 (ws)"]
            HTTPSrv["HTTP Server<br/>(Canvas, A2UI)"]
        end
        
        subgraph SubProc["Subprocess (필요시)"]
            DockerD["Docker Sandbox<br/>(non-main agents)"]
            IMSGD["imsg CLI<br/>(JSON-RPC stdio)"]
            MCPD["MCP Server<br/>(stdio)"]
        end
        
        subgraph FS["File System (~/.openclaw)"]
            ConfigF[(openclaw.json)]
            SessF[(sessions.json + JSONL)]
            AuthF[(auth-profiles.json)]
            PluginDB[(plugin-state.sqlite)]
            MemF[(memory/)]
            CronF[(cron/)]
        end
    end
    
    iOS -.->|Tailscale<br/>or local network| GWMain
    Android -.->|Tailscale<br/>or local network| GWMain
    
    macOSApp <-->|ws://localhost:18789| GWMain
    CLI <-->|ws://localhost:18789| GWMain
    WebBrowser <-->|ws://localhost:18789| GWMain
    
    GWMain --> SubProc
    GWMain <--> FS
    
    GWMain -.->|HTTPS| Internet[(인터넷)]
    Internet -.->|Telegram| ExtTG[Telegram API]
    Internet -.->|Discord| ExtDC[Discord API]
    Internet -.->|Anthropic| ExtAN[Anthropic API]
    Internet -.->|OpenAI| ExtOA[OpenAI API]
    
    style Desktop fill:#FFE4B5
    style Mobile fill:#FFB6C1
    style GW fill:#FFFACD
    style FS fill:#F0E68C
    style SubProc fill:#D8BFD8
```

### 특징

- 모든 데이터 로컬 파일 시스템에 저장
- `ws://localhost:18789`은 loopback only → 인증 없이 사용 가능
- 모바일은 LAN 또는 Tailscale 통해 접근

---

## 2. 클라우드 배포 — Fly.io

원격 머신에서 Gateway 실행 (모바일 우선 시나리오).

```mermaid
flowchart TB
    subgraph User["👤 사용자"]
        iOS_Mobile["iOS App"]
        Android_Mobile["Android App"]
        Desktop_Browser["Browser<br/>(웹 UI)"]
    end
    
    subgraph FlyEdge["Fly.io Edge (region: iad)"]
        FlyProxy["Fly Proxy<br/>(force_https=true)"]
    end
    
    subgraph FlyMachine["Fly.io Machine"]
        Container["Docker Container<br/>(node:24-bookworm-slim)"]
        Container --> GWApp["Gateway App<br/>:3000"]
        Container --> Volume["/data volume<br/>(openclaw_data)"]
    end
    
    Volume -.->|persistent| StoragePath[".openclaw/<br/>config, sessions, auth"]
    
    iOS_Mobile -->|wss://| FlyProxy
    Android_Mobile -->|wss://| FlyProxy
    Desktop_Browser -->|wss://| FlyProxy
    
    FlyProxy --> Container
    
    GWApp -.->|HTTPS| Internet[(인터넷)]
    Internet -.-> ExtAPIs[External APIs]
    
    style FlyEdge fill:#87CEEB
    style FlyMachine fill:#FFE4B5
    style User fill:#FFB6C1
```

### Fly.io 설정 (`fly.toml`)

```toml
app = "openclaw"
primary_region = "iad"

[http_service]
  internal_port = 3000
  force_https = true

[processes]
  app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"

[mounts]
  source = "openclaw_data"
  destination = "/data"

[env]
  OPENCLAW_STATE_DIR = "/data/.openclaw"
```

### 보안 추가 사항

- **TLS** (`wss://`) 필수
- **Auth token** 필수 (loopback 아님)
- 페어링 코드로 모바일 등록
- Tailscale 옵션도 가능

---

## 3. 컨테이너화 (Docker)

`Dockerfile`은 다단계 빌드:

```mermaid
flowchart TB
    subgraph Build1["Stage 1: ext-deps"]
        Stage1["선택적 확장의<br/>package.json 추출"]
    end
    
    subgraph Build2["Stage 2: build"]
        Stage2A["FROM node:24-bookworm"]
        Stage2B["pnpm install"]
        Stage2C["pnpm build"]
        Stage2D["pnpm canvas:a2ui:bundle"]
        Stage2A --> Stage2B --> Stage2C --> Stage2D
    end
    
    subgraph Build3["Stage 3: runtime"]
        Stage3A["FROM node:24-bookworm-slim"]
        Stage3B["COPY dist + node_modules"]
        Stage3C["CMD node dist/index.js gateway ..."]
        Stage3A --> Stage3B --> Stage3C
    end
    
    Build1 --> Build2 --> Build3
    
    style Build3 fill:#90EE90
```

### 선택적 플러그인 빌드

```bash
docker build \
  --build-arg OPENCLAW_EXTENSIONS="telegram,discord,anthropic,openai" \
  -t openclaw .
```

→ 사용 안 하는 플러그인 제외 → 이미지 크기 ↓

### Docker Compose 예

```yaml
# docker-compose.yml
version: "3.8"
services:
  openclaw:
    image: openclaw:latest
    ports:
      - "3000:3000"
    volumes:
      - openclaw-data:/data
    environment:
      OPENCLAW_STATE_DIR: /data/.openclaw
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}
    restart: unless-stopped

volumes:
  openclaw-data:
```

---

## 4. Render 배포

```yaml
# render.yaml
services:
  - type: web
    name: openclaw
    runtime: node
    plan: starter
    buildCommand: "pnpm install && pnpm build"
    startCommand: "node dist/index.js gateway --allow-unconfigured --port $PORT"
    envVars:
      - key: OPENCLAW_STATE_DIR
        value: /data/.openclaw
      - key: NODE_VERSION
        value: 22
    disk:
      name: data
      mountPath: /data
      sizeGB: 1
```

### 특징

- starter plan (저렴, 슬립 모드)
- 1GB 디스크 (제한적)
- 자동 HTTPS

---

## 5. 모바일 앱 배포

### 5.1 macOS — Sparkle 자동 업데이트

```mermaid
sequenceDiagram
    participant App as macOS App
    participant Sparkle
    participant Appcast as appcast.xml
    participant DMG as Release DMG
    
    App->>Sparkle: 백그라운드 update 체크
    Sparkle->>Appcast: 최신 버전 확인
    Appcast-->>Sparkle: latest version + DSA 서명
    
    alt 새 버전 있음
        Sparkle->>App: notify user
        App->>Sparkle: 사용자 동의
        Sparkle->>DMG: 다운로드
        DMG-->>Sparkle: bytes + signature
        Sparkle->>Sparkle: DSA 서명 검증
        Sparkle->>Sparkle: 자동 설치 + 재시작
    end
```

`appcast.xml` 위치: 레포 루트.

### 5.2 iOS — App Store

- TestFlight (베타)
- App Store (정식 릴리스)
- Watch App + Share Extension 동시 배포

### 5.3 Android — APK / Play Store

- APK direct (사이드로드)
- Play Store (정식)
- Foreground service 권한 필요

---

## 6. CI / CD Pipeline

```mermaid
flowchart TB
    PR[PR 생성]
    PR --> Smart{check:changed lane}
    
    Smart --> CoreL[core lane]
    Smart --> ExtL[extension lane]
    Smart --> SDKL[SDK lane]
    
    CoreL --> Tests[ci.yml<br/>typecheck + lint + test]
    ExtL --> Tests
    SDKL --> Tests
    
    Tests --> CITestbox[ci-check-testbox.yml<br/>분산 테스트]
    
    CITestbox --> Pass{Pass?}
    Pass -->|no| Fail[빌드 실패]
    Pass -->|yes| Land[main에 land]
    
    Land --> ReleaseFlow[Release flow]
    ReleaseFlow --> NPMRelease[npm release.yml]
    ReleaseFlow --> DockerRelease[docker-release.yml]
    ReleaseFlow --> macOSRelease[macos-release.yml]
    
    NPMRelease --> NPMRegistry[npm Registry]
    DockerRelease --> DH[DockerHub + GHCR]
    macOSRelease --> Sparkle[Sparkle appcast<br/>+ DMG]
    
    style Fail fill:#FFB6C1
    style Pass fill:#90EE90
    style Land fill:#FFE4B5
```

### 주요 워크플로우

| 워크플로우 | 트리거 | 역할 |
|----------|--------|------|
| `ci.yml` | 모든 PR | 기본 CI |
| `ci-check-testbox.yml` | PR | Testbox 분산 |
| `docker-release.yml` | release tag | DockerHub + GHCR |
| `macos-release.yml` | release tag | macOS DMG + Sparkle |
| `npm-release.yml` | release tag | npm |
| `install-smoke.yml` | nightly | 설치 시나리오 |
| `full-release-validation.yml` | manual | 전체 검증 |
| `package-acceptance.yml` | PR | 설치 가능 패키지 검증 |
| `qa-lab.yml` | manual | 라이브 채널 QA |
| `parity-gate.yml` | PR | 변경 lane 일관성 |

### Wait Matrix (`AGENTS.md:115`)

```
- never: Auto response, Labeler, Stale, ...
- conditional: CI (exact SHA만), Docs (docs 변경 시), ...
- release/manual only: Docker Release, macOS Release, ...
- explicit/surface only: QA-Lab, CodeQL, ...
```

매 PR마다 모든 워크플로우 기다리지 않음 (선택적).

---

## 7. Crabbox / Testbox / Blacksmith

### 7.1 Crabbox (라이브 시나리오)

```mermaid
flowchart LR
    Dev[개발자] --> CrabboxCmd[crabbox user@scenario]
    CrabboxCmd --> Pool[Crabbox Pool<br/>(Linux/Windows/macOS)]
    Pool --> VM[가상 머신]
    VM --> WebVNC[WebVNC 화면]
    WebVNC --> Dev
    
    style Pool fill:#87CEEB
    style VM fill:#FFE4B5
```

OS별 시나리오 검증:
- Windows-only 버그
- macOS Voice Wake
- Linux Discord 통합 등

### 7.2 Testbox (분산 CI)

```bash
# Pre-warm
blacksmith testbox warmup ci-check-testbox.yml --ref main --idle-timeout 90

# Run
blacksmith testbox run --id tbx_xxx \
  "env NODE_OPTIONS=--max-old-space-size=4096 \
       OPENCLAW_TEST_PROJECTS_PARALLEL=6 \
       OPENCLAW_VITEST_MAX_WORKERS=1 \
       pnpm test"

# Download
blacksmith testbox download --id tbx_xxx
```

| Timeout | 시간 |
|---------|------|
| 기본 | 90분 |
| 멀티시간 | 240분 |
| All-day | 720분 |
| Overnight | 1440분 |

---

## 8. 환경별 배포 매트릭스

| 환경 | 위치 | TLS | Auth | 영속성 |
|------|------|-----|------|--------|
| **로컬 (loopback)** | localhost | ❌ | none/auto | 로컬 ~/.openclaw |
| **LAN (private)** | local network | ❌ | device-token + `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` | 로컬 |
| **Tailscale** | tailnet | ❌ (TS 자체 암호화) | device-token | 로컬 또는 Fly |
| **Fly.io** | edge | ✅ | device-token | Fly volume |
| **Render** | edge | ✅ | device-token | Render disk |
| **Self-hosted (VPS)** | 사용자 서버 | ✅ (사용자 설정) | device-token | 서버 디스크 |

---

## 9. 영속성 / 백업

### 9.1 백업 대상

```mermaid
flowchart TB
    Backup[Backup target<br/>~/.openclaw/]
    Backup --> Cfg["openclaw.json (+ .bak.{0,1,2})"]
    Backup --> Sess["agents/{id}/sessions/<br/>(가장 큼)"]
    Backup --> Auth["agents/{id}/agent/auth-profiles.json<br/>(가장 민감)"]
    Backup --> Cron["cron/jobs.json + jobs-state.json"]
    Backup --> Mem["memory/wiki.md"]
    Backup --> Plugins["plugin-state/state.sqlite"]
```

### 9.2 백업 자동화 (사용자 책임)

OpenClaw는 자체 백업 메커니즘 없음. 사용자가 직접:
- `tar -czf openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw`
- Time Machine (macOS)
- rsync to NAS / cloud

### 9.3 Fly.io 자체 스냅샷

`fly volumes snapshots create openclaw_data` — Fly.io 인프라 레벨.

---

## 10. 모니터링 / 진단

### 10.1 Logs

```
~/.openclaw/logs/
└── ... (구체적 구조는 코드 베이스에서 확인 필요)
```

CLI:
```bash
openclaw doctor                # 진단
openclaw doctor --fix          # 자동 수정
openclaw gateway status --deep # Gateway 헬스체크
./scripts/clawlog.sh           # 로그 tail
```

### 10.2 Startup Trace

```bash
OPENCLAW_GATEWAY_STARTUP_TRACE=1 openclaw gateway start
```

각 단계 (config.snapshot, plugins.bootstrap, ...) timing 측정.

### 10.3 Diagnostics Timeline

```bash
OPENCLAW_DIAGNOSTICS_TIMELINE=1 openclaw gateway start
```

이벤트 루프 + function span을 JSON 형식으로 export.

### 10.4 Test Performance

```bash
pnpm test:perf:imports src/foo.test.ts
pnpm test:perf:hotspots --limit 10
```

---

## 11. 보안 배치 다이어그램

```mermaid
flowchart TB
    subgraph Trust["신뢰 영역 (사용자)"]
        DesktopOS[Desktop OS]
        DesktopOS --> Profile["~/.profile<br/>(API keys)"]
        DesktopOS --> StateDir["~/.openclaw<br/>(0o600)"]
        DesktopOS --> Keychain["macOS Keychain<br/>(device tokens)"]
    end
    
    subgraph Boundary["보안 경계"]
        WSAuth[WebSocket Auth]
        TLS_Ring[TLS Ring]
        SBox[Docker Sandbox]
    end
    
    subgraph Untrusted["비신뢰 영역"]
        ExtAPIs[External APIs]
        IncomingMsgs[Incoming messages<br/>(채널)]
    end
    
    Trust --> Boundary
    Boundary --> Untrusted
    
    Untrusted -.-> InboundCheck{Filter}
    InboundCheck -.->|allowFrom| Boundary
    InboundCheck -.->|deny| Drop[버림]
    
    style Trust fill:#90EE90
    style Boundary fill:#FFE4B5
    style Untrusted fill:#FFB6C1
```

### 보안 레이어

| 레이어 | 메커니즘 |
|--------|---------|
| **Network** | TLS, Tailscale, loopback 분리 |
| **Authentication** | 6가지 모드, `safeEqualSecret` constant-time |
| **Authorization** | scopes 기반, allowFrom |
| **Rate limiting** | scope별 (default, shared-secret, device-token, hook-auth) |
| **Sandbox** | Docker/SSH 격리 (non-main agents) |
| **File permissions** | 0o600 (소유자만) |
| **Secret refs** | env / file / exec (인라인 비밀 거부) |
| **DoS defense** | preauth budget, handshake timeout, payload size limit |

---

## 12. 다중 머신 / HA 시나리오

```mermaid
flowchart TB
    subgraph Primary["Primary Gateway"]
        GWPrime[Gateway #1]
        StorePrime[(Storage)]
    end
    
    subgraph Secondary["Secondary (옵션)"]
        GWSec[Gateway #2]
        StoreSec[(Storage replica)]
    end
    
    Primary -.->|❌ 자동 동기 X| Secondary
    
    Note1[OpenClaw는 단일 사용자<br/>단일 머신 모델<br/>HA 미지원] -.- Secondary
    
    style Primary fill:#90EE90
    style Secondary fill:#FFB6C1
```

OpenClaw는 **단일 머신 단일 사용자**가 의도된 모델. HA / 멀티 노드 지원 없음.

대안:
- 사용자가 직접 백업 + 복원
- Fly.io의 단일 region 영속 볼륨
- (HA 필요 시 직접 인프라 구축)

---

## 13. 업그레이드 전략

```mermaid
flowchart TB
    NewVer[새 버전 릴리스]
    NewVer --> CheckChannel{어느 채널?}
    
    CheckChannel -->|stable| Latest[npm @latest]
    CheckChannel -->|beta| Beta[npm @beta]
    
    Latest --> AutoUp{자동 업데이트?}
    AutoUp -->|macOS| Sparkle[Sparkle 자동]
    AutoUp -->|iOS| AppStore[App Store]
    AutoUp -->|Android| PlayStore[Play Store / APK]
    AutoUp -->|Linux/Server| Manual[수동 npm/docker pull]
    
    Manual --> RunDoctor[openclaw doctor --fix]
    RunDoctor --> Migrate[Config 마이그레이션]
    Migrate --> Verify[verification]
    
    Sparkle --> Verify
    AppStore --> Verify
    PlayStore --> Verify
    
    style Verify fill:#90EE90
```

### 다운그레이드

OpenClaw는 명시적 다운그레이드 지원 없음. 옛 버전 npm install 가능하나 config 호환성 보장 X.

---

## 14. 종합 — 배포 결정 트리

```mermaid
flowchart TD
    Start[배포 결정]
    Start --> Q1{사용자가 모바일만?}
    Q1 -->|yes| Q2{원격 접근 필요?}
    Q1 -->|no, 데스크탑 사용| Local[로컬 데스크탑]
    
    Q2 -->|yes| Cloud[Fly.io / Render]
    Q2 -->|no, LAN만| LAN[로컬 + Tailscale]
    
    Local --> Easy[가장 쉬움<br/>npm install -g openclaw<br/>openclaw onboard]
    
    LAN --> Setup1[로컬 install +<br/>Tailscale 설치]
    
    Cloud --> Q3{어느 클라우드?}
    Q3 -->|Fly.io| Fly[Fly.io<br/>fly.toml 사용]
    Q3 -->|Render| Render[Render<br/>render.yaml 사용]
    Q3 -->|Self-hosted| Docker[Docker<br/>VPS / NAS]
    
    style Easy fill:#90EE90
    style Setup1 fill:#FFFACD
    style Fly fill:#87CEEB
    style Render fill:#87CEEB
    style Docker fill:#FFE4B5
```
