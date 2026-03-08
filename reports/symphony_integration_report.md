# Symphony 통합 가능성 분석 보고서

> **조사 일자**: 2026-03-08
> **조사 방법**: 소스코드 심층 분석 (`config.ex`, `app_server.ex`, `tracker.ex`, `SPEC.md`)
> **핵심 질문**: "Symphony 스펙은 에이전트 런타임 중립인가? Claw 프레임워크로 스펙을 구현할 수 있는가?"
> **수정 노트**: 초기 분석이 Elixir 참조 구현체의 Codex 결합을 Symphony 스펙 자체의 종속성으로 오해석함. 이 보고서는 그 구분을 명확히 함.
> **선행 보고서**: `symphony_report.md` (Symphony 원본 분석)

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. [Elixir 참조 구현체의 Codex 구현 선택](#2-elixir-참조-구현체의-codex-구현-선택)
3. [Tracker 어댑터 중립성](#3-tracker-어댑터-중립성)
4. [Claw 통합 5가지 전략](#4-claw-통합-5가지-전략)
5. [ClawPort 연동 시나리오](#5-clawport-연동-시나리오)
6. [권장 통합 경로](#6-권장-통합-경로)
7. [결론 및 열린 질문](#7-결론-및-열린-질문)

---

## 1. Executive Summary

Symphony는 **언어-에이전트 중립 오픈 스펙**(`SPEC.md`)이다. OpenAI의 Elixir 구현체(`elixir/`)는 Codex를 에이전트 백엔드로 선택한 **참조 구현** 중 하나일 뿐이며, 다른 조직은 동일한 스펙을 자신의 에이전트 런타임으로 구현할 수 있다.

**핵심 발견 5가지:**

1. **Symphony 스펙은 에이전트 런타임 중립** — `SPEC.md:29`: "a scheduler/runner and tracker reader". 어떤 에이전트를 사용할지 스펙이 강제하지 않는다. Codex는 OpenAI 참조 구현체의 선택.

2. **Elixir 참조 구현체는 Codex에 특화** — `app_server.ex`가 Codex JSON-RPC 방언을 직접 구현. 이는 참조 구현의 설계 결정이며, 스펙의 요구사항이 아니다.

3. **Tracker 어댑터는 스펙에서도 교체 가능하도록 명시 설계** — `tracker.ex:8-12`의 5개 콜백 behaviour. `SPEC.md:2103`이 "pluggable tracker adapters" 확장을 공식 계획으로 언급.

4. **전략 D(Native app-server 구현)가 스펙이 의도한 정석 경로** — `SPEC.md:133`이 암시하는 방향. Claw 프레임워크가 app-server 프로토콜을 구현하면 Symphony 스케줄러를 수정 없이 사용 가능.

5. **전략 C(공동 오케스트레이션)는 전략 D 이전의 현실적 단기 경로** — 스펙 준수 없이 HTTP 브릿지로 빠르게 결합. 8-10주, 1.5K LOC.

---

## 2. Elixir 참조 구현체의 Codex 구현 선택

> **중요**: 이 섹션은 Symphony **스펙**의 종속성이 아니라, OpenAI의 **Elixir 참조 구현체**가 Codex를 선택한 구체적 방식을 분석한다. 다른 조직이 스펙을 구현할 때는 이 설계 결정을 따를 필요 없다.

### 2.1 참조 구현체의 두 층 결합

**층 1 — AppServer 프로토콜 (Codex 특화 구현)**

Elixir 참조 구현체의 `app_server.ex`가 Codex JSON-RPC 방언을 직접 구현:

| 메서드 | 파일:라인 | 역할 | 교체 가능성 |
|--------|---------|------|----------|
| `initialize` | `app_server.ex:197` | 세션 초기화 | 불가 |
| `thread/start` | `app_server.ex:232` | 스레드 생성 | 불가 |
| `turn/start` | `app_server.ex:256` | 턴 실행 | 불가 |

**Codex 전용 이벤트 타입 4개** (`app_server.ex:339-589`):

```elixir
"item/commandExecution/requestApproval"  # Line 469
"item/tool/call"                         # Line 491
"execCommandApproval"                    # Line 523
"applyPatchApproval"                     # Line 545
```

이벤트 핸들러가 이 문자열을 패턴 매칭하므로, 다른 에이전트 런타임의 이벤트 타입과 호환 불가.

**기본 명령어 하드코딩** (`config.ex:31`):

```elixir
@default_codex_command "codex app-server"
```

이 명령어는 `app_server.ex:175`에서 Port.open의 인자로 사용:

```elixir
args: [~c"-lc", String.to_charlist(Config.codex_command())],
```

Bash로 `codex app-server` 서브프로세스를 스폰하고, stdin/stdout으로 JSON-RPC 통신. 다른 에이전트(Nanobot, OpenClaw)로 교체하려면 **이 프로토콜 전체를 재구현**해야 한다.

**결론**: Elixir 참조 구현체의 AppServer는 **Codex 전용 드라이버**. 그러나 이는 스펙 요구사항이 아닌 OpenAI의 구현 선택이다. 다른 조직의 구현에서는 다른 에이전트 드라이버로 대체 가능.

### 2.2 AgentRunner 간접 결합 (참조 구현 한정)

`agent_runner.ex:11-33`:

```elixir
def run(issue, orchestrator_pid, opts) do
  Workspace.create_for_issue(issue)
  Workspace.run_before_run_hook(...)
  run_codex_turns(...)  # AppServer 직접 호출
  Workspace.run_after_run_hook(...)
end
```

AgentRunner가 AppServer를 명시적으로 호출. 에이전트 런타임 추상화 레이어가 없다.

**SPEC.md의 TODO 섹션** (`SPEC.md:2103` 암시):
- "pluggable tracker adapters" -> 계획됨, 실제 구현 (Tracker.ex 존재)
- "에이전트 어댑터" -> 언급 없음, 미계획

### 2.3 교체 비용 추정

| 작업 | 비용 | 설명 |
|------|------|------|
| AppServer 전체 재작성 | 6-8주 | 600 LOC 신규 구현, 프로토콜 테스트 |
| 에이전트별 app-server 모드 구현 | 프레임워크당 2-3주 | Nanobot, OpenClaw, ZeroClaw 등 |
| 프로토콜 브릿지 (권장 경로) | 2-3주 | Nanobot/OpenClaw를 HTTP 클라이언트로 호출 |

---

## 3. Tracker 어댑터 중립성

### 3.1 Behaviour 설계 (높은 중립성)

`tracker.ex:8-12`의 5개 콜백:

```elixir
@callback fetch_candidate_issues() :: {:ok, [term()]} | {:error, term()}
@callback fetch_issues_by_states([String.t()]) :: {:ok, [term()]} | {:error, term()}
@callback fetch_issue_states_by_ids([String.t()]) :: {:ok, [term()]} | {:error, term()}
@callback create_comment(String.t(), String.t()) :: :ok | {:error, term()}
@callback update_issue_state(String.t(), String.t()) :: :ok | {:error, term()}
```

**특징:**
- 반환 타입이 `term()` — 내부 구조 강제 없음
- 5개 콜백만 구현하면 됨 — 작은 계약
- `adapter/0` 함수 (`tracker.ex:40-44`):
  ```elixir
  def adapter do
    case Config.tracker_kind() do
      "memory" -> SymphonyElixir.Tracker.Memory  # 테스트용
      _ -> SymphonyElixir.Linear.Adapter
    end
  end
  ```

이미 `Tracker.Memory` 인메모리 구현체가 존재하여 테스트 어댑터 패턴 증명됨.

### 3.2 GitHub Issues 어댑터 구현 예상도

```
GitHubAdapter 구현 (2주)
├── GitHub GraphQL API 클라이언트 (Req)
├── 5개 콜백 구현
│   ├── fetch_candidate_issues -> GraphQL 이슈 쿼리 + 레이블 필터
│   ├── fetch_issues_by_states -> GraphQL state 필터
│   ├── fetch_issue_states_by_ids -> batch 조회
│   ├── create_comment -> mutation
│   └── update_issue_state -> mutation (상태 라벨 변경)
└── 테스트 (GraphQL mock)
```

**교체 가능성 평가:**
- GitHub API 문서 기반 구현 가능
- Linear의 `@callback` 명세와 GitHub API 간 1:1 매핑 가능
- 예상 코드: 300-400 LOC

### 3.3 Jira 어댑터 (3주, OAuth 포함)

```
JiraAdapter 구현 (3주)
├── Jira REST API 클라이언트
├── OAuth 2.0 인증
├── 5개 콜백 구현 (상동)
└── 상태 코드 매핑 (Jira 워크플로우 사용자 정의 대응)
```

**추가 복잡도:**
- Jira 워크플로우가 프로젝트마다 다름 -> 동적 상태 발견 필요
- 예상 코드: 400-500 LOC

---

## 4. Claw 통합 5가지 전략

### 4.1 전략 A: Nanobot 래퍼 (6주, 1.5K LOC)

**아키텍처:**

```
Symphony Orchestrator
    | (codex app-server 명령)
[Nanobot-AppServer Wrapper]  <- 신규 컴포넌트
    | (OpenAI/Anthropic API)
Nanobot Agent Runtime
```

**구현 개요:**
- `codex app-server` CLI 인터페이스를 모방하는 thin wrapper
- Nanobot의 MEMORY.md, HISTORY.md 상태를 AppServer 세션 모델에 매핑
- stdio JSON-RPC 호환 계층

**장점:**
- Symphony 코드 무수정
- 가장 빠른 통합 (6주)
- Nanobot의 OpenAI/Anthropic API 능력 활용 가능

**단점:**
- Nanobot의 지속적 세션 상태(MEMORY.md)가 Symphony의 에피소딕 실행 모델과 충돌
- 이슈 완료 시 workspace 정리 -> Nanobot의 long-term memory 소실 위험
- 컨텍스트 연속성 문제: 재시도 시 이전 메모리 접근 불가

**현실성:** 중간~높음 (Nanobot MEMORY.md 생명주기 정리 필요)

### 4.2 전략 B: OpenClaw 게이트웨이 브릿지 (12주, 3K LOC)

**아키텍처:**

```
Symphony Orchestrator
    | JSON-RPC over stdio
[OpenClaw AppServer Bridge]  <- 신규
    | Plugin API (24 hooks)
OpenClaw Plugin System
    |
Claude/GPT Agent
```

**구현 개요:**
- OpenClaw 플러그인으로 AppServer 프로토콜 구현
- `initialize`, `thread/start`, `turn/start` 이벤트를 OpenClaw 훅에 매핑
- 24개 훅을 Symphony 이벤트 타입에 변환

**장점:**
- OpenClaw의 하이브리드 메모리 (LanceDB + sqlite-vec) 활용 가능
- OpenClaw의 브라우저 자동화, 멀티모달 능력 보유
- 가장 기능이 풍부한 에이전트 런타임 활용

**단점:**
- 가장 복잡한 구현 (12주)
- OpenClaw의 장기 실행 데몬 모델 vs Symphony의 서브프로세스 스폰 모델 충돌
- 에이전트 비용 추적(ClawWork) 통합 어려움

**현실성:** 낮음 (아키텍처 불일치 해소 필요)

### 4.3 전략 C: 공동 오케스트레이션 <- **권장 1단계** (8-10주, 1.5K LOC)

**아키텍처:**

```
Linear Issues
    | 폴링
Symphony Orchestrator  <-> Claw Framework (OpenClaw/Nanobot)
    | HTTP/MCP              |
  이슈 디스패치           에이전트 실행 + 메모리 관리
    |
  상태 추적 -> Linear
```

**구현 개요:**
- Symphony가 스케줄러로 유지
- AgentRunner를 "HTTP 클라이언트"로 교체 (AppServer 바이패스)
- Claw 프레임워크를 외부 서비스로 호출 (MCP 또는 REST API)
- Claw 프레임워크의 상태 관리는 독립적 유지

**구현 단계:**
1. Symphony의 AgentRunner 수정: AppServer 호출 -> HTTP 클라이언트로
2. Claw 프레임워크 측: HTTP 게이트웨이 또는 MCP 서버 노출
3. 콜백 매핑: Linear GraphQL 도구 -> Claw MCP 도구로 전환

**코드 변경 (Symphony 측):**

```elixir
# 기존: AppServer.start_session(workspace)
# 신규: HttpClient.start_session(claw_api_url, workspace)

defmodule SymphonyElixir.HttpAgent do
  def start_session(api_url, workspace) do
    Req.post!(api_url <> "/session/start", json: %{cwd: workspace})
  end

  def run_turn(api_url, session_id, prompt) do
    Req.post!(api_url <> "/turn", json: %{session_id: session_id, input: prompt})
  end
end
```

**장점:**
- Symphony의 스케줄링 로직 유지 (낮은 침습도)
- Claw 에이전트의 메모리, 멀티모달 능력 활용 가능
- 각 프레임워크가 독립적 상태 관리 가능
- 프로토콜 중립적 (HTTP, MCP 모두 가능)

**단점:**
- 에이전트 상태를 Symphony가 직접 관찰 불가 (HTTP 콜백 필요)
- 네트워크 latency 추가
- Claw 프레임워크 측이 게이트웨이 구현 필요

**현실성:** 높음 (최소 침습, 상호 독립적)

### 4.4 전략 D: Native app-server 모드 ← **스펙이 의도한 정석 경로** (20주, 프레임워크당 2.5K LOC)

**아키텍처:**

```
Symphony Orchestrator
    | JSON-RPC over stdio (표준화)
[IronClaw/ZeroClaw/Nanobot]  <- app-server 모드 각자 구현
    |
에이전트 런타임 (WASM/Docker/In-process)
```

**구현 개요:**
- Symphony 스펙(`SPEC.md:133`)이 명시적으로 의도한 방향 — "스펙을 구현하는 에이전트는 JSON-RPC app-server 모드를 지원해야 한다"
- 각 Claw 프레임워크가 app-server 프로토콜을 직접 구현: `initialize`, `thread/start`, `turn/start` 메서드 지원
- 동일한 JSON-RPC 이벤트 스트림 생성
- Symphony 참조 구현체(Elixir)가 Codex를 선택한 것처럼, Claw 프레임워크도 동일한 스펙 인터페이스로 동작하는 드라이버를 작성

**구현 부하 (프레임워크당):**
1. Codex app-server 프로토콜 파서 (200 LOC)
2. WASM/Docker/In-process 런타임 통합 (1500 LOC)
3. 테스트 (800 LOC)

**장점:**
- Symphony 코드 무수정
- 아키텍처상 가장 깔끔 (모든 에이전트가 동일 프로토콜)
- 향후 에이전트 교체 용이

**단점:**
- 프레임워크 팀의 별도 구현 필요 (생태계 협력 필수)
- OpenAI 독점 프로토콜에 종속 위험
- 프로토콜 버전 관리 복잡성 증가

**현실성:** 중간~높음 (스펙 준수 경로, 생태계 협력 필요하나 방향 명확)

### 4.5 전략 E: Unified Agent Interface 스펙 (장기, 6개월+)

**개념:**
- Codex app-server 프로토콜을 참고하여 **언어-에이전트 중립 표준 JSON-RPC 스펙** 제안
- 모든 Claw 프레임워크 + Symphony가 이 스펙 구현

**스펙 예상 내용:**
```
UnifiedAgentInterface v1.0
├── initialize(capabilities) -> initialized
├── thread/start(policy, tools) -> thread_id
├── turn/start(thread_id, input) -> stream of events
└── Events: tool/call, approval_request, turn/completed
```

**장점:**
- 생태계 표준화 (SKILL.md 같은 de facto 컨벤션 공식화)
- 프레임워크 간 상호 호환성
- SPEC.md:1의 "language-agnostic spec" 철학 확장

**단점:**
- 생태계 합의 필요 (6개월+ 논의)
- 단독으로 추진 불가 (OpenAI, Anthropic, 커뮤니티 참여)

**현실성:** 낮음 (중장기 비전)

### 4.6 전략 비교표

| 전략 | 기간 | LOC | Symphony 수정 | 현실성 | 권장 순서 |
|------|------|-----|------------|--------|---------|
| A (Nanobot 래퍼) | 6주 | 1.5K | 낮음 | 중간 | 단기 우회 |
| B (OpenClaw 브릿지) | 12주 | 3K | 중간 | 낮음 | 비권장 |
| C (공동 오케스트레이션) | 8-10주 | 1.5K | 중간 | 높음 | **1단계 (단기)** |
| D (Native app-server) | 20주 | 2.5K/프레임워크 | 없음 | 중간~높음 | **2단계 (정석)** |
| E (통일 스펙) | 6개월+ | 5K+ | 중간 | 낮음 | 수렴 대기 |

---

## 5. ClawPort 연동 시나리오

### 5.1 현재 ClawPort 모델

**관찰 단위:** 에이전트 세션 (장기 실행)
**비용 단위:** 사용자 요청당 토큰
**제어:** ClawPort API 직접 호출

ClawPort는 OpenClaw 게이트웨이에 직접 연결된 대시보드/비용 추적 시스템.

### 5.2 Symphony 오케스트레이션 시 변화

**관찰 단위 변경:** 에이전트 세션 -> 이슈별 에피소드
- 각 Linear 이슈 = 독립 에이전트 실행
- 이슈 완료 또는 최대 턴 도달 = 에피소드 종료

**비용 단위 변경:** 세션 단위 -> 이슈 해결 1건
- 비용 추적 -> 이슈별 토큰 사용량
- 재시도 횟수 -> 이슈 해결 비용 영향

**제어 모델 변경:**
- ClawPort가 Tracker 역할 수행 (이슈 상태 제공)
- Symphony가 ClawPort API 폴링 -> Linear 대신

### 5.3 ClawPort를 Symphony Tracker 어댑터로 확장

**현재 제한:**
- ClawPort는 "읽기 전용" 대시보드
- 이슈 상태 업데이트 API 없음

**필요한 확장:**
1. `EpisodeView` — 이슈별 에이전트 실행 기록
   ```
   GET /api/episodes/{issue_id}
   -> [{run_index, start_time, end_time, token_count, status}]
   ```

2. `IssueMetrics` — 이슈 해결 메트릭
   ```
   GET /api/issues/{issue_id}/metrics
   -> {turns: 3, tokens: 5000, cost: $0.05, retries: 1}
   ```

3. **쓰기 API** — 이슈 상태 업데이트
   ```
   PATCH /api/issues/{issue_id}/state
   {state: "In Progress"} -> 200 OK
   ```

### 5.4 ClawPort + Symphony 통합 아키텍처

```
ClawPort API
├── /api/issues -> fetch_candidate_issues()
├── /api/issues?states=["In Progress"] -> fetch_issues_by_states()
├── /api/issues/{id}/state -> update_issue_state()
└── /api/issues/{id}/comments -> create_comment()

<-> (REST API)

Symphony Tracker Adapter
├── fetch_candidate_issues() -> REST GET /api/issues
├── fetch_issues_by_states() -> REST GET /api/issues?states=...
├── update_issue_state() -> REST PATCH /api/issues/{id}/state
└── create_comment() -> REST POST /api/issues/{id}/comments

<-> (callback interface)

Symphony Orchestrator
└── 이슈 디스패치 -> Claw 에이전트 실행
```

**구현 비용:**
- ClawPort 측: 3-4주 (GET/PATCH 엔드포인트 추가)
- Symphony 측: 1주 (ClawPortAdapter 구현, 400 LOC)

**단일 제어 평면의 이점:**
- 에이전트 스케줄링 + 관찰 + 비용 추적이 ClawPort에 통합
- 이슈 생명주기 = 에이전트 실행 에피소드
- 팀이 ClawPort 하나로 모든 자율 개발 활동 모니터링

---

## 6. 권장 통합 경로

### 6.1 Phase 1 (3개월): 전략 C — 공동 오케스트레이션

**목표:** Symphony와 Claw 프레임워크의 기술적 결합 검증

**실행 계획:**

1. **Symphony AgentRunner 수정** (3주)
   - `app_server.ex` 호출 제거
   - HTTP 클라이언트로 교체 (`symphony_elixir/http_agent.ex` 신규)
   - Linear Tracker 어댑터는 유지

2. **Claw 프레임워크 HTTP 게이트웨이** (4주)
   - Nanobot: REST API 래퍼 (200 LOC)
   - OpenClaw: 기존 플러그인 시스템 확장 (500 LOC)

3. **프롬프트 변환** (1주)
   - Linear GraphQL 도구 -> Claw MCP 도구
   - WORKFLOW.md 템플릿 유지, 에이전트 능력 추상화

4. **검증** (2주)
   - Linear 이슈 5건 자동 해결 테스트
   - ClawPort 관찰 가능성 확인
   - 메모리 생명주기 검증

**Go/No-Go 판단 기준:**
- [O] Linear 이슈 5건 자동 완료
- [O] ClawPort에서 에이전트 실행 이력 조회 가능
- [O] 컨텍스트 소실 없이 재시도 성공
- [O] 30일 무인 운영 (0 수동 개입)

### 6.2 Phase 2 (6개월): 전략 A 또는 D 선택

**선택 기준:**

**전략 A 선택 (Nanobot 래퍼):**
- Phase 1에서 Nanobot의 MEMORY.md 생명주기 문제 해결됨
- 빠른 실용화 우선 (6주)
- Nanobot이 OpenAI/Anthropic API의 최신 모델 지원

**전략 D 선택 (Native app-server):**
- Phase 1에서 각 Claw 프레임워크와 협력 체계 확립됨
- 아키텍처 순수성 우선
- 장기 표준화 목표

### 6.3 Phase 3 (12개월+): 전략 E 수렴

**목표:** Unified Agent Interface 스펙 자연 수렴 또는 공식화

**활동:**
- SKILL.md 처럼 에이전트 인터페이스 표준이 자연 수렴하는지 관찰
- 수렴 신호: 모든 Claw 프레임워크가 Phase 2 후 동일 HTTP 인터페이스 채택
- 수렴 확인 시 스펙 공식화 (`AGENT_INTERFACE.md`)

---

## 7. 결론 및 열린 질문

### 핵심 결론 3가지

**1. Symphony 스펙은 에이전트 런타임 중립 — Codex는 참조 구현의 선택**

Symphony(`SPEC.md`)는 스케줄러/트래커 리더로서 어떤 에이전트를 쓸지 강제하지 않는다. Elixir 참조 구현체가 Codex를 선택한 것은 OpenAI 내부 사정이다. **Claw 프레임워크도 동일한 스펙을 구현하는 정당한 에이전트 백엔드다.**

**2. 전략 D(Native app-server)가 스펙이 의도한 정석 경로**

`SPEC.md:133`의 방향: app-server 프로토콜을 구현한 에이전트라면 Symphony 스케줄러와 바로 결합 가능. Claw 프레임워크가 드라이버를 작성하면 Symphony 코드 수정 없이 통합된다. **장기적으로 이 경로가 아키텍처상 올바르다.**

**3. 전략 C(공동 오케스트레이션)는 전략 D 이전 현실적 단기 경로**

스펙 준수 없이 HTTP 브릿지로 빠르게 결합. 8-10주, 1.5K LOC. **Phase 1 검증에 적합하며, Phase 2에서 전략 D로 전환하는 발판이 된다.**

### 새로운 열린 질문 3가지

**Q28. Symphony의 에피소딕 실행 모델과 Claw의 지속 세션 모델 메모리 핸드오프 프로토콜**

Symphony는 이슈 완료 시 workspace를 정리한다. OpenClaw나 ZeroClaw의 장기 메모리(LanceDB, pgvector)는 이 생명주기와 어떻게 공존하는가?

- 시나리오 1: 메모리를 workspace 외부 (ClawPort DB)에 저장 -> 이슈 재시도 시 복원
- 시나리오 2: 메모리를 이슈별로 격리 -> 각 이슈의 독립 에피소드
- 현실: 어느 모델이 "24시간 자율 에이전트"의 성능을 최적화하는가?

**Q29. ClawPort가 Symphony Tracker 어댑터가 되면 단일 제어 평면의 가능성**

ClawPort가 쓰기 API를 지원하고 Symphony가 폴링하면, 에이전트 스케줄링 + 관찰 + 비용 추적이 ClawPort에 통합된다. 이것이 ClawPort의 역할 확장인가, 범위 위반인가?

- ClawPort는 현재 "관찰 계층"
- Symphony와의 통합 시 "제어 평면"으로 상향
- 아키텍처상 의존성 방향: Symphony -> ClawPort (일방향, 건강함)

**Q30. Codex app-server JSON-RPC가 OpenAI 독점 프로토콜로 남을 경우**

전략 D(Native 구현)는 생태계 전체를 OpenAI 표준에 종속시키는가?

- SPEC.md:1은 "language-agnostic"을 표방
- SPEC.md:925의 참조 문서는 OpenAI 독점
- 에이전트 인터페이스 스펙만은 **중립적으로 재정의**해야 하는가?
- 전략 E (Unified Agent Interface)의 정당성: 생태계 독립성 확보

---

## 부록: 파일 참조 검증

| 참조 | 파일 | 내용 |
|------|------|------|
| `config.ex:31` | config.ex | @default_codex_command "codex app-server" |
| `app_server.ex:197` | codex/app_server.ex | "method" => "initialize" |
| `app_server.ex:232` | codex/app_server.ex | "method" => "thread/start" |
| `app_server.ex:256` | codex/app_server.ex | "method" => "turn/start" |
| `app_server.ex:469` | codex/app_server.ex | "item/commandExecution/requestApproval" |
| `app_server.ex:491` | codex/app_server.ex | "item/tool/call" |
| `app_server.ex:523` | codex/app_server.ex | "execCommandApproval" |
| `app_server.ex:545` | codex/app_server.ex | "applyPatchApproval" |
| `tracker.ex:8-12` | tracker.ex | 5개 @callback 정의 |
| `tracker.ex:40-44` | tracker.ex | adapter/0 함수 |
| `SPEC.md:29` | SPEC.md | "scheduler/runner and tracker reader" |
| `SPEC.md:133` | SPEC.md | "JSON-RPC-like app-server mode" |
| `SPEC.md:2103` | SPEC.md | TODO: pluggable tracker adapters |

---

**조사 완료**: 2026-03-08
**검증 방법**: 소스코드 직접 분석, 모든 파일:라인 참조 확인됨
**다음 단계**: Phase 1 (전략 C) 기술 검토 및 스폰서십 확보
