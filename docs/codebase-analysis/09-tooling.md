# 09. 개발 / 테스트 / 배포 도구체인

## 패키지 매니저 + 런타임

### 기본 권장 — pnpm + Node 22+
- `pnpm install`, `pnpm build`, `pnpm test`
- workspace 효율성, deduplicated `node_modules`

### 대안 — Bun 1.3.13+
- `AGENTS.md:53`: "Keep Node + Bun paths working"
- 매 릴리스에서 두 런타임 모두 검증

### 비지원
- npm, yarn (테스트되지 않음)

## 정적 분석 도구 (Rust 기반, 빠름)

OpenClaw는 의도적으로 **빠른 Rust 기반 도구**를 선호:

### 타입체크 — `tsgo`
- Microsoft TypeScript Go port (실험적이지만 매우 빠름)
- 증분 타입체크
- `tsc --noEmit` **금지** (`AGENTS.md:67`)

```bash
pnpm tsgo:all              # 전체
pnpm tsgo:check-prod       # 프로덕션만
pnpm check:test-types      # 테스트
```

### 린트 — `oxlint`
- Rust 기반 ESLint 호환 린터
- ESLint 대비 50–100x 빠름

```bash
pnpm lint:all
pnpm lint:fix
# 직접: scripts/run-oxlint.mjs
```

### 포맷터 — `oxfmt`
- Prettier 대체
- 더 빠르고 일관성 있음

```bash
pnpm format:check
pnpm format

# 특정 파일
pnpm exec oxfmt --check --threads=1 src/foo.ts
pnpm exec oxfmt --write --threads=1 src/foo.ts
```

`AGENTS.md:65`:
> Formatting: use `oxfmt`, not Prettier.

### 테스트 — Vitest
- Jest 호환 API
- ESM 네이티브
- HMR 같은 빠른 watch 모드

```bash
pnpm test                       # 전체
pnpm test:changed               # 변경 파일 기반
pnpm test:serial                # 직렬 (메모리 압박 시)
pnpm test:coverage              # 커버리지
pnpm test:extensions            # extensions/ 만
pnpm test extensions/<id>       # 특정 플러그인
pnpm test <path-or-filter>      # 임의 필터
```

규칙:
- 절대 raw `vitest` 호출 금지 (workspace cache 충돌)
- Jest flag 금지 (`--runInBand` 등)
- 직렬 실행: `pnpm test:serial` 또는 `OPENCLAW_VITEST_MAX_WORKERS=1`

## 변경 기반 게이트 (Smart Gates)

핵심 효율 도구:

### `pnpm check:changed`
- 변경된 파일 분석 → 필요한 lane만 실행
- core / extension / SDK / config 별로 다른 lane

### Lane 분류

`AGENTS.md:152-160`:
| 변경 영역 | 트리거되는 검증 |
|----------|---------------|
| core prod | core prod typecheck + core tests |
| core tests | core test typecheck/tests |
| extension prod | extension prod typecheck + extension tests |
| extension tests | extension test typecheck/tests |
| public SDK/plugin contract | extension prod/test도 |
| unknown root/config | 모든 lane |

### Staged preview
```bash
pnpm check:changed --staged    # commit 직전 체크
pnpm changed:lanes --json      # 어떤 lane이 트리거되는지 explain
```

### 환경변수
- `OPENCLAW_LOCAL_CHECK=1` — 로컬에서 헤비 체크 활성
- `OPENCLAW_LOCAL_CHECK_MODE=throttled|full` — 모드
- CI/공유 환경: `OPENCLAW_LOCAL_CHECK=0`

## 원격 검증 인프라

### Crabbox — 라이브 시나리오
- Linux/Windows/macOS 풀
- WebVNC로 화면 직접 보기
- 라이브 채널/디바이스 시나리오

```bash
crabbox user@scenario --ref main --os windows
```

### Testbox (via Blacksmith)
- Fly.io 분산 컴퓨팅
- 전체 테스트 스위트 병렬화

```bash
# Pre-warm
blacksmith testbox warmup ci-check-testbox.yml --ref main --idle-timeout 90
# → tbx_abc123 ID 반환

# Run
blacksmith testbox run --id tbx_abc123 \
  "env NODE_OPTIONS=--max-old-space-size=4096 \
       OPENCLAW_TEST_PROJECTS_PARALLEL=6 \
       OPENCLAW_VITEST_MAX_WORKERS=1 \
       pnpm test"

# Download artifacts
blacksmith testbox download --id tbx_abc123
```

### Timeout 등급
| 등급 | 시간 |
|------|------|
| 기본 | 90분 |
| 멀티시간 | 240분 |
| All-day | 720분 |
| Overnight | 1440분 |
| 그 이상 | 명시 승인 + cleanup 필요 |

### 사용 정책

`AGENTS.md:170`:
> Testbox use: Broad fan-out commands such as `pnpm check`, full `pnpm test`, Docker/E2E/live/package/build gates ... belong in Testbox by default.

- 대규모 검증 → Testbox
- 좁은 타깃 (`pnpm test src/foo.test.ts`) → 로컬
- 광범위 fan-out 발견되면 → 중단 후 Testbox로 이동

## 빌드

### Production build
```bash
pnpm build
# → dist/index.js (Node entry)
# → 번들된 ESM
```

### Hard build gate

`AGENTS.md:172`:
> Hard build gate: `pnpm build` before push if build output, packaging, lazy/module boundaries, or published surfaces can change.

- 빌드 산출물, 패키징, lazy 경계, 공개 표면 변경 시 push 전 `pnpm build` 필수
- 동적 import 검증 (`[INEFFECTIVE_DYNAMIC_IMPORT]` 경고)

### Architecture 검증
```bash
pnpm check:architecture        # 계층 경계
pnpm check:import-cycles       # 순환 의존
pnpm config:docs:gen           # 설정 docs 재생성
pnpm config:docs:check         # 동기 검증
pnpm plugin-sdk:api:gen        # SDK API
pnpm plugin-sdk:api:check
```

### 생성 파일 추적

```
docs/.generated/*.sha256       # 변경 해시만 git 추적
docs/.generated/*.json         # 본체는 .gitignore
```

## 배포

### Docker

`Dockerfile`:
```dockerfile
# Stage 1: extension deps
FROM node:24-bookworm AS ext-deps
COPY extensions/ /work/extensions/
# 선택적 추출

# Stage 2: build
FROM node:24-bookworm AS build
COPY . /work
RUN pnpm install
RUN pnpm build
RUN pnpm canvas:a2ui:bundle  # 또는 stub

# Stage 3: runtime
FROM node:24-bookworm-slim AS runtime
COPY --from=build /work/dist /app/dist
CMD ["node", "/app/dist/index.js", "gateway", "--allow-unconfigured", "--port", "3000"]
```

### Docker 빌드 인자
```bash
docker build \
  --build-arg OPENCLAW_EXTENSIONS="telegram,discord,anthropic,openai" \
  -t openclaw .
```

선택적 플러그인만 포함 → 이미지 크기 ↓.

### Fly.io

`fly.toml`:
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
  destination = "/data"   # ~/.openclaw + workspace 영속화
```

### Render

`render.yaml`:
```yaml
services:
  - type: web
    name: openclaw
    runtime: node
    plan: starter
    envVars:
      - key: OPENCLAW_STATE_DIR
        value: /data/.openclaw
    disk:
      name: data
      mountPath: /data
      sizeGB: 1
```

### npm Release

- `package.json:version`을 CalVer로 bump
- `pnpm publish` (또는 GitHub Action)
- 베타: `vYYYY.M.D-beta.N` → `npm publish --tag beta`
- 안정: `latest` 태그

`AGENTS.md:188`:
> Releases/publish/version bumps need explicit approval. Release docs: `docs/reference/RELEASING.md`; use `$openclaw-release-maintainer`.

릴리스는 메인테이너만 수행.

## CI 워크플로우 (`.github/workflows/`)

### 주요 워크플로우

| 워크플로우 | 트리거 | 역할 |
|-----------|--------|------|
| `ci.yml` | 모든 PR | 기본 CI (typecheck, lint, test, build) |
| `ci-check-testbox.yml` | PR | Testbox 분산 테스트 |
| `docker-release.yml` | release tag | Docker 이미지 (DockerHub + GHCR) |
| `macos-release.yml` | release tag | macOS 앱 + Sparkle appcast |
| `npm-release.yml` | release tag | npm publish |
| `install-smoke.yml` | nightly | 전 세계 설치 시나리오 |
| `full-release-validation.yml` | manual | 전체 검증 |
| `package-acceptance.yml` | PR | 설치 가능 패키지 검증 |
| `qa-lab.yml` | manual | QA 라이브 채널 |
| `parity-gate.yml` | PR | 변경 lane 일관성 |

### CI Wait 매트릭스

`AGENTS.md:115-122`:

```
- never: Auto response, Labeler, Docs Sync Publish Repo, Stale, ...
- conditional:
    CI: 정확한 SHA만
    Docs: 로컬 docs proof 없을 때만
    Workflow Sanity: 워크플로우/composite 변경 시
    Plugin NPM Release: 플러그인 패키지 변경 시
- release/manual only:
    Docker Release, OpenClaw NPM Release, macOS Release, ...
- explicit/surface only:
    QA-Lab, Scheduled Live And E2E, CodeQL, ...
```

매 PR마다 모든 워크플로우 기다리지 않음 — 변경된 표면에 따라 선택적.

### CI Polling

```bash
# 정확한 SHA로 폴링
gh api repos/openclaw/openclaw/actions/runs/<id> \
  --jq '{status,conclusion,head_sha,updated_at,name,path}'

# 30-60초 간격, 실패/완료 후에만 jobs/logs 조회
```

### Full Release Validation

특수 SHA 처리:
```bash
pnpm ci:full-release --sha <sha>
# GitHub dispatch는 raw SHA를 ref로 못 받으므로
# 임시 pinned 브랜치를 만들고 child headSha 검증
```

## Git Hooks (`git-hooks/`)

### Pre-commit
- 스테이지된 파일 포맷팅만 (전체 검증 X)
- `oxfmt` 자동 적용

### Commit
- `scripts/committer "msg" file1 file2` 권장
- 컨벤션: 단순/간결/그룹화

```bash
scripts/committer "fix: telegram message edits" extensions/telegram/src/outbound.ts
```

## 보안 / Secrets

### 자격증명 위치
```
~/.openclaw/
├── credentials/                  # 채널 auth
│   ├── telegram.json
│   ├── discord.json
│   └── ...
├── agents/
│   └── <agentId>/
│       └── agent/
│           └── auth-profiles.json   # 모델 auth
└── workspace/                    # 사용자 데이터
```

### 환경변수
- `~/.profile`에 키 보관 (`AGENTS.md:182`)
- 일반: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` 등
- OpenClaw 전용: `OPENCLAW_STATE_DIR`, `OPENCLAW_WORKSPACE_DIR`

### 절대 커밋 금지
- 실제 전화번호, 비디오, 자격증명, 라이브 설정
- `.env` (있는 경우 `.gitignore`)

## 문서 / Changelog

### 위치
- 사용자 문서: `docs/`
- 개발자 문서: `AGENTS.md`, `CONTRIBUTING.md`
- 비전: `VISION.md`
- 변경: `CHANGELOG.md`

### Changelog 규칙

`AGENTS.md:177`:
> Changelog bullets are always single-line. No wrapping/continuation across multiple lines.

```markdown
## 2026.5.6

### Changes
- New: Add Zalo Personal channel support. Thanks @username
- Update: Improve memory recall latency by 200ms

### Fixes
- Fix: Telegram bot token validation
```

규칙:
- 한 줄 단위 (다음 줄 들여쓰기 X)
- `Thanks @author` (인간 GitHub 계정)
- `Thanks @codex` 등 봇 이름 금지
- 사용자 영향 없는 변경은 changelog 없음 (test/internal)

## Doctor

진단/복구 도구:
```bash
openclaw doctor                # 진단만
openclaw doctor --fix          # 자동 수정
openclaw doctor --fix=<rule>   # 특정 규칙
```

레거시 설정 마이그레이션:
- runtime이 옛 포맷 처리 X
- 대신 `doctor --fix`가 정규 contract로 변환
- runtime은 깨끗한 contract만 알면 됨

## Generated/API Drift

```bash
pnpm check:architecture            # 계층 경계
pnpm config:docs:gen               # 설정 docs 재생성
pnpm config:docs:check             # 동기 확인
pnpm plugin-sdk:api:gen            # API 추출
pnpm plugin-sdk:api:check
pnpm canvas:a2ui:bundle            # A2UI 번들
```

생성된 파일은 SHA만 추적, 실제 JSON은 `.gitignore`.
