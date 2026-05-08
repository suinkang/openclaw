# 10. 핵심 설계 철학과 보안 모델

`AGENTS.md`는 단순 가이드라인이 아니라 OpenClaw의 설계 헌법입니다. 본 문서는 그 핵심 원칙을 정리합니다.

## 1. Core는 확장에 무관 (Extension-Agnostic)

`AGENTS.md:26`:
> Core stays extension-agnostic. No bundled ids in core when manifest/registry/capability contracts work.

### 의미
Core 코드는 어떤 채널/프로바이더/도구가 존재하는지 **하드코딩 금지**. 모든 확장 정보는 매니페스트로부터 동적으로 들어와야 합니다.

### 구체 규칙

```typescript
// ❌ 금지: core에서 특정 ID 분기
function processChannel(channelId: string) {
  if (channelId === "telegram") return processTelegram();
  if (channelId === "discord") return processDiscord();
}

// ✅ 권장: 매니페스트/레지스트리 조회
function processChannel(channelId: string, registry: ChannelRegistry) {
  const channel = registry.get(channelId);
  return channel.process();
}
```

### 예외
- 아예 channel-specific 동작이 필요하면 → 그 동작을 해당 extension으로 이동
- "여러 owner가 필요한 generic seam"인 경우만 core에 추가

## 2. Plugin SDK는 유일한 공개 경계

`AGENTS.md:27`:
> Extensions cross into core only via `openclaw/plugin-sdk/*`, manifest metadata, injected runtime helpers, documented barrels (`api.ts`, `runtime-api.ts`).

### 강제 규칙

```
확장 prod 코드 금지 import:
  - core src/**
  - src/plugin-sdk-internal/**
  - 다른 확장 src/**
  - 패키지 외부 상대 경로

core/tests 금지 import:
  - 깊은 플러그인 내부 (extensions/*/src/**)
  - onboard.js 같은 internal entry
```

위반 시 `pnpm check:architecture`가 빌드 실패. madge로 검증.

### 진입점

플러그인 작성자는 항상:
```typescript
import { defineChannelPlugin } from "openclaw/plugin-sdk";
// 절대 from "../../src/..."
```

## 3. Owner Boundary

`AGENTS.md:32`:
> Owner boundary: fix owner-specific behavior in the owner module.

### 원칙
- Telegram 버그 → `extensions/telegram/`에서 수정
- Anthropic 인증 문제 → `extensions/anthropic/`에서 수정
- 공통 seam 추가는 **여러 owner가 필요할 때만**

### 안티 패턴
```typescript
// ❌ core에서 owner-specific 처리
// src/agents/runtime.ts
if (provider === "anthropic" && model.startsWith("claude-3-7")) {
  // claude-3-7 specific workaround
}

// ✅ owner module에서 처리
// extensions/anthropic/stream-wrappers.ts
function handleClaude37Quirks(stream) { /* ... */ }
```

## 4. Prepared Runtime Facts

`AGENTS.md:40`:
> Request-time runtime resolution: when a path already knows the provider id, model ref, channel id, ..., carry that as a prepared runtime fact instead of rediscovering it later.

### 핵심
요청 처리 hot path에서 broad 발견(discovery) 금지. 시작/dispatch 시점에 결정한 사실을 컨텍스트로 전달.

### Prepared facts 예
- `AgentRuntimePlan`
- `ProviderRuntimePluginHandle`
- 활성/런타임 레지스트리
- 매니페스트/공개 artifact 조회
- single-provider resolver
- lazy 레지스트리 생성

### 금지 함수 (hot path에서)
```
loadOpenClawPlugins()
resolveProviderPluginsForHooks()
resolvePluginCapabilityProviders()
resolvePluginDiscoveryProvidersRuntime()
getChannelPlugin()
broad model/tool/media registry builders
```

이 함수들은 startup/setup/admin/legacy 경로에서만 사용.

### 안티 패턴 해결법
- 산발적 캐시 레이어 추가 ❌
- → canonical fact를 더 일찍 옮기기 ✓
- → 기존 prepared-runtime 객체 재사용 ✓
- → 마지막 마이그레이션 caller가 사라지면 중복 lookup branch 삭제 ✓

## 5. Additive Protocol Changes

`AGENTS.md:45`:
> Gateway protocol changes: additive first; incompatible needs versioning/docs/client follow-through.

### Additive (자유롭게 가능)
- 선택적 필드
- 새 메서드/이벤트
- 새 enum 값 (기본/폴백 정의 시)

### Breaking (큰 의식 필요)
- 필드 제거/이름 변경
- 필수 필드 추가
- 시그니처 변경

Breaking은 새 메서드 이름 + 구버전 deprecation 유지 + 메이저 bump.

## 6. No Legacy Compatibility in Hot Paths

`AGENTS.md:34`:
> No legacy compatibility in core/runtime paths. When old config/store shapes need support, add an `openclaw doctor --fix` rewrite/repair rule.

### 패턴
- runtime은 **canonical contract**만 알기
- 옛 포맷 마이그레이션은 `openclaw doctor --fix`가 처리
- runtime은 항상 깨끗한 contract만 받음

### 이유
- runtime 코드 단순화
- 매 요청마다 마이그레이션 비용 X
- 명시적 fix 단계가 디버깅 가능

## 7. Prompt Cache Determinism

`AGENTS.md:48`:
> Prompt cache: deterministic ordering for maps/sets/registries/plugin lists/files/network results before model/tool payloads. Preserve old transcript bytes when possible.

### 이유
LLM 프롬프트 캐싱은 **바이트 정확도** 비교. 캐시 히트율이 비용에 직결.

### 위반 vs 준수

```typescript
// ❌ 캐시 미스 (Object/Set 순서 비결정적)
const tools = Object.values(toolRegistry);
const plugins = Array.from(pluginsSet);
const prompt = JSON.stringify({ tools, plugins, ... });

// ✅ 캐시 히트 (결정적 정렬)
const tools = Object.values(toolRegistry).sort((a, b) => 
  a.id.localeCompare(b.id)
);
const plugins = Array.from(pluginsSet).sort();
const prompt = JSON.stringify({ tools, plugins, ... });
```

### 적용 영역
- 도구 목록
- 플러그인 목록
- 파일 결과
- 네트워크 결과
- Map / Set / Registry

매 모델/도구 페이로드 전에 정렬 필수.

## 8. 보안 / 권한 / DM 정책

### Credential 위치
```
~/.openclaw/
├── credentials/                          # 채널 auth (plaintext, 파일 권한 600 권장)
│   ├── telegram.json
│   ├── discord.json
│   └── ...
└── agents/<agentId>/agent/
    └── auth-profiles.json                # 모델 auth
```

### 절대 커밋 금지

`AGENTS.md:181`:
> Never commit real phone numbers, videos, credentials, live config.

### DM 정책 기본값

```yaml
channels:
  telegram:
    dmPolicy: "pairing"          # 기본값 — 안전
    allowFrom: ["explicit_id"]
```

| 정책 | 의미 | 보안 |
|------|------|------|
| `pairing` | 미지인은 일회성 코드 인증 | 안전 (기본) |
| `open` | 모든 DM 허용 | 명시 옵트인 필요 |
| `allowFrom: ["..."]` | 화이트리스트 | 가장 안전 |
| `allowFrom: ["*"]` | 전체 허용 | 위험 |

### Sandbox 모드

```yaml
agents:
  defaults:
    sandbox:
      mode: "non-main"           # Docker 컨테이너
```

- `main` — 메인 프로세스 (개인 DM)
- `non-main` — Docker 격리 (그룹/공개)
- 자격증명/파일 시스템 분리

## 9. 의존성 정책

`AGENTS.md:185`:
> Dependency patches/overrides/vendor changes need explicit approval. `pnpm.patchedDependencies` exact versions only.

- 패치는 정확한 버전만
- 메인테이너 승인 필수

`AGENTS.md:189`:
> Carbon pins owner-only: do not change `@buape/carbon` unless Shadow asks.

특정 라이브러리는 owner만 변경 가능.

## 10. 의존성 owner 따라가기

`AGENTS.md:33`:
> Dependency ownership follows runtime ownership: extension-only deps stay plugin-local; root deps only for core imports or intentionally internalized bundled plugin runtime.

### 규칙
- Extension-only 의존성 → 그 extension의 `package.json`
- Root `package.json`은 core import 또는 의도적 internalized 번들 plugin runtime만

이유:
- 미사용 플러그인 사용자가 불필요한 의존성 안 받음
- 플러그인 분리 가능
- 라이선스/보안 추적 단순

## 11. Secrets / Token 처리

### 채널 토큰
- `~/.openclaw/credentials/<channel>.json`
- 절대 컨텍스트/로그/응답에 노출 금지
- 디버깅 시 `redacted` 처리

### 모델 API key
- `~/.openclaw/agents/<id>/agent/auth-profiles.json`
- OAuth refresh 자동
- 만료 시 사용자 재인증 안내

### 환경변수
- `~/.profile`에 두기
- `AGENTS.md:182`: "Env keys: check `~/.profile`"

## 12. EXFOLIATE! 슬로건

README.md의 "EXFOLIATE! EXFOLIATE!"는 옛 Claw 게임의 명령어 오마주.

해석:
- **벗겨내라** — 불필요한 추상화/레이어 제거
- 반복적으로, 깨끗하게 작동
- "Personal AI assistant you run on your own devices" — 자체 호스팅 철학

"단순함을 향해 깎아내라"는 디자인 메타포.

## 13. 단일 사용자 / 자체 호스팅

OpenClaw는 의도적으로 **단일 사용자** 어시스턴트:
- 멀티 테넌트 X
- 클라우드 SaaS X
- 자체 데이터 / 자체 모델 / 자체 채널

`README.md:21`:
> **OpenClaw** is a _personal AI assistant_ you run on your own devices.

이 결정이 다음을 가능하게 함:
- 더 풍부한 권한 (전화번호, 메모리, 캘린더 등)
- 사용자 데이터가 자기 인프라에 머물기
- 채널 봇 토큰을 사용자가 직접 소유
- 모델 비용을 사용자가 직접 결제

## 14. Generic Seam 추가 시점

`AGENTS.md:32`:
> Shared/core gets generic seams only; no owner ids, dependency strings, defaults, migrations, or recovery policy.

### 새 seam이 OK인 시점
- 여러 owner가 동일 패턴 필요
- 진정으로 generic (특정 owner 의존 X)

### 새 seam이 안 되는 시점
- 한 owner만 필요 → 그 owner module로
- generic이지만 default/migration 포함 → owner-specific

## 15. Tests as Behavior

`AGENTS.md:138`:
> Avoid brittle tests that grep workflow/docs strings for operator policy. Prefer executable behavior, parsed config/schema checks, or live run proof.

### 좋은 테스트
- 실제 동작 검증 (LLM mock 응답 → 채널 outbound 확인)
- 파싱된 config/schema 검사
- 라이브 run proof

### 나쁜 테스트
- 워크플로우 YAML grep
- docs 문자열 grep
- 실행 안 한 채 정책 가정

## 16. Memory Hygiene

플러그인이 메모리에 남기는 흔적:
- 자격증명 캐시 → 명시 만료
- 세션 상태 → 정해진 lifecycle
- 모듈 캐시 → 테스트에서 정리

`AGENTS.md:147`:
> Clean timers/env/globals/mocks/sockets/temp dirs/module state; `--isolate=false` safe.

테스트는 깨끗한 종료 (timer, env, mock, socket, temp 모두 cleanup) → 빠른 vitest `--isolate=false` 가능.

## 17. 사람 > 봇 (Credit)

`AGENTS.md:177`:
> ... using credited human GitHub username(s). Never add `Thanks @codex`, `Thanks @openclaw`, `Thanks @clawsweeper`, or `Thanks @steipete`.

Changelog credit는 항상 인간. 봇/자동화/메인테이너 본인 X. 인간 미상이면 빈 칸이 낫다 (가짜 추측 금지).

## 정리

OpenClaw 설계의 모든 길은 다음 4가지로 환원:

1. **분리 (Separation)** — Core / SDK / Plugins / Apps의 명확한 경계
2. **준비 (Preparation)** — Hot path는 prepared facts, broad discovery는 startup만
3. **결정성 (Determinism)** — Cache, ordering, behavior 모두 결정적
4. **안전 (Safety)** — DM 정책, sandbox, credential 격리, manifest 검증
