# Paperclip 상세 분석 보고서

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub URL** | https://github.com/paperclipai/paperclip |
| **Stars** | 53,248 |
| **Forks** | 8,940 |
| **언어** | TypeScript 97.1% |
| **LOC** | ~251,900 (`*.ts` + `*.tsx`, repos_applied/paperclip 실측) |
| **서비스 파일** | 74 (server/src/services/) |
| **DB 스키마 테이블** | 60+ (packages/db/src/schema/) |
| **라이선스** | MIT |
| **개발 팀** | paperclipai |
| **최신 릴리스** | v2026.410.0 (security release) |
| **생성일** | 2026-03-02 (약 6주 경과 기준 2026-04-14) |
| **홈페이지** | https://paperclip.ing |
| **태그라인** | "If OpenClaw is an _employee_, Paperclip is the _company_" (README L32) |
| **슬로건** | "Open-source orchestration for zero-human companies" |
| **로컬 경로** | `repos_applied/paperclip/` |

---

## 2. 핵심 특징

Paperclip은 **에이전트 오케스트레이션 메타 레이어**다. 자체 LLM 추론 엔진도, 자체 에이전트 런타임도 가지지 않는다. 대신 기존 Claw 런타임들(OpenClaw, Claude Code, Codex, Cursor, Gemini, OpenCode, pi)을 **직원으로 고용**하고, 그 위에 **회사 구조**를 덮어씌운다: 조직도, 목표, 이슈/티켓, 월간 예산, 승인 게이트, 불변 감사 로그.

README가 이 포지셔닝을 가장 날카롭게 표현한다:

> "If OpenClaw is an _employee_, Paperclip is the _company_."

MetaClaw(R35)가 "1 에이전트 × N 백엔드 스왑"이라면, Paperclip은 **"N 에이전트 × N 런타임 × 영속 조직 구조"**다. 둘은 보완적이다 — MetaClaw가 에이전트 한 명의 두뇌를 교체 가능한 것으로 만든다면, Paperclip은 여러 에이전트(각자 다른 Claw 런타임일 수 있는)를 한 회사의 직원으로 묶는다.

**3-스텝 사용 모델** (README 인용):

| # | 단계 | 예시 |
|---|------|------|
| 01 | Define the goal | "Build the #1 AI note-taking app to $1M MRR." |
| 02 | Hire the team | CEO, CTO, engineers — any bot, any provider. |
| 03 | Approve and run | Review strategy. Set budgets. Hit go. |

Coming Soon으로 예고된 **Clipmart**는 사전 제작된 "회사 템플릿"(조직 구조 + 에이전트 설정 + 스킬)을 한 번의 클릭으로 실행하는 마켓플레이스다. 이는 Claw 생태계 최초의 "회사 단위 공유 아티팩트" 시도다.

---

## 3. 아키텍처

### 디렉토리 구조

```
paperclip/
├── cli/                          # CLI 진입점 (npx paperclipai onboard --yes)
├── server/                       # Node.js/Express 오케스트레이션 API
│   └── src/
│       ├── services/             # 74개 도메인 서비스
│       │   ├── heartbeat.ts      # 4,707 LOC — 에이전트 기동/직렬화/런 관리
│       │   ├── cron.ts           # 373 LOC — 커스텀 zero-dep cron 파서
│       │   ├── issues.ts         # 2,497 LOC — 티켓 CRUD + FOR UPDATE 체크아웃
│       │   ├── budgets.ts        # 958 LOC — 정책 기반 예산 감시 + auto-pause
│       │   ├── company-skills.ts # 2,368 LOC — SKILL.md 발견 + 주입
│       │   ├── approvals.ts      # 272 LOC — 승인 게이트 워크플로
│       │   ├── activity-log.ts   # 불변 감사 로그
│       │   ├── agents.ts         # 에이전트 CRUD + 상태
│       │   ├── companies.ts      # 멀티 테넌트 격리
│       │   └── ... (65개 더)
│       ├── routes/               # 27개 REST 라우트
│       │   ├── heartbeat-runs.ts
│       │   ├── issues.ts
│       │   ├── org-chart-svg.ts  # SVG 조직도 렌더링 엔드포인트
│       │   └── ...
│       └── adapters/             # 어댑터 런타임 로딩
├── packages/
│   ├── adapters/                 # 7개 이종 런타임 어댑터
│   │   ├── claude-local/
│   │   ├── codex-local/
│   │   ├── cursor-local/
│   │   ├── gemini-local/
│   │   ├── openclaw-gateway/
│   │   ├── opencode-local/
│   │   └── pi-local/
│   ├── adapter-utils/            # 어댑터 공통 유틸
│   ├── db/                       # Drizzle 스키마 + 마이그레이션
│   │   └── src/schema/           # 60+ 테이블 (아래 "데이터 모델" 참조)
│   ├── mcp-server/               # MCP 서버 (에이전트가 Paperclip API 호출용)
│   ├── plugins/                  # 플러그인 SDK + 예제
│   │   ├── create-paperclip-plugin/
│   │   ├── sdk/
│   │   └── examples/
│   └── shared/                   # 공유 타입/상수
├── skills/                       # 번들 스킬 (SKILL.md 형식)
│   ├── paperclip/                # 핵심 워크플로 스킬
│   ├── paperclip-create-agent/
│   ├── paperclip-create-plugin/
│   └── para-memory-files/
├── ui/                           # React + Vite 보드 UI
├── docker/                       # Dockerfile, compose 구성
├── docs/                         # Mintlify 문서
├── evals/promptfoo/              # 프롬프트 평가
├── tests/                        # Vitest 단위 + Playwright e2e
├── releases/                     # 릴리스 노트 (v2026.410.0 최신)
└── scripts/                      # 빌드/배포 스크립트
```

### 실행 흐름

```
사용자 → CLI(`npx paperclipai onboard --yes`)
    ↓  임베디드 PGlite(DATABASE_URL 없을 시) 또는 외부 Postgres 기동
    ↓
Server(Express) ↔ UI(React/Vite)
    ↓
[이슈 생성] → [에이전트 할당] → [Heartbeat 기동 트리거]
                                    ↓
                        WakeupOptions.source ∈ {timer, assignment, on_demand, automation}
                                    ↓
                        startLocksByAgent Map<agentId, Promise> 직렬화
                                    ↓
              adapter(claude-local/codex-local/cursor-local/gemini-local/
                      openclaw-gateway/opencode-local/pi-local) 호출
                                    ↓
                        FOR UPDATE 티켓 체크아웃 + checkoutRunId 기록
                                    ↓
                        agentTaskSessions 세션 컨텍스트 재개
                                    ↓
              외부 에이전트 런타임 실행 → 결과 → costEvents 적재
                                    ↓
                        budgetService 주기 평가 → 임계치 초과 시 agent auto-pause
                                    ↓
                        activity_log 불변 감사 기록
```

### 주요 의존성 (server/package.json 기준)

- **ORM**: Drizzle ORM (v0.38.4)
- **DB**: `embedded-postgres` (v18.1.0-beta.16) — 개발용 PGlite, 프로덕션용 외부 Postgres
- **Web**: Express + Pino 로거 + WebSocket
- **Auth**: Better-Auth (이메일/비번, Drizzle adapter)
- **Schedule**: **외부 큐 없음** (Bull/BullMQ/Redis/SQS 미사용) — `cron.ts` 자체 구현
- **Telemetry**: `@paperclipai/shared/telemetry` (CI에서 비활성)

---

## 4. Heartbeat 시스템 — 에이전트 기동 모델

Paperclip은 **에이전트를 상시 실행하지 않는다**. 대신 "심장박동(heartbeat)"이라는 이벤트 단위로 기동시킨다. 이는 ZeroClaw의 세션 스냅샷 + Autoresearch의 `TIME_BUDGET=300s` 루프와 구조적으로 다르다.

### 4개 기동 소스

`WakeupOptions.source` (`server/src/services/heartbeat.ts:349`):

```typescript
interface WakeupOptions {
  source?: "timer" | "assignment" | "on_demand" | "automation";
  triggerDetail?: "manual" | "ping" | "callback" | "system";
  reason?: string | null;
  payload?: Record<string, unknown> | null;
  idempotencyKey?: string | null;
  requestedByActorType?: "user" | "agent" | "system";
  requestedByActorId?: string | null;
  contextSnapshot?: Record<string, unknown>;
}
```

| 소스 | 의미 | 사용 사례 |
|------|------|----------|
| `timer` | cron 스케줄 만기 | "매일 09:00 마케팅 에이전트 점검" |
| `assignment` | 이슈/티켓이 에이전트에 할당됨 | 새 이슈 → 담당자 즉시 기동 |
| `on_demand` | 사용자/에이전트가 명시적으로 wake 호출 | 대시보드 "Ping" 버튼 |
| `automation` | 시스템 이벤트(예: deferred wake 승격) | 차단 이슈 해제 시 자동 재개 |

`triggerDetail`은 4개 소스 각각의 하위 분류(`manual`, `ping`, `callback`, `system`)로, 동일한 `source="on_demand"`를 사용자 수동 호출과 에이전트 콜백으로 구별한다.

### 커스텀 Zero-Dep Cron 파서 (`cron.ts`, 373 LOC)

외부 cron 라이브러리 없이 표준 5필드 cron 표현식(분/시/일/월/요일)을 지원한다:

```
┌────────────── minute (0–59)
│ ┌──────────── hour   (0–23)
│ │ ┌────────── day of month (1–31)
│ │ │ ┌──────── month  (1–12)
│ │ │ │ ┌────── day of week (0–6, Sun=0)
* * * * *
```

지원 구문: `*`, `N`, `N-M`, `N/S`, `*/S`, `N-M/S`, `N,M,...`. `nextCronTick()`은 시간을 전진하며 각 분이 파싱된 cron과 매치되는지 검사한다(4년/~2.1M 반복 바운드로 무한 루프 방지).

**왜 자체 구현했는가**: (1) Node.js 임베디드 Postgres 한 프로세스로 모든 걸 돌리는 제로-의존성 철학, (2) cron 표현식 외에 `assignment`/`on_demand`/`automation` 이벤트와 동일한 파이프라인으로 합류시키기 위해 단순 parser + `nextCronTick` 함수형 인터페이스가 필요했기 때문.

### Per-Agent 직렬화 락

```typescript
// server/src/services/heartbeat.ts:79
const startLocksByAgent = new Map<string, Promise<void>>();

// server/src/services/heartbeat.ts:72-73
const HEARTBEAT_MAX_CONCURRENT_RUNS_DEFAULT = 1;
const HEARTBEAT_MAX_CONCURRENT_RUNS_MAX = 10;
```

한 에이전트에 대해 동시에 최대 1(기본)~10(최대) 개 heartbeat run만 허용한다. `Map<string, Promise<void>>` 기반 인메모리 락이 에이전트 단위 직렬화를 보장한다.

### 세션 재개 (Sessioned Adapters)

```typescript
// server/src/services/heartbeat.ts:86-93
const SESSIONED_LOCAL_ADAPTERS = new Set([
  "claude_local",
  "codex_local",
  "cursor",
  "gemini_local",
  "opencode_local",
  "pi_local",
]);
```

6개 로컬 어댑터는 **이전 heartbeat의 세션 상태를 `agentTaskSessions` 테이블에서 로드해 이어간다**. `openclaw_gateway`는 게이트웨이 자체가 세션을 관리하므로 이 집합에서 제외된다. 이는 session_context_report.md가 분석한 "세션 스냅샷 재개" 모델을 조직 차원으로 끌어올린 형태다.

---

## 5. Ticket / Issue 시스템 — FOR UPDATE 기반 체크아웃

이슈(티켓)는 Paperclip의 실행 단위다. 이 시스템의 핵심은 **크래시 복구까지 고려한 원자적 체크아웃**이다.

### 상태 머신

7개 상태: `backlog → todo → in_progress → in_review → blocked → done / cancelled`. 주요 필드 (`packages/db/src/schema/issues.ts`):

- `id`, `identifier`, `companyId`, `title`, `description`
- `status`, `assigneeAgentId`, `assigneeUserId`
- `executionWorkspaceId`, `executionRunId`, **`checkoutRunId`** (체크아웃 락)
- `parentId`, `projectId`, timestamps, hidden state
- 관계: `issue_labels`, `issue_comments`, `issue_attachments`, `issue_documents`, `issue_read_states`, `issue_relations` (blocks/blocked_by DAG), `issue_approvals`, `issue_execution_decisions`, `issue_work_products`, `issue_inbox_archives`

### `FOR UPDATE` 행락 기반 체크아웃

`server/src/services/issues.ts:823-828`:

```typescript
const lockedIssueIds = [issueId, ...deduped].sort();
await dbOrTx.execute(
  sql`SELECT ${issues.id} FROM ${issues}
      WHERE ${and(eq(issues.companyId, companyId), inArray(issues.id, lockedIssueIds))}
      ORDER BY ${issues.id}
      FOR UPDATE`,
);
```

이슈 ID를 **정렬된 순서로** `FOR UPDATE` 잠근다 — 데드락 회피의 고전적 기법이다. 이슈와 blocked-by 관계 이슈들을 한 트랜잭션에서 동시 잠그고 사이클을 검증한다(`assertNoBlockingCycles`).

### Stale Checkout 복구 — `adoptStaleCheckoutRun`

Claw 생태계 최초의 **DB 네이티브 크래시 복구** 패턴이 여기 있다. `server/src/services/issues.ts:874-909`:

```typescript
async function isTerminalOrMissingHeartbeatRun(runId: string) {
  const run = await db
    .select({ status: heartbeatRuns.status })
    .from(heartbeatRuns)
    .where(eq(heartbeatRuns.id, runId))
    .then((rows) => rows[0] ?? null);
  if (!run) return true;
  return TERMINAL_HEARTBEAT_RUN_STATUSES.has(run.status);
}

async function adoptStaleCheckoutRun(input: {
  issueId: string;
  actorAgentId: string;
  actorRunId: string;
  expectedCheckoutRunId: string;
}) {
  const stale = await isTerminalOrMissingHeartbeatRun(input.expectedCheckoutRunId);
  if (!stale) return null;

  const adopted = await db
    .update(issues)
    .set({
      checkoutRunId: input.actorRunId,
      executionRunId: input.actorRunId,
      executionLockedAt: now,
      updatedAt: now,
    })
    .where(
      and(
        eq(issues.id, input.issueId),
        eq(issues.status, "in_progress"),
        eq(issues.assigneeAgentId, input.actorAgentId),
        eq(issues.checkoutRunId, input.expectedCheckoutRunId),  // CAS 가드
      ),
    )
    .returning(...)
    .then((rows) => rows[0] ?? null);

  return adopted;
}
```

동작 원리:
1. 이전 체크아웃이 참조한 `heartbeatRuns.id`가 **종료/실종 상태**인지 검사
2. 그렇다면 현재 actor가 "입양(adopt)" 시도 — `WHERE checkoutRunId = expectedCheckoutRunId` 조건으로 **CAS(compare-and-swap)** 실행
3. 성공하면 새 `actorRunId`로 교체; 실패하면 다른 actor가 이미 입양한 것이므로 `null` 반환

외부 큐(Redis/SQS) 없이 Postgres 단일 DB만으로 **작업자 크래시 후 안전한 작업 재할당**을 구현한다. 이는 BullMQ의 `stalled job` 복구 로직과 개념상 동등하지만, 별도 브로커 없이 실행된다.

### `checkout` 함수 (`issues.ts:1733-1860`)

정상 체크아웃 경로도 유사하게 트랜잭션 내 `SELECT FOR UPDATE`로 감쌌다:

```typescript
// server/src/services/issues.ts:1747
// Wrapped in a transaction with SELECT FOR UPDATE to make the read + clear atomic,
```

`sameRunLock()` 헬퍼(`issues.ts:128-131`)가 동일 run ID의 재진입을 허용한다(idempotent).

---

## 6. 이종 런타임 어댑터

Paperclip의 포용성은 `packages/adapters/` 7개 패키지로 구현된다:

| 어댑터 | 대상 런타임 | 세션 재개 | 비고 |
|--------|------------|-----------|------|
| `claude-local` | Claude Code CLI | ✅ | Anthropic 공식 |
| `codex-local` | OpenAI Codex | ✅ | OpenAI |
| `cursor-local` | Cursor | ✅ | Anysphere |
| `gemini-local` | Gemini CLI | ✅ | Google |
| `opencode-local` | OpenCode | ✅ | 오픈소스 |
| `pi-local` | pi.inc | ✅ | pi |
| `openclaw-gateway` | OpenClaw 게이트웨이 | ❌ (게이트웨이가 관리) | HTTP로 위임 |

### 어댑터 플러그인 레지스트리 (`adapter-plugin.md`)

어댑터는 정적 번들 + 동적 플러그인 **두 경로**로 등록된다:

- **정적**: `server/src/adapters/index.ts`에서 `getServerAdapter(name)` 해석
- **동적**: `~/.paperclip/adapter-plugins.json` 설정에 따라 `registerServerAdapter()` / `registerUIAdapter()`로 런타임 등록. 서버 측 레지스트리 검증을 통과해야 실제 수락된다.

이 구조는 **커뮤니티가 새 Claw 런타임을 파일 한 줄 수정 없이 플러그인으로 추가할 수 있는** 공개 API다. 7개는 번들이지만 이론상 무한 확장 가능하다.

### 에이전트 인증 — Agent API Keys

에이전트가 Paperclip에 역방향으로 API를 호출할 때(예: 티켓 업데이트, 비용 보고) **Bearer API key**를 사용한다:

- 스키마: `packages/db/src/schema/agent_api_keys.ts`
- 저장: **해시 상태**로 `agent_api_keys.keyHash` 컬럼에 보관 (평문 미저장)
- 범위: company-scoped — 회사 간 격리 강제

---

## 7. 스킬 시스템 — SKILL.md 기반 런타임 주입

Paperclip도 Claw 생태계 표준인 SKILL.md 형식을 채택했다. `server/src/services/company-skills.ts` (2,368 LOC)가 담당한다.

### 스킬 디스커버리 경로

런타임에 다음 디렉토리를 워킹하며 `SKILL.md` 파일을 수집:

- `.agents/skills/`
- `.continue/skills/`
- `skills/` (저장소 번들)

### 번들 vs 커스텀 네임스페이싱 (`buildSkillRuntimeName`)

```
"paperclipai/paperclip/*" → slug                  // 번들 스킬은 slug만
otherwise                 → `${slug}--${hashSkillValue(key)}`  // 커스텀은 해시 suffix
```

`paperclipai/paperclip/*` 경로의 번들 스킬 4개(`paperclip`, `paperclip-create-agent`, `paperclip-create-plugin`, `para-memory-files`)는 짧은 slug로 등록되고, 외부 스킬은 해시가 붙어 **네임스페이스 충돌을 방지**한다.

### 스킬 주입 타이밍

스킬은 에이전트 실행 전 단계에서 어댑터로 전달된다. 어댑터별로 세부 주입 방식이 다르지만, 공통적으로 Paperclip이 스킬 메타데이터를 수집해 어댑터 `config`에 포함시킨다.

---

## 8. 예산 / 거버넌스 / 감사

런타임 샌드박스(WASM, Docker, seccomp 등)는 **없다** — Paperclip은 코드를 실행하지 않고 위임만 하기 때문. 대신 **오케스트레이션 레이어의 정책/감사**로 거버넌스를 구현한다. 이것이 **보안 Tier 3**으로 분류되는 이유다.

### 예산 정책 (`server/src/services/budgets.ts`, 958 LOC)

- **정책 기반**: `budget_policies` 테이블에 회사/프로젝트/에이전트 범위의 soft/hard 임계치 저장
- **주기 평가**: `evaluateCostEvent()`가 `cost_events`를 주기적으로 합산 → 정책과 대조
- **Eventual Consistency**: 예산 검사는 **트랜잭션 내에서 실행되지 않는다**. 관측과 강제 집행 사이에 소량의 과다 지출이 발생할 수 있다 (설계상 tradeoff)
- **자동 일시정지**: hard 임계치 초과 → `budget_incidents` 생성 + 에이전트 `pauseReason: "budget"`

```typescript
// budgets.ts:27-32
type ScopeRecord = {
  companyId: string;
  name: string;
  paused: boolean;
  pauseReason: "manual" | "budget" | "system" | null;
};
```

### 승인 게이트 (`approvals.ts` + `issue_approvals.ts`)

중요 결정(예: 목표 변경, 에이전트 고용, 예산 조정)은 `approvals` 테이블에 기록되고, 인간 보드 멤버가 `issue_approvals`를 통해 승인/거부한다. 에이전트는 승인 완료 전에는 해당 이슈를 진행할 수 없다.

### 불변 감사 로그 (`activity_log`)

`activity_log` 테이블은 append-only다. 모든 뮤테이션(에이전트 호출, 이슈 변경, 승인 결정, 비용 이벤트)이 `logActivity()`를 통해 기록된다. Hermes Agent의 SHA-256 스캐너(R22)처럼 "코드 실행 전" 검사는 아니지만, "조직 활동 사후 감사" 측면의 완전한 추적을 보장한다.

### 멀티 테넌트 격리

- `companyId` 컬럼이 거의 모든 주요 테이블에 존재 (`agents`, `issues`, `cost_events`, `company_memberships`, `company_secrets`, `activity_log`, `approvals`, `agent_api_keys`...)
- 모든 API 라우트는 `companyId` 기반 access check를 강제
- "Multi-company isolation in single deployment" — 단일 Paperclip 인스턴스에서 여러 독립 회사를 운영 가능

### 자격증명 저장 — `company_secrets`

- 회사 스코프로 secret 저장 (`company_secrets`, `company_secret_versions`)
- 평문 저장 여부는 코드 확인 필요하나, 별도 암호화 볼트 구현 없음 → IronClaw/ZeroClaw/NullClaw 수준의 Tier 1 암호화 미달
- 따라서 **Tier 3** (정책/감사 기반, 암호화/샌드박싱 없음)

---

## 9. 신규 패턴 (R-번호)

### R43: Multi-Source Heartbeat Activation

4개 비동기 이벤트 소스(`timer` / `assignment` / `on_demand` / `automation`)를 **단일 스케줄링 파이프라인으로 멀티플렉싱**하는 에이전트 기동 모델. 외부 job queue(Bull/BullMQ/Redis/SQS) 없이 자체 zero-dep cron parser(`cron.ts`, 373 LOC) + per-agent `Map<string, Promise<void>>` 직렬화 락(`heartbeat.ts:79`)으로 구현. `HEARTBEAT_MAX_CONCURRENT_RUNS_DEFAULT=1`(max 10)로 에이전트별 동시성 제한.

- **구현**: `server/src/services/heartbeat.ts:348-357`, `cron.ts:1-373`
- **원리**: 4개 이질적 기동 이벤트(스케줄/할당/수동/시스템)를 `WakeupOptions` 공통 타입 하나로 통합 → 어떤 트리거든 동일한 lifecycle(체크아웃 → 실행 → 감사 → 해제) 경로 사용 → 이벤트 소스별 분기 로직 제거
- **기존 패턴과의 차이**:
  - R9(Sleep Consolidation, always-on-memory-agent): 30분 주기 메모리 통합 **단일 목적 루프**
  - R36(MadMax 유휴-창 RL, MetaClaw): RL 훈련 **창** 스케줄링 전용
  - Paperclip: **범용 에이전트 실행 활성화 모델** — 업무 단위 기동 + cron + 이벤트 반응을 하나의 추상화로 통합

### R44: Stale-Aware Atomic Task Checkout

외부 큐(Redis/SQS/BullMQ) 없이 **Postgres 단일 DB만으로 작업자 크래시 복구**를 구현하는 패턴. `FOR UPDATE` 행락으로 동시 체크아웃 방지 + `isTerminalOrMissingHeartbeatRun()`으로 이전 체크아웃의 heartbeat run이 종료/실종 상태인지 검사 + `WHERE checkoutRunId = expectedCheckoutRunId` CAS 가드로 원자적 입양(adopt).

- **구현**: `server/src/services/issues.ts:864-909` (`adoptStaleCheckoutRun`, `isTerminalOrMissingHeartbeatRun`), `issues.ts:822-828` (정렬된 `FOR UPDATE`)
- **원리**: 일반적인 "stalled job" 복구는 BullMQ 같은 외부 브로커의 책임이지만, Paperclip은 이슈 상태와 worker run 생명주기를 DB 레벨에서 결합해 브로커 없이 동등한 안전성을 확보. 정렬된 행락으로 데드락 회피.
- **기존 패턴과의 차이**:
  - Autoresearch(R3): 크래시 발생 시 `git reset`으로 로컬 상태 롤백 — **단일 워커 가정**
  - ZeroClaw Soul Snapshot: 세션 재개 지향 — 동시성 없음
  - OpenClaw-RL(R7): async queue drain — 외부 자체 큐 사용
  - Paperclip R44: **Postgres 네이티브 멀티-워커 크래시 복구** (Claw 생태계 최초)

### R45: Heterogeneous-Runtime Org Chart (이종 런타임 조직도)

서로 다른 Claw 런타임(Claude Code + OpenClaw + Codex + Cursor + Gemini + OpenCode + pi)의 에이전트들을 **하나의 영속 조직 구조**에 직원으로 묶는 패턴. 각 직원에게 hierarchy(reporting lines), 월간 예산, 담당 프로젝트, 이슈 큐, SKILL.md를 부여하고 한 대시보드에서 관리.

- **구현**: `packages/adapters/{claude,codex,cursor,gemini,openclaw-gateway,opencode,pi}-local/`, `server/src/services/issues.ts`, `server/src/routes/org-chart-svg.ts`
- **원리**: 에이전트를 **어떤 런타임으로 구동되는지와 무관하게** 동일한 인터페이스(heartbeat 수신, 이슈 체크아웃, 비용 보고, 감사 기록)로 추상화. 어댑터가 런타임별 세부사항을 흡수 → 상위 조직 모델은 런타임 다양성에 둔감.
- **기존 패턴과의 차이**:
  - R21(Bounded Delegation Tree, Hermes Agent): **세션 범위**의 하위 에이전트 트리, 휘발성, 동일 런타임 가정
  - R35(Swap-Runtime Proxy, MetaClaw): **1 에이전트 × N 백엔드**를 스왑 가능하게 추상화 — 동시 실행되는 것은 1개
  - Paperclip R45: **N 에이전트 × N 런타임 × 영속 조직** — 여러 런타임이 동시에 다른 역할로 영속 활동

---

## 10. 비교 테이블

| 축 | Paperclip | MetaClaw | Symphony | ClawWork |
|----|-----------|----------|----------|----------|
| **카테고리** | 조직 오케스트레이션 | 에이전트 메타 프록시 | 운영 자동화 | 벤치마크 |
| **스케줄 모델** | Heartbeat (timer+event+manual+auto) | SlowUpdateScheduler(유휴창 RL) | CI/PR 이벤트 | 수동 실행 |
| **런타임 추상화** | **N 에이전트 × N 런타임** (7 어댑터) | **1 에이전트 × N 백엔드** (swap) | 없음 (고정) | 없음 (Nanobot 확장) |
| **자기 개선** | ❌ (사용자 거버넌스) | ✅ (RL + 스킬 진화) | ❌ | ❌ |
| **예산 강제** | ✅ Eventual (정책 기반 auto-pause) | ✅ 훈련 비용 한도 | ❌ | ✅ 경제 시뮬 |
| **감사 로그** | ✅ 불변 `activity_log` | ✅ 세션 로그 | ✅ Elixir/OTP 로그 | ✅ 지출 이력 |
| **승인 게이트** | ✅ `issue_approvals` | ❌ | ✅ PR 리뷰 | ❌ |
| **멀티 테넌트** | ✅ `companyId` 격리 | ❌ (단일 사용자 지향) | ❌ | ❌ |
| **LLM 직접 호출** | ❌ (BYOA만) | ✅ (OpenAI-compatible 프록시) | ❌ | ✅ |
| **보안 Tier** | **Tier 3** (정책/감사) | Tier 3 | Tier 3 | Tier 3 |
| **신규 R 번호** | R43, R44, R45 | R35, R36, R37 | — | — |
| **Stars (2026-04-14)** | **53,248** | 2,700+ | — | — |
| **LOC** | ~251,900 TS | ~64,000 Py | Elixir | Python |

Paperclip은 응용 계층 중 **유일하게 멀티 테넌트 + 승인 게이트 + N×N 런타임 추상화를 모두 갖춘** 프로젝트다. 반면 자기 개선(RL)과 LLM 직접 호출은 범위 밖으로 둔다 — "회사 운영"과 "모델 개선"을 분리하는 관심사 분리 설계.

---

## 11. 한계

1. **런타임 샌드박스 부재**: Paperclip 자체는 어댑터를 통해 외부 프로세스/CLI를 호출할 뿐, WASM/Docker/seccomp 수준의 격리를 제공하지 않는다. 런타임 보안은 기저 어댑터(예: Claude Code의 seccomp+bwrap, R26)에 위임된다. 어댑터가 약하면 전체가 약하다.

2. **예산 Eventual Consistency의 실제 영향**: 예산 검사가 트랜잭션에 포함되지 않으므로, 검사 시점과 `pause` 시점 사이에 추가 LLM 호출/지출이 발생 가능. 특히 병렬 heartbeat가 예산 한도에 근접할 때 초과 가능성 존재. Paperclip은 이를 trade-off로 명시하지 않음.

3. **LLM 직접 호출 없음 (BYOA 강제)**: 사용자가 어댑터가 지원하는 런타임(7종)을 별도 설치/구성해야 함. Claude Code 라이선스, Codex 서브스크립션 등 외부 조건이 Paperclip 사용성을 좌우.

4. **Embedded Postgres 단일 노드 제약**: 개발·중소 규모 배포는 PGlite 또는 단일 Postgres 인스턴스로 충분하지만, 수천 개 회사 × 수만 에이전트 규모에서는 분산 DB 마이그레이션이 불가피. 공식 가이드 미제공.

5. **자격증명 암호화 미구현**: `company_secrets`는 DB 레벨에만 존재하며, IronClaw의 ChaCha20-Poly1305 또는 ZeroClaw의 WASM 프록시 injection 수준의 격리 없음. Postgres 인스턴스 접근이 전체 자격증명 유출로 이어짐.

6. **Clipmart 마켓플레이스 미완성**: "회사 템플릿" 공유 마켓플레이스가 이 플랫폼의 킬러 기능이지만 "COMING SOON"으로 로드맵에만 존재. 실제 공급 시점/품질 검증 부재 시 가치 제안이 약해짐.

---

## 12. 참고 링크

- **GitHub**: https://github.com/paperclipai/paperclip
- **Homepage**: https://paperclip.ing
- **Docs**: https://paperclip.ing/docs
- **Discord**: https://discord.gg/m4HZY7xNG3
- **License**: MIT (`repos_applied/paperclip/LICENSE`)
- **내부 문서**:
  - `repos_applied/paperclip/README.md`
  - `repos_applied/paperclip/AGENTS.md` — 인간·AI 컨트리뷰터 가이드
  - `repos_applied/paperclip/adapter-plugin.md` — 어댑터 플러그인 구조
  - `repos_applied/paperclip/doc/SPEC-implementation.md` — V1 빌드 스펙 (참조 예정)
  - `repos_applied/paperclip/doc/PRODUCT.md`, `doc/GOAL.md`, `doc/DATABASE.md`
- **관련 보고서**:
  - `reports/repos_applied/details/metaclaw_report.md` — 가장 유사한 메타 레이어 (R35 비교)
  - `reports/repos_applied/details/symphony_report.md` — 운영 자동화 대비
  - `reports/repos/session_context_report.md` — Heartbeat 세션 재개 모델
  - `reports/repos/security_report.md` — Tier 3 분류 기준
  - `reports/repos_applied/repos_applied_report.md` — §3.7 Paperclip 서브섹션

---

**최종 수정**: 2026-04-14
