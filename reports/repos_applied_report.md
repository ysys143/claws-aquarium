# Claw 생태계 응용 계층 비교 분석 — ClawWork, ClawPort, Moltbook

> **조사 일자**: 2026-03-07
> **조사 방법**: 3개 scientist 에이전트가 각 레포의 실제 소스코드를 병렬 심층 분석
> **핵심 질문**: "Claw 프레임워크 위에 구축된 응용 프로젝트들은 어떤 계층을 추가하고, 어떤 문제를 해결하며, 어떤 패턴을 공유하는가?"
> **선행 보고서**: session_context_report.md, security_report.md, browser_actions_report.md, memory_architecture_report.md

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. [repos_applied란 무엇인가](#2-repos_applied란-무엇인가)
3. [개별 분석](#3-개별-분석)
   - 3.1 ClawWork
   - 3.2 ClawPort (clawport-ui)
   - 3.3 Moltbook API
4. [교차 분석](#4-교차-분석)
5. [선행 보고서와의 연결](#5-선행-보고서와의-연결)
6. [결론 및 열린 질문](#6-결론-및-열린-질문)

---

## 1. Executive Summary

3개 응용 프로젝트의 소스코드를 분석한 결과, Claw 생태계의 **응용 계층**은 프레임워크 코드가 해결하지 않은 3가지 새로운 문제를 다룬다:

| 문제 | 해결 프로젝트 | 접근법 |
|------|-------------|--------|
| **에이전트 성능 측정** | ClawWork | 경제적 생존 게임 (GPT 평가 + BLS 임금 기반 지급) |
| **에이전트 팀 관찰/제어** | ClawPort | OpenClaw 게이트웨이 프록시 대시보드 (Org Map + 채팅 + 비용) |
| **에이전트 간 사회적 상호작용** | Moltbook | AI 에이전트 전용 소셜 네트워크 (Reddit-like API) |

**가장 주목할 발견 4가지:**

1. **프레임워크 비침습적 확장 패턴이 정립됐다.** ClawWork는 Nanobot 소스 수정 없이 7가지 기법(서브클래싱, 도구 추가, 메서드 오버라이드, 런타임 클래스 교체, 투명 래퍼, 스킬 주입, 설정 분리)으로 경제 추적 레이어를 삽입한다 (`clawmode_integration/provider_wrapper.py:37`, `agent_loop.py:46`).

2. **"Zero Own Key" 아키텍처가 새로운 패턴으로 등장했다.** ClawPort는 자체 Anthropic API 키를 전혀 보유하지 않는다. 9개 API 라우트 전체가 OpenClaw 게이트웨이(localhost:18789)에 위임하며, 이를 통해 사용자의 기존 인증과 대화 컨텍스트를 그대로 활용한다 (`app/api/chat/[id]/route.ts:9-12`).

3. **소셜 네트워크 설계가 AI 에이전트의 행동 패턴을 명시적으로 고려했다.** Moltbook API는 포스트 생성을 30분에 1회로 제한하는데, 이는 사람이 아닌 루프 실행 에이전트의 스팸을 방지하기 위한 의도적 설계다 (`src/config/index.js:28-32`). Reddit hot 알고리즘을 SQL로 직접 구현하여 외부 AI 추천 엔진 없이 피드를 구성한다 (`src/services/PostService.js:129`).

4. **SKILL.md가 응용 계층에서도 핵심 인터페이스다.** ClawWork는 경제 프로토콜을 `clawmode_integration/skill/SKILL.md`로 주입하고, Moltbook API 루트 엔드포인트는 `documentation: 'https://www.moltbook.com/skill.md'`를 반환한다 (`src/app.js:53`). browser_actions_report.md가 분석한 NanoClaw의 SKILL.md 패턴이 생태계 전체로 확산됐다.

---

## 2. repos_applied란 무엇인가

`repos/`가 Claw 런타임 프레임워크들(OpenClaw, Nanobot, NanoClaw, IronClaw, ZeroClaw, PicoClaw, TinyClaw, Moltbook)을 담고 있다면, `repos_applied/`는 그 프레임워크 **위에** 구축된 응용 프로젝트들이다.

```
┌─────────────────────────────────────────────────────┐
│                응용 계층 (repos_applied/)             │
│  ClawWork           ClawPort          Moltbook API   │
│  (벤치마크)          (대시보드)         (소셜 네트워크) │
└───────────────────────┬─────────────────────────────┘
                        │ 사용/확장
┌───────────────────────▼─────────────────────────────┐
│              프레임워크 계층 (repos/)                  │
│  Nanobot    OpenClaw    NanoClaw    IronClaw ...      │
└─────────────────────────────────────────────────────┘
```

| 프로젝트 | 기반 프레임워크 | 관계 유형 | 추가하는 계층 |
|----------|--------------|-----------|-------------|
| **ClawWork** | Nanobot | 확장 (서브클래싱 + 래핑) | 경제 추적 + 작업 평가 + 벤치마크 |
| **ClawPort** | OpenClaw | 프록시 (게이트웨이 위임) | UI + 관찰 + 제어 인터페이스 |
| **Moltbook API** | 독립 (생태계 인프라) | 플랫폼 (에이전트 활동 공간) | 에이전트 간 사회적 상호작용 |

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

### 3.3 Moltbook API — AI 에이전트 소셜 네트워크

**핵심 철학**: "에이전트가 서로 어울릴 공간이 필요하다" — 에이전트가 1급 사용자인 Reddit-like 플랫폼

#### 3.3.1 아키텍처

Express 4.18 + PostgreSQL 단순 모놀리스. 미들웨어 체인: helmet → cors → compression → morgan → body parsing → routes. 서비스 레이어는 static class 6개로 인스턴스화 없이 사용한다 (`src/services/`).

주목할 설계: devDependencies 완전 비어 있음 (`package.json:43`) — 테스트 프레임워크, TypeScript 전무. `test/api.test.js`가 존재하나 실제 테스트 러너 설정 없음.

루트 엔드포인트 (`src/app.js:53`):
```json
{ "documentation": "https://www.moltbook.com/skill.md" }
```
NanoClaw의 SKILL.md 패턴을 API 진입점에 명시적으로 채택했다.

#### 3.3.2 에이전트 신원 & 인증

```
POST /agents/register
  → api_key = "moltbook_" + randomBytes(32).hex  (73자)
  → api_key_hash = SHA-256(api_key)              ← DB에 저장
  → api_key                                       ← 응답에만 1회 반환, DB 미저장
```

원본 키는 DB에 저장되지 않는다 (`scripts/schema.sql:16`). JWT 설정이 `src/config/index.js:25`에 있으나 발급/검증 코드가 없다 — 데드 코드.

**Claim 시스템** — AI 에이전트 뒤의 인간을 식별하는 2단계 소유권 증명:

```
1. 등록: verification_code 발급 (예: "reef-X4B2", 해양 단어 16개 어휘)
2. 인간이 트위터에 verification_code 게시
3. is_claimed = true, owner_twitter_handle 저장
```

단, `AgentService.claim()` 메서드는 구현됐으나 이를 호출하는 HTTP 라우트가 없다 — 미완성 (`src/middleware/auth.js:64-81`). 별도 `moltbook-auth` 저장소에 있을 가능성이 있다 (`.github/workflows/auto-reply-issue.yml`에서 8개 분리 저장소 확인).

#### 3.3.3 피드 알고리즘 — SQL 내장 Reddit hot

4종 정렬 알고리즘이 모두 SQL로 구현됐다. 외부 추천 엔진 없음:

```sql
-- Hot (src/services/PostService.js:129)
LOG(GREATEST(ABS(p.score), 1)) * SIGN(p.score)
+ EXTRACT(EPOCH FROM p.created_at) / 45000 DESC

-- Rising (시간 감쇠 1.5승 적용)
(p.score / POWER(EXTRACT(EPOCH FROM NOW() - p.created_at)/3600 + 2, 1.5)) DESC
```

개인화 피드는 "구독 submolt OR 팔로우 에이전트" 합집합을 순수 SQL JOIN으로 처리한다 (`src/services/PostService.js:182-193`).

#### 3.3.4 댓글 트리 — 애플리케이션 레이어 재구성

PostgreSQL 재귀 CTE를 사용하지 않고 평탄한 쿼리 결과를 2패스로 트리 재구성한다:

```javascript
// src/services/CommentService.js:120-139
// Pass 1: Map 생성 (id → node)
// Pass 2: parent_id로 연결 (root 수집)
```

최대 깊이 10단계, 소프트 삭제(`content = "[deleted]"` 교체)로 트리 구조를 유지한다.

#### 3.3.5 DB 설계 특이점

**비정규화 카운터 전략** — `follower_count`, `following_count`, `subscriber_count`, `comment_count`를 행에 직접 저장하고 트랜잭션 내 원자적 UPDATE로 동기화 (`src/config/database.js:91-111`).

`posts` 테이블에 `submolt_id UUID`(외래키)와 `submolt VARCHAR(24)`(이름)를 중복 저장 — 검색 성능을 위한 의도적 비정규화 (`scripts/schema.sql:89-90`).

#### 3.3.6 AI 에이전트를 위한 레이트 리미팅

| 타입 | 한도 | 윈도우 | 의도 |
|------|------|--------|------|
| 일반 요청 | 100 | 1분 | 기본 DoS 방어 |
| 포스트 생성 | **1** | **30분** | 에이전트 루프 스팸 방지 |
| 댓글 | 50 | 1시간 | 적정 허용 |

포스트 30분 1개 제한은 사람이 아닌 AI 에이전트의 루프 실행을 명시적으로 고려한 설계다. 단, Redis 연결 코드가 전무하고 인메모리 `Map`만 사용하므로 멀티 서버 환경에서 레이트 리미팅이 완전히 우회된다 (`src/middleware/rateLimit.js`).

#### 3.3.7 MoltBot — Claude API 직접 호출

`.github/workflows/auto-reply-issue.yml`: 모든 GitHub 이슈에 `claude-sonnet-4-20250514`를 직접 호출해 자동 응답. 봇 루프 방지 조건 포함. 자동 라벨: bug/enhancement/question/api/frontend/documentation + 항상 needs-triage.

시스템 프롬프트에 8개 분리 저장소(moltbook-api, moltbook-auth, moltbook-rate-limiter 등)가 명시 — 마이크로서비스 분리 계획을 확인할 수 있다.

---

## 4. 교차 분석

### 4.1 Claw 프레임워크 의존도

| 프로젝트 | 의존 프레임워크 | 결합 방식 | 교체 용이성 |
|----------|--------------|-----------|------------|
| **ClawWork** | Nanobot | 서브클래싱 + 래핑 (비침습적) | 중간 (AgentLoop API 의존) |
| **ClawPort** | OpenClaw | 게이트웨이 프록시 (URL + 토큰) | 높음 (URL 변경으로 교체 가능) |
| **Moltbook API** | 없음 (독립) | SKILL.md 컨벤션만 차용 | N/A |

ClawPort는 OpenClaw API(`/v1/chat/completions`, `gateway call`)에만 의존하므로 이론적으로 호환 게이트웨이를 제공하는 다른 프레임워크로 교체 가능하다. ClawWork는 Nanobot 내부 클래스(`AgentLoop`, `LiteLLMProvider`)에 의존하므로 교체 비용이 더 높다.

### 4.2 SKILL.md의 생태계 확산

browser_actions_report.md에서 NanoClaw의 SKILL.md 패턴을 "SKILL.md + 컨테이너 위임"으로 분류했다. 응용 계층에서도 이 패턴이 확산됐다:

| 사용처 | SKILL.md 역할 |
|--------|-------------|
| NanoClaw (`repos/`) | 에이전트 행동 절차 정의 (컨테이너 IPC 통신) |
| ClawWork (`repos_applied/`) | 경제 프로토콜 주입 (`always: true`로 상시 로딩) |
| Moltbook API (`repos_applied/`) | API 문서 진입점 (`skill.md` URL 반환) |

### 4.3 비용 추적의 3가지 접근

3개 프로젝트가 각각 다른 수준에서 비용을 추적한다:

| 프로젝트 | 추적 단위 | 방법 | 세밀도 |
|----------|---------|------|--------|
| **ClawWork** | 작업(task) 단위 | 토큰 가로채기 → 잔액 실시간 차감 | 채널별(LLM/검색/OCR) |
| **ClawPort** | cron run 단위 | OpenClaw 실행 기록 파싱 | 모델별, 이상 감지 |
| **Moltbook** | 없음 | — | — |

ClawWork는 비용을 에이전트 생존에 직접 연결(비용 = 잔액 차감)하는 반면, ClawPort는 비용을 관찰 대상으로 시각화한다. Moltbook은 비용 개념이 없다.

### 4.4 미완성 패턴의 공통성

3개 프로젝트 모두 "설계됐으나 구현되지 않은" 부분을 가진다:

| 프로젝트 | 미완성 항목 | 코드 근거 |
|----------|-----------|-----------|
| ClawWork | Trading 시스템 | `livebench/trading/` 존재, `live_agent.py:189` 비활성화 |
| ClawWork | 포트폴리오 가치 | `economic_tracker.py:496` `net_worth = balance` TODO |
| ClawPort | 메모리 수정 | Memory UI 읽기 전용 |
| ClawPort | 멀티 워크스페이스 계층 | `agents-registry.ts:462-464` 부분 지원 |
| Moltbook | Claim HTTP 라우트 | `auth.js:64-81` 메서드 존재, 라우트 없음 |
| Moltbook | Redis 레이트 리미팅 | `.env.example:9` 선언, 구현 전무 |
| Moltbook | Feed 서비스 분리 | README 명시, `FeedService.js` 미존재 |

### 4.5 테스트 성숙도 격차

| 프로젝트 | 테스트 | 접근 |
|----------|--------|------|
| **ClawPort** | 536개 / 24 스위트 | 단위 테스트 + vi.mock 체계적 사용 |
| **ClawWork** | 없음 | 수동 실행 스크립트만 |
| **Moltbook** | `api.test.js` 존재, 러너 없음 | 형식적 파일만 |

ClawPort의 테스트 성숙도가 다른 두 프로젝트를 압도한다. ClawWork는 경제 엔진(`EconomicTracker`)이 복잡한 상태 관리를 담당하는데 테스트가 전혀 없다는 점이 특이하다.

---

## 5. 선행 보고서와의 연결

### 5.1 session_context_report.md와의 연결

session_context_report.md는 "프레임워크 레벨에서 자동으로 컨텍스트를 분리하는 구현체가 없다"는 결론을 내렸다. ClawWork의 독립 시뮬레이션 모드가 하나의 답을 보여준다 — **작업(task) 단위로 에이전트 실행을 완전히 분리**한다. 각 GDPVal 작업은 독립적인 LangChain 대화 세션으로 실행되며, 세션 간 컨텍스트 유출이 없다. 이는 "프로젝트 수명주기 관리 프레임워크"의 단순한 형태다.

### 5.2 memory_architecture_report.md와의 연결

ClawPort의 메모리 브라우저가 OpenClaw의 Tier 1 메모리 설정(`vectorWeight`, `halfLifeDays`, `mmrLambda`)을 직접 시각화한다. 기존 보고서가 분석한 "이중 주입 경로"와 "하이브리드 검색"이 실제로 UI에서 어떻게 보이는지를 ClawPort가 구체화했다.

### 5.3 browser_actions_report.md와의 연결

browser_actions_report.md가 "SKILL.md + 컨테이너 위임"을 NanoClaw 고유 패턴으로 분류했다. ClawWork와 Moltbook이 각각 SKILL.md를 다른 방식으로 활용함으로써 이 패턴이 특정 프레임워크를 넘어 생태계 전체의 인터페이스 컨벤션으로 확산됐음을 확인했다.

### 5.4 security_report.md와의 연결

security_report.md는 "자격증명 암호화를 구현한 곳은 IronClaw와 ZeroClaw뿐"이라는 결론을 내렸다. 응용 계층에서도 이 패턴이 반복된다:

- Moltbook API: SHA-256 해시만 저장 (원본 키 DB 미저장) — 기본적 크레덴셜 보호
- ClawPort: `OPENCLAW_GATEWAY_TOKEN` 환경변수, .env 파일 의존 — Tier 3 수준
- ClawWork: `~/.nanobot/config.json` 평문 저장 (Nanobot 상속) — Tier 3 수준

---

## 6. 결론 및 열린 질문

### 핵심 결론

1. **Claw 생태계의 응용 계층이 3가지 독립적 방향으로 분화됐다.** 벤치마크(ClawWork), 관찰/제어(ClawPort), 사회적 공간(Moltbook). 이 3가지가 상호 보완적이다 — ClawWork로 성능을 측정하고, ClawPort로 팀을 운영하고, Moltbook에서 에이전트들이 활동한다.

2. **비침습적 프레임워크 확장 패턴이 실용적으로 검증됐다.** ClawWork의 7가지 기법은 Nanobot 소스 수정 없이 완전한 경제 추적 레이어를 삽입한다. 이 패턴은 다른 프레임워크 확장에도 재사용 가능하다.

3. **AI 에이전트 전용 소셜 플랫폼 설계가 사람 대상 설계와 다른 결정을 요구한다.** 포스트 생성 30분 1회 제한, Claim 시스템(에이전트+인간 소유권 증명), skill.md API 문서화 — 모두 AI 에이전트를 1급 사용자로 설계할 때 나오는 결정이다.

4. **"Zero Own Key" 아키텍처가 새로운 설계 패턴을 제시한다.** 여러 앱이 동일 OpenClaw 게이트웨이를 공유할 수 있으며, 각 앱이 별도 AI 키를 관리할 필요가 없다. 단일 게이트웨이가 인증, 비용 추적, 대화 관리를 담당한다.

### 열린 질문

16. **ClawWork의 경제 모델이 실제 에이전트 능력 측정에 타당한가?** $10 시작, 작업당 BLS 임금 지급은 설계상 인위적이다. "경제적 생존"이 "실제 업무 능력"의 유효한 프록시인가?

17. **비침습적 확장 7가지 기법 중 최선은 무엇인가?** ClawWork는 런타임 클래스 교체(`__class__` 할당)를 사용했는데, 이는 파이썬에서 기술적으로 가능하지만 예측하기 어렵다. 더 안전한 추상화가 있는가?

18. **ClawPort의 LocalStorage 대화 저장이 확장 가능한가?** 이미지 많은 대화에서 5MB 한계에 도달한다. 서버 사이드 저장으로의 전환이 "Zero Own Key" 아키텍처와 충돌하는가?

19. **AI 에이전트 소셜 네트워크에서 스팸 방어의 근본적 한계는 무엇인가?** Moltbook의 레이트 리미팅은 인메모리라 멀티 서버 우회가 쉽다. 에이전트가 복수의 API 키를 등록할 수 있다면 포스트 30분 1회 제한은 의미가 없다. AI 에이전트 전용 플랫폼에서 스팸 방어는 어떤 다른 메커니즘이 필요한가?

20. **Claim 시스템(에이전트-인간 소유권 연결)이 실제로 필요한가?** idea2.md의 맥락에서, 에이전트가 특정 인간에게 귀속된다는 사실을 플랫폼이 알아야 하는 이유는 무엇인가? 익명 에이전트 vs 귀속된 에이전트의 행동 차이가 있는가?

21. **"Zero Own Key" 아키텍처의 단일 실패 지점 문제.** ClawPort는 OpenClaw 게이트웨이 없이 완전히 작동 불가다. 오프라인 에이전트 팀 모니터링을 위한 fallback은 어떻게 설계할 수 있는가?
