# Claw 생태계 응용 계층 비교 분석 — ClawWork, ClawPort

> **조사 일자**: 2026-03-07
> **조사 방법**: 2개 scientist 에이전트가 각 레포의 실제 소스코드를 병렬 심층 분석
> **핵심 질문**: "Claw 프레임워크 위에 구축된 응용 프로젝트들은 어떤 계층을 추가하고, 어떤 문제를 해결하며, 어떤 패턴을 공유하는가?"
> **선행 보고서**: session_context_report.md, security_report.md, browser_actions_report.md, memory_architecture_report.md

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. [repos_applied란 무엇인가](#2-repos_applied란-무엇인가)
3. [개별 분석](#3-개별-분석)
   - 3.1 ClawWork
   - 3.2 ClawPort (clawport-ui)
   - 3.3 Symphony
   - 3.4 Moltbook API
4. [교차 분석](#4-교차-분석)
5. [선행 보고서와의 연결](#5-선행-보고서와의-연결)
6. [결론 및 열린 질문](#6-결론-및-열린-질문)

---

## 1. Executive Summary

2개 응용 프로젝트의 소스코드를 분석한 결과, Claw 생태계의 **응용 계층**은 프레임워크 코드가 해결하지 않은 2가지 새로운 문제를 다룬다:

| 문제 | 해결 프로젝트 | 접근법 |
|------|-------------|--------|
| **에이전트 성능 측정** | ClawWork | 경제적 생존 게임 (GPT 평가 + BLS 임금 기반 지급) |
| **에이전트 팀 관찰/제어** | ClawPort | OpenClaw 게이트웨이 프록시 대시보드 (Org Map + 채팅 + 비용) |
| **에이전트 워크플로 자동화** | Symphony | Elixir/OTP 기반 이슈 트래커 → 에이전트 디스패치 → PR 랜딩 데몬 |
| **에이전트 소셜 플랫폼** | Moltbook API | AI 에이전트를 위한 소셜 네트워크 (Reddit/X 스타일, 어떤 Claw도 미의존) |

**가장 주목할 발견 3가지:**

1. **프레임워크 비침습적 확장 패턴이 정립됐다.** ClawWork는 Nanobot 소스 수정 없이 7가지 기법(서브클래싱, 도구 추가, 메서드 오버라이드, 런타임 클래스 교체, 투명 래퍼, 스킬 주입, 설정 분리)으로 경제 추적 레이어를 삽입한다 (`clawmode_integration/provider_wrapper.py:37`, `agent_loop.py:46`).

2. **"Zero Own Key" 아키텍처가 새로운 패턴으로 등장했다.** ClawPort는 자체 Anthropic API 키를 전혀 보유하지 않는다. 9개 API 라우트 전체가 OpenClaw 게이트웨이(localhost:18789)에 위임하며, 이를 통해 사용자의 기존 인증과 대화 컨텍스트를 그대로 활용한다 (`app/api/chat/[id]/route.ts:9-12`).

3. **SKILL.md가 응용 계층에서도 핵심 인터페이스다.** ClawWork는 경제 프로토콜을 `clawmode_integration/skill/SKILL.md`로 주입하여 에이전트에 상시 로딩한다 (`skill/SKILL.md:4`). browser_actions_report.md가 분석한 NanoClaw의 SKILL.md 패턴이 응용 계층으로도 확산됐다.

---

## 2. repos_applied란 무엇인가

`repos/`가 Claw 런타임 프레임워크들(OpenClaw, Nanobot, NanoClaw, IronClaw, ZeroClaw, PicoClaw, TinyClaw)을 담고 있다면, `repos_applied/`는 그 프레임워크 **위에** 구축된 응용 프로젝트들이다.

```
┌──────────────────────────────────────────┐
│          응용 계층 (repos_applied/)        │
│  ClawWork                ClawPort         │
│  (벤치마크)               (대시보드)        │
└──────────────────┬───────────────────────┘
                   │ 사용/확장
┌──────────────────▼───────────────────────┐
│          프레임워크 계층 (repos/)           │
│  Nanobot    OpenClaw    NanoClaw ...       │
└──────────────────────────────────────────┘
```

| 프로젝트 | 기반 프레임워크 | 관계 유형 | 추가하는 계층 |
|----------|--------------|-----------|-------------|
| **ClawWork** | Nanobot | 확장 (서브클래싱 + 래핑) | 경제 추적 + 작업 평가 + 벤치마크 |
| **ClawPort** | OpenClaw | 프록시 (게이트웨이 위임) | UI + 관찰 + 제어 인터페이스 |
| **Symphony** | 없음 (Elixir/OTP) | 운영 자동화 계층 | 이슈 트래커 → 에이전트 디스패치 → PR 랜딩 |
| **Moltbook API** | 없음 (Node.js/Express) | 플랫폼 목적지 | AI 에이전트 소셜 네트워크 (Reddit/X 스타일) |

---

## 3. 개별 분석

### 3.1 ClawWork — AI 경제 벤치마크

**핵심 철학**: "에이전트가 실제로 돈을 벌 수 있는가?" — 기술 지표 대신 경제적 생존을 측정

#### 3.1.1 이중 실행 경로

ClawWork는 두 개의 독립적인 경로를 제공한다:

```
독립 시뮬레이션 (livebench/)
  GDPVal parquet → TaskManager → LiveAgent (LangChain + MCP) → WorkEvaluator → EconomicTracker
                                                               ↑
ClawMode 통합 (clawmode_integration/)               Nanobot AgentLoop 서브클래스
  Telegram/Discord → Nanobot → ClawWorkAgentLoop → /clawwork 명령 → 동일 EconomicTracker
```

두 경로가 `EconomicTracker`, `WorkEvaluator`, `TaskManager`를 공유한다. `LiveAgent`는 `langchain_openai.ChatOpenAI`와 MCP를 사용하며, ClawMode는 기존 Nanobot `AgentLoop`를 서브클래싱하여 메신저 게이트웨이에 통합된다 (`livebench/agent/live_agent.py:38`, `clawmode_integration/agent_loop.py:46`).

#### 3.1.2 Nanobot 비침습적 확장 — 7가지 기법

Nanobot 소스 파일을 단 한 줄도 수정하지 않고 경제 추적 레이어를 삽입하는 방법:

| 기법 | 구체적 내용 | 파일:라인 |
|------|-----------|-----------|
| **서브클래싱** | `ClawWorkAgentLoop(AgentLoop)` | `agent_loop.py:46` |
| **도구 추가** | `_register_default_tools()` 오버라이드, `super()` 호출 유지 | `agent_loop.py:76` |
| **메서드 오버라이드** | `_process_message()`로 start/end_task 자동화 | `agent_loop.py:91` |
| **런타임 클래스 교체** | `self.provider.__class__ = CostCapturingLiteLLMProvider` | `agent_loop.py:63` |
| **투명 래퍼** | `TrackedProvider`로 `chat()` 가로채기 | `provider_wrapper.py:37` |
| **스킬 주입** | `SKILL.md` + `always: true`로 경제 프로토콜 상시 로딩 | `skill/SKILL.md:4` |
| **설정 분리** | `~/.nanobot/config.json`의 `agents.clawwork` 섹션 | `config.py:59` |

`CostCapturingLiteLLMProvider`는 `_parse_response()`를 오버라이드하여 OpenRouter의 `response.usage.cost`와 `response._hidden_params["response_cost"]`를 포착한다 (`provider_wrapper.py:18-34`). TrackedProvider는 `__getattr__`로 나머지 모든 호출을 원본 프로바이더에 위임한다 (`provider_wrapper.py:71`).

#### 3.1.3 경제 엔진의 세부 설계

**비용 추적 우선순위** (`economic_tracker.py:158-173`):

```
OpenRouter 직접 보고 비용 > litellm 계산 > 로컬 공식 (input_price × tokens)
```

실제 API 응답 비용을 우선 사용해 추정 오차를 최소화한다.

**지급 이중 구조** — 평가 점수와 지급액의 비선형 관계:

```
evaluation_score (GPT, 0-10) → 정규화 (÷10) → 0.0-1.0
  score >= 0.6:  payment = score × max_payment  (선형 비례)
  score < 0.6:   payment = $0.00               (하드 클리프)
```

0.59와 0.60 사이의 차이가 max_payment 전액이 된다 (`economic_tracker.py:380-395`, `livebench/llm_evaluator.py:166`). 최소 임계치 0.6은 인스턴스 생성 시 파라미터로 설정 가능하다.

**$10 시작 잔액의 설계 근거**: Tavily 검색 1회 $0.0008, LLM 수십 회 호출이 잔액의 수%를 소진한다. 품질 미달 2-3회 반복 시 실제로 파산 위기가 발생하도록 설계됐다. README에 기재된 실제 결과($19K, $15K 등)는 초기 $10 잔액 대비 1000-2000배 수익으로 `initial_balance` 설정값이 인위적임을 시사한다 (`livebench/configs/` 참조).

**생존 상태 분류** (`economic_tracker.py:524-538`):

| 잔액 | 상태 |
|------|------|
| ≤ $0 | bankrupt |
| $0 ~ $100 | struggling |
| $100 ~ $500 | stable |
| > $500 | thriving |

#### 3.1.4 평가 시스템

44개 직업 카테고리별 GPT 루브릭 (`eval/meta_prompts/{Occupation}.json`). Fallback이 완전히 제거됐다 — 루브릭 파일이 없으면 `FileNotFoundError`, LLM 평가 실패 시 `raise ValueError`로 명시적 차단 (`evaluator.py:43`). 평가 전용 API 키(`EVALUATION_API_KEY`) 분리 지원.

**지급 공식** (GDPVal 기반):
```
Payment = quality_score × (estimated_hours × BLS_hourly_wage)
범위: $82.78 ~ $5,004.00, 평균: $259.45
```

#### 3.1.5 미완성 부분

| 항목 | 상태 | 코드 근거 |
|------|------|-----------|
| Trading 시스템 | 코드 존재, 완전 비활성화 | `live_agent.py:189` |
| 포트폴리오 가치 계산 | TODO | `economic_tracker.py:496` |
| 이미지/PDF 아티팩트 분석 | 메타데이터만 전달 | `llm_evaluator.py:264` |
| `get_cost_analytics()` | `record["type"]` KeyError 잠재 버그 | `economic_tracker.py:641` |

---

### 3.2 ClawPort — OpenClaw 에이전트 대시보드

**핵심 철학**: "에이전트 팀을 사람이 볼 수 있게" — 자체 AI 키 없이 OpenClaw를 완전히 프록시

#### 3.2.1 "Zero Own Key" 아키텍처

ClawPort는 자체 Anthropic API 키를 전혀 보유하지 않는다. 모든 AI 호출이 OpenClaw 게이트웨이(localhost:18789)를 경유한다:

```
Browser  →  Next.js API Routes
               ├── 텍스트: openai.chat.completions(baseURL="localhost:18789/v1") → 스트리밍 SSE
               ├── 비전: execFile(openclaw CLI) → chat.send → 폴링 chat.history → SSE
               ├── 음성(STT): Whisper via localhost:18789/v1/audio/transcriptions
               ├── 음성(TTS): openclaw TTS → SSE 청크
               └── 로그 스트림: spawn(openclaw logs --follow --json) → SSE
```

9개 API 라우트 전체에서 직접 Anthropic API 호출 0건 (`app/api/chat/[id]/route.ts:9-12`).

#### 3.2.2 에이전트 자동 발견 — 4단계 폴백 체인

`lib/agents-registry.ts:505`의 `loadRegistry()` 우선순위:

```
1. User Override     $WORKSPACE_PATH/clawport/agents.json
       ↓ 없으면
2. Auto-Discovery    IDENTITY.md → root SOUL.md → agents/*/SOUL.md
                     → sub-agents/*.md / members/*.md
       ↓ 없으면
3. CLI-Only          openclaw agents list --json 결과만 사용
       ↓ 없으면
4. Bundled Fallback  lib/agents.json (빌드/테스트용)
```

어떤 OpenClaw 워크스페이스도 별도 설정 없이 즉시 작동한다. `parseSoulHeading()`이 5가지 SOUL.md 헤딩 포맷을 처리하고, 15가지 색상 팔레트에서 에이전트 색상을 자동 배정한다 (`agents-registry.ts:11-55`).

#### 3.2.3 채팅 파이프라인 — 텍스트 vs 비전 분기

**비전 파이프라인이 CLI 기반인 이유** (3가지 기술적 제약):

1. 게이트웨이 HTTP 엔드포인트가 `image_url` 컨텐츠 파트를 제거함
2. WebSocket `operator.write` 스코프는 device keypair 서명 필요 — CLI만 보유
3. macOS ARG_MAX(1MB) 제약 → 1200px JPEG(0.85 품질) 리사이징 필수

```
Client Canvas API 리사이징 (최대 1200px)
  → 최신 user 메시지만 이미지 감지 (route.ts:60-61)
  → execFile("openclaw", ["gateway", "call", "chat.send", ...], timeout:15s)
  → 2초 폴링 chat.history (최대 60초, timestamp >= sendTs 매칭)
  → 단일 SSE 청크 반환
```

`lib/anthropic.ts:123,142,161`

#### 3.2.4 메모리 브라우저

`lib/memory.ts`의 `getMemoryConfig()`가 OpenClaw의 `openclaw.json`을 직접 파싱해 하이브리드 검색 설정을 읽는다:

```json
{ "vectorWeight": 0.7, "textWeight": 0.3, "halfLifeDays": 30, "mmrLambda": 0.7,
  "softThresholdTokens": 80000 }
```

memory_architecture_report.md에서 분석한 OpenClaw Tier 1 메모리 아키텍처 설정을 UI에서 그대로 시각화한다. 단, 읽기 전용 — MEMORY.md 수정 불가 (`lib/memory.ts:131`).

#### 3.2.5 비용 대시보드

순수 함수 파이프라인으로 구성:

```
toRunCosts() → computeJobCosts() + computeDailyCosts() + computeModelBreakdown()
             + detectAnomalies() + computeWeekOverWeek() + computeCacheSavings()
```

이상 감지 조건: 동일 job 3회 이상 실행 AND 중앙값 토큰의 5배 초과 (`lib/costs.ts:143-148`). 내장 모델 가격표: Claude Opus/Sonnet/Haiku 7개 항목, 미지원 모델은 Sonnet 가격으로 폴백.

#### 3.2.6 UI 특이점

- **Org Map**: React Flow + Dagre, Hierarchy/Teams 두 모드. Teams 모드에서 팀별 독립 서브그래프 배치 (`components/OrgMap.tsx:27-28`)
- **마크다운 렌더링**: 외부 라이브러리 없이 regex 기반 구현 (테이블, 이미지 미지원)
- **대화 저장**: localStorage + base64 data URL (blob URL은 새로고침 시 소멸하므로 불사용)
- **슬래시 명령어**: 6개 (`/clear /help /info /soul /tools /crons`), 완전 클라이언트 사이드, API 전송 필터링
- **5개 테마**: CSS custom properties (`--bg`, `--text-primary`, `--accent` 등 33개 시맨틱 토큰)

#### 3.2.7 테스트

536개 테스트, 24개 스위트, 모두 `lib/` 디렉토리에 소스 파일과 동일 위치. 핵심 패턴:

```typescript
vi.mock('child_process')                           // CLI subprocess 격리
vi.useFakeTimers({ shouldAdvanceTime: true })      // 폴링 루프 시뮬레이션 필수
vi.stubEnv('WORKSPACE_PATH', '/mock')              // 환경변수 격리
```

`shouldAdvanceTime: true`는 비전 파이프라인의 2초 폴링 루프 테스트를 가능하게 하는 핵심 옵션이다 (`CLAUDE.md:292-304`).

---

### 3.3 Symphony — 운영 자동화 계층

> **상세 분석**: `reports/repos_applied/details/symphony_report.md`

**핵심 철학**: "이슈 트래커를 폴링하여 코딩 에이전트를 자동 디스패치하고, PR 랜딩까지 감시하는 운영 자동화 레이어" — Elixir/OTP로 구현됐으며, 소스코드(`elixir/`)와 언어 독립적 사양(`SPEC.md`)을 분리한다.

#### 3.3.1 포지셔닝

```
┌─────────────────────────────────────────────────────────────────┐
│            운영 자동화 계층 (새 범주)                              │
│  Symphony — 이슈 트래커 → 에이전트 디스패치 → PR 랜딩 데몬         │
└───────────────────────┬─────────────────────────────────────────┘
                        │ 조율/실행
┌───────────────────────▼─────────────────────────────────────────┐
│                응용 계층 (repos_applied/)                         │
│  ClawWork (벤치마크)  ClawPort (대시보드)                           │
└───────────────────────┬─────────────────────────────────────────┘
                        │ 사용/확장
┌───────────────────────▼─────────────────────────────────────────┐
│              프레임워크 계층 (repos/)                              │
│  Nanobot  OpenClaw  NanoClaw  IronClaw  ZeroClaw  PicoClaw …    │
└─────────────────────────────────────────────────────────────────┘
```

Symphony는 `SPEC.md:29`에서 자신을 "a scheduler/runner and tracker reader"로 정의한다. Claw 프레임워크들이 없던 **3번째 계층**이다.

#### 3.3.2 핵심 아키텍처

**Tech Stack**: Elixir `~> 1.19` (OTP 28), GenServer 기반 5개 프로세스, Bandit HTTP, Phoenix LiveView, Linear GraphQL.

**핵심 모듈**:

| 모듈 | 책임 |
|------|------|
| `WorkflowStore` | WORKFLOW.md 핫리로드 (1초, mtime+phash2 변경 감지) |
| `Orchestrator` | 5초 폴링, 에이전트 태스크 디스패치, 상태 기계 |
| `AgentRunner` | 이슈별 워크스페이스 생성 + AppServer 세션 + 턴 루프 |
| `Codex.AppServer` | JSON-RPC 2.0 over stdio Port (Codex 서브프로세스 제어) |
| `Linear.Adapter` | GraphQL CRUD: 이슈 페치, 상태 전환, PR 연결 |

#### 3.3.3 WORKFLOW.md — 핫리로드 에이전트 정책

`WORKFLOW.md`는 YAML 프론트매터(런타임 설정: 폴링 간격, 동시성, 훅)와 Liquid 템플릿 본문(에이전트 프롬프트)이 하나의 파일에 공존하는 구조다. `WorkflowStore`가 1초마다 변경을 감지하여 재시작 없이 핫리로드한다 (`workflow_store.ex:141-148`). SKILL.md 철학의 상위 확장이다.

#### 3.3.4 SKILL.md 공식화

Symphony는 `.codex/skills/` 아래 6개 스킬을 제공한다 (`commit`, `debug`, `land`, `linear`, `pull`, `push`). 각 스킬은 YAML 프론트매터(`name`, `description`)와 마크다운 절차 본문으로 구성된다. `land` 스킬은 비동기 Python 감시 헬퍼(`land_watch.py`)까지 포함한 실행 플레이북이다.

**SKILL.md 진화 경로**: NanoClaw(원조, 순수 마크다운) → ClawWork(주입 헤더) → **Symphony(YAML 프론트매터 + 교차 참조)**. OpenAI가 독립적으로 동일한 컨벤션을 채택했다.

#### 3.3.5 5단계 Proof-of-Work 체인

| Layer | 내용 | 강제 방식 |
|-------|------|---------|
| 1. 로컬 게이트 | `make -C elixir all` | `push/SKILL.md:29` 지침 |
| 2. CI | `make all` (format+compile+dialyzer+tests+specs.check) | GitHub Actions 모든 PR |
| 3. PR 본문 린트 | `mix pr_body.check` — 헤딩/플레이스홀더/빈 섹션 검사 | `.github/workflows/pr-description-lint.yml` |
| 4. Codex AI 리뷰 | `## Codex Review — <persona>` 댓글 승인 확인 | `land_watch.py:330-347` |
| 5. Async 감시자 | CI 폴링 + 리뷰 감지 + HEAD 변경 감지 (3개 병렬 태스크) | `land_watch.py` 출구 코드 2/3/4/5 |

---

### 3.4 Moltbook API — AI 에이전트 소셜 네트워크

> **상세 분석**: `reports/repos_applied/details/moltbook_report.md`

**핵심 철학**: "AI 에이전트를 위한 소셜 네트워크" — 인간이 Reddit/X를 쓰듯, AI 에이전트가 서로 소통하고 커뮤니티를 형성하며 평판(karma)을 쌓는 플랫폼. `package.json`의 설명: "The social network for AI agents".

#### 3.4.1 핵심 전제

1. **에이전트가 1등급 시민이다.** 사람(human) 계정이 존재하지 않는다. 모든 계정은 `agent`다.
2. **에이전트는 소유자(human)에 의해 검증된다.** Twitter/X OAuth를 통해 실제 사람이 에이전트를 "claim"해야 핵심 기능을 사용할 수 있다.
3. **API 키가 유일한 인증 수단이다.** `moltbook_<64hex>` 불투명 토큰. DB에는 SHA-256 해시만 저장된다 (`auth.js:97`).

#### 3.4.2 기술 스택

| 항목 | 선택 |
|------|------|
| 언어 | JavaScript (CommonJS) |
| 프레임워크 | Node.js ≥18, Express 4.18 |
| 데이터베이스 | PostgreSQL (pg, 연결풀 max=20) |
| 인증 | `moltbook_<64hex>` Bearer 토큰, SHA-256 해시 저장 |
| 캐시/레이트리밋 | 인메모리 슬라이딩 윈도우 (Redis 선택적) |
| 파일 수 | 23개 소스 파일 |

#### 3.4.3 아키텍처

7개 PostgreSQL 테이블: `agents`, `submolts`, `submolt_moderators`, `posts`, `comments`, `votes`, `subscriptions`, `follows`.

서비스 계층: `AgentService`, `PostService`, `CommentService`, `VoteService`, `SubmoltService`, `SearchService`.

#### 3.4.4 Claw 프레임워크 의존성: 없음

Moltbook은 어떤 Claw 프레임워크도 직접 의존하지 않는다. 대신 Claw 에이전트들이 HTTP REST API를 호출하여 참여하는 **목적지(destination)**다.

```
┌───────────────────────────────────────────┐
│         Claw 프레임워크 에이전트              │
│  OpenClaw / Nanobot / NanoClaw / ...       │
│   await fetch('moltbook.com/api/v1/...')   │
└───────────────────┬───────────────────────┘
                    │  REST API
┌───────────────────▼───────────────────────┐
│           Moltbook API                     │
│     Node.js + Express + PostgreSQL         │
└───────────────────────────────────────────┘
```

#### 3.4.5 주요 특이점

**skill.md 소비 인터페이스**: `src/app.js:53`에서 `documentation: 'https://www.moltbook.com/skill.md'`를 명시한다. Claw 에이전트가 SKILL.md 형식의 문서를 읽고 도구로 활용하는 패턴을 Moltbook도 채택했다.

**에이전트 Claim 시스템**: 에이전트 자가 등록 후, 소유자(사람)가 Twitter/X verification_code 트윗으로 신원을 증명해야 핵심 기능이 활성화된다. "에이전트가 자율적으로 행동하되, 인간이 책임을 진다" 원칙의 구현.

**Reddit 피드 알고리즘**: hot(로그 스케일 + 시간감쇠), rising(윌슨 스코어 변형)을 SQL로 직접 구현. 45000초 감쇠 상수는 Reddit과 동일.

**투표 상태 기계**: 업보트/다운보트 toggle + change 5가지 상태를 `VoteService`가 처리. karma 자동 연동.

**미완성**: Twitter/X OAuth claim 검증, Redis 레이트리밋, ESLint, DB 마이그레이션 스크립트 미구현.

---

## 4. 교차 분석

### 4.1 Claw 프레임워크 의존도

| 프로젝트 | 의존 프레임워크 | 결합 방식 | 교체 용이성 |
|----------|--------------|-----------|------------|
| **ClawWork** | Nanobot | 서브클래싱 + 래핑 (비침습적) | 중간 (AgentLoop API 의존) |
| **ClawPort** | OpenClaw | 게이트웨이 프록시 (URL + 토큰) | 높음 (URL 변경으로 교체 가능) |
| **Symphony** | 없음 | Codex app-server JSON-RPC over stdio | 높음 (SPEC.md 준수 에이전트면 교체 가능) |
| **Moltbook API** | 없음 | REST API 소비 (에이전트가 HTTP 클라이언트) | 해당 없음 (목적지, 프레임워크 중립) |

ClawPort는 OpenClaw API에만 의존하므로 이론적으로 호환 게이트웨이로 교체 가능하다. Symphony는 `SPEC.md` 준수 에이전트(`app-server` JSON-RPC over stdio)면 Codex 이외의 에이전트도 교체 가능하다. Moltbook은 어떤 프레임워크도 의존하지 않는 완전 독립 플랫폼이다.

### 4.2 SKILL.md의 생태계 확산

browser_actions_report.md에서 NanoClaw의 SKILL.md 패턴을 "SKILL.md + 컨테이너 위임"으로 분류했다. 응용 계층에서도 이 패턴이 확산됐으며, 3단계 진화가 확인된다:

| 사용처 | SKILL.md 역할 | 형식 |
|--------|-------------|------|
| NanoClaw (`repos/`) | 에이전트 행동 절차 정의 (컨테이너 IPC 통신) | 순수 마크다운 |
| ClawWork (`repos_applied/`) | 경제 프로토콜 주입 (`always: true`로 상시 로딩) | 주입 헤더 |
| Symphony (`repos_applied/`) | `.codex/skills/` 6개 실행 플레이북 | YAML 프론트매터 + 교차 참조 |
| Moltbook (`repos_applied/`) | `skill.md` 소비 인터페이스 (에이전트용 API 문서) | 소비 목적 |

**OpenAI가 독립적으로 동일한 컨벤션을 채택**했다는 사실이 SKILL.md 패턴의 수렴 압력을 입증한다.

### 4.3 비용 추적의 2가지 접근

4개 프로젝트의 비용 추적 전략은 각각 다르다:

| 프로젝트 | 추적 단위 | 방법 | 세밀도 |
|----------|---------|------|--------|
| **ClawWork** | 작업(task) 단위 | 토큰 가로채기 → 잔액 실시간 차감 | 채널별(LLM/검색/OCR) |
| **ClawPort** | cron run 단위 | OpenClaw 실행 기록 파싱 | 모델별, 이상 감지 |
| **Symphony** | 없음 (운영 비용 추적 없음) | — | — |
| **Moltbook** | 없음 (API 호출 비용 추적 없음) | — | — |

ClawWork는 비용을 에이전트 생존에 직접 연결(비용 = 잔액 차감)하는 반면, ClawPort는 비용을 관찰 대상으로 시각화한다.

### 4.4 미완성 패턴의 공통성

4개 프로젝트 모두 "설계됐으나 구현되지 않은" 부분을 가진다:

| 프로젝트 | 미완성 항목 | 코드 근거 |
|----------|-----------|-----------|
| ClawWork | Trading 시스템 | `livebench/trading/` 존재, `live_agent.py:189` 비활성화 |
| ClawWork | 포트폴리오 가치 | `economic_tracker.py:496` `net_worth = balance` TODO |
| ClawPort | 메모리 수정 | Memory UI 읽기 전용 |
| ClawPort | 멀티 워크스페이스 계층 | `agents-registry.ts:462-464` 부분 지원 |
| Symphony | in-memory 상태 영속화 | `SPEC.md:47` 재시작 복구 요구, 실행 중 에이전트 소실 가능 |
| Moltbook | Twitter/X OAuth claim | `AgentService.claim()` 메서드 존재, OAuth 라우트 없음 |

### 4.5 테스트 성숙도 격차

| 프로젝트 | 테스트 | 접근 |
|----------|--------|------|
| **ClawPort** | 536개 / 24 스위트 | 단위 테스트 + vi.mock 체계적 사용 |
| **Symphony** | Elixir ExUnit | dialyxir 정적 분석 + `make all` CI 강제 |
| **ClawWork** | 없음 | 수동 실행 스크립트만 |
| **Moltbook** | 없음 | ESLint도 미설치 (devDependencies 비어있음) |

ClawPort와 Symphony는 각각 다른 방식으로 품질을 강제한다. Symphony는 `specs_check.ex`로 모든 공개 함수의 `@spec` 선언을 CI에서 강제하며, 이는 4개 프로젝트 중 가장 엄격한 정적 분석이다.

---

## 5. 선행 보고서와의 연결

### 5.1 session_context_report.md와의 연결

session_context_report.md는 "프레임워크 레벨에서 자동으로 컨텍스트를 분리하는 구현체가 없다"는 결론을 내렸다. ClawWork의 독립 시뮬레이션 모드가 하나의 답을 보여준다 — **작업(task) 단위로 에이전트 실행을 완전히 분리**한다. 각 GDPVal 작업은 독립적인 LangChain 대화 세션으로 실행되며, 세션 간 컨텍스트 유출이 없다. 이는 "프로젝트 수명주기 관리 프레임워크"의 단순한 형태다.

### 5.2 memory_architecture_report.md와의 연결

ClawPort의 메모리 브라우저가 OpenClaw의 Tier 1 메모리 설정(`vectorWeight`, `halfLifeDays`, `mmrLambda`)을 직접 시각화한다. 기존 보고서가 분석한 "이중 주입 경로"와 "하이브리드 검색"이 실제로 UI에서 어떻게 보이는지를 ClawPort가 구체화했다.

### 5.3 browser_actions_report.md와의 연결

browser_actions_report.md가 "SKILL.md + 컨테이너 위임"을 NanoClaw 고유 패턴으로 분류했다. ClawWork가 SKILL.md를 경제 프로토콜 주입에 활용함으로써 이 패턴이 특정 프레임워크를 넘어 응용 계층의 인터페이스 컨벤션으로 확산됐음을 확인했다.

### 5.4 security_report.md와의 연결

security_report.md는 "자격증명 암호화를 구현한 곳은 IronClaw와 ZeroClaw뿐"이라는 결론을 내렸다. 응용 계층에서도 이 패턴이 반복된다:

- ClawPort: `OPENCLAW_GATEWAY_TOKEN` 환경변수, .env 파일 의존 — Tier 3 수준
- ClawWork: `~/.nanobot/config.json` 평문 저장 (Nanobot 상속) — Tier 3 수준

---

## 6. 결론 및 열린 질문

### 핵심 결론

1. **Claw 생태계의 응용 계층이 2가지 독립적 방향으로 분화됐다.** 벤치마크(ClawWork), 관찰/제어(ClawPort). 이 2가지가 상호 보완적이다 — ClawWork로 성능을 측정하고, ClawPort로 팀을 운영한다.

2. **비침습적 프레임워크 확장 패턴이 실용적으로 검증됐다.** ClawWork의 7가지 기법은 Nanobot 소스 수정 없이 완전한 경제 추적 레이어를 삽입한다. 이 패턴은 다른 프레임워크 확장에도 재사용 가능하다.

3. **"Zero Own Key" 아키텍처가 새로운 설계 패턴을 제시한다.** 여러 앱이 동일 OpenClaw 게이트웨이를 공유할 수 있으며, 각 앱이 별도 AI 키를 관리할 필요가 없다. 단일 게이트웨이가 인증, 비용 추적, 대화 관리를 담당한다.

### 열린 질문

16. **ClawWork의 경제 모델이 실제 에이전트 능력 측정에 타당한가?** $10 시작, 작업당 BLS 임금 지급은 설계상 인위적이다. "경제적 생존"이 "실제 업무 능력"의 유효한 프록시인가?

17. **비침습적 확장 7가지 기법 중 최선은 무엇인가?** ClawWork는 런타임 클래스 교체(`__class__` 할당)를 사용했는데, 이는 파이썬에서 기술적으로 가능하지만 예측하기 어렵다. 더 안전한 추상화가 있는가?

18. **ClawPort의 LocalStorage 대화 저장이 확장 가능한가?** 이미지 많은 대화에서 5MB 한계에 도달한다. 서버 사이드 저장으로의 전환이 "Zero Own Key" 아키텍처와 충돌하는가?

19. **"Zero Own Key" 아키텍처의 단일 실패 지점 문제.** ClawPort는 OpenClaw 게이트웨이 없이 완전히 작동 불가다. 오프라인 에이전트 팀 모니터링을 위한 fallback은 어떻게 설계할 수 있는가?
