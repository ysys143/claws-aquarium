# OpenAI Symphony 심층 분석 보고서

> **조사 일자**: 2026-03-08
> **조사 방법**: 4개 scientist 에이전트가 실제 소스코드를 병렬 심층 분석 (A: 아키텍처, B: 오케스트레이션, C: CI/PR/Linear 통합, D: Claw 생태계 비교)
> **핵심 질문**: "OpenAI Symphony는 Claw 생태계와 어떤 패턴을 공유하고, 어떤 공백을 채우며, 어떤 새로운 질문을 제기하는가?"
> **선행 보고서**: repos_applied_report.md (ClawWork, ClawPort, Moltbook 분석)

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. [Symphony란 무엇인가 — 포지셔닝](#2-symphony란-무엇인가--포지셔닝)
3. [아키텍처 분석](#3-아키텍처-분석)
   - 3.1 Tech Stack
   - 3.2 핵심 모듈 구조
   - 3.3 데이터 플로우
4. [에이전트 오케스트레이션](#4-에이전트-오케스트레이션)
   - 4.1 에이전트 생명주기
   - 4.2 스케줄링 알고리즘
   - 4.3 Codex 통합 (AppServer + DynamicTool)
   - 4.4 실패 처리와 재시도
5. [CI/PR/Linear 통합](#5-ciprlinear-통합)
   - 5.1 Codex Skill 시스템
   - 5.2 Linear 이슈 생명주기
   - 5.3 Proof-of-Work 계층
   - 5.4 Land/Push/Pull 워크플로
   - 5.5 Specs Check 시스템
6. [Claw 생태계와의 비교](#6-claw-생태계와의-비교)
   - 6.1 공유 패턴
   - 6.2 핵심 차이점
   - 6.3 Symphony가 채우는 공백
7. [결론 및 열린 질문](#7-결론-및-열린-질문)

---

## 1. Executive Summary

OpenAI Symphony는 **이슈 트래커를 폴링하여 코딩 에이전트를 자동 디스패치하고, PR 랜딩까지 감시하는 운영 자동화 레이어**다. Elixir/OTP로 구현됐으며, 소스코드(`elixir/`)와 언어 독립적 사양(`SPEC.md`)을 분리한다.

**가장 주목할 발견 5가지:**

1. **Claw 생태계에 없던 3번째 레이어가 확인됐다.** repos/(프레임워크)와 repos_applied/(응용) 사이에, 에이전트 풀 스케줄링을 담당하는 "운영 자동화 레이어"가 독립적으로 존재한다. Symphony가 이 레이어의 구체적 구현체다.

2. **SKILL.md 컨벤션이 OpenAI에서도 독립적으로 채택됐다.** Symphony의 `.codex/skills/{name}/SKILL.md` 구조는 NanoClaw 원조와 동일하지만, YAML 프론트매터(`name`, `description`)와 교차 참조로 한 단계 공식화됐다. `land/SKILL.md`는 비동기 Python 감시 헬퍼(`land_watch.py`)까지 포함한 실행 플레이북이다.

3. **단일 `WORKFLOW.md`가 정책과 프롬프트를 동시에 버전 관리한다.** YAML 프론트매터(런타임 설정: 폴링 간격, 동시성, 훅)와 Liquid 템플릿 본문(에이전트 프롬프트)이 하나의 파일에 공존한다. 코드 변경 없이 `WorkflowStore`가 1초마다 mtime+phash2로 핫리로드한다 (`workflow_store.ex:141-148`).

4. **다중 계층 Proof-of-Work가 에이전트 자율성을 통제한다.** 로컬 `make all` → CI `make all` → PR 본문 린트(`pr_body.check`) → Codex AI 리뷰 승인 → squash-merge의 5단계 검증 체인 중 하나라도 실패하면 PR이 랜딩되지 않는다. 각 단계가 기계적으로 강제된다.

5. **Claw 에이전트들은 이슈 트래커 없이, Symphony는 에이전트 프레임워크 없이 작동한다.** 두 생태계의 결합이 자율 소프트웨어 개발 파이프라인의 완성에 필요한 조각들을 채울 수 있다.

---

## 2. Symphony란 무엇인가 — 포지셔닝

```
┌─────────────────────────────────────────────────────────────────┐
│            운영 자동화 계층 (새 범주)                              │
│  Symphony — 이슈 트래커 → 에이전트 디스패치 → PR 랜딩 데몬         │
└───────────────────────────┬─────────────────────────────────────┘
                            │ 조율/실행
┌───────────────────────────▼─────────────────────────────────────┐
│                응용 계층 (repos_applied/)                         │
│  ClawWork (벤치마크)  ClawPort (대시보드)  Moltbook (소셜)         │
└───────────────────────────┬─────────────────────────────────────┘
                            │ 사용/확장
┌───────────────────────────▼─────────────────────────────────────┐
│              프레임워크 계층 (repos/)                              │
│  Nanobot  OpenClaw  NanoClaw  IronClaw  ZeroClaw  PicoClaw …    │
└─────────────────────────────────────────────────────────────────┘
```

Symphony는 `SPEC.md:29`에서 자신을 "a scheduler/runner and tracker reader"로 정의한다. 결정적 근거:

- `SPEC.md:53-55`: "Ticket writes…are typically performed by the coding agent." — Symphony 자체는 티켓을 쓰지 않는다. 비즈니스 로직을 에이전트에 위임하는 비침습 원칙.
- `orchestrator.ex:1-41`: Elixir GenServer로 폴링 루프, 동시성 제한, 재시도 큐를 in-memory 상태로 관리하는 스케줄러.
- `WORKFLOW.md:29-33`: `max_concurrent_agents: 10`, `max_turns: 20` — 프레임워크가 아닌 운영 파라미터.
- `SPEC.md:1`: "Status: Draft v1 (language-agnostic)" — 스펙과 구현의 명시적 분리.

| 계층 | 역할 | Symphony의 위치 |
|------|------|----------------|
| repos/ (프레임워크) | 에이전트 런타임 정의 | 해당 없음 |
| repos_applied/ (응용) | 특정 문제 해결 | 해당 없음 |
| **Symphony (운영 자동화)** | 에이전트 풀 스케줄링 | 여기 |

---

## 3. 아키텍처 분석

### 3.1 Tech Stack

**언어/런타임**: Elixir `~> 1.19` (OTP 28), `mise` 관리, escript로 배포 (`bin/symphony`, `mix.exs:8,93-96`)

**핵심 의존성** (`mix.exs:64-80`):

| 패키지 | 역할 |
|--------|------|
| `bandit ~> 1.8` | HTTP 서버 (Cowboy 대체) |
| `phoenix_live_view ~> 1.1.0` | 실시간 웹 대시보드 |
| `req ~> 0.5` | Linear GraphQL HTTP 클라이언트 |
| `solid ~> 1.2` | Liquid 템플릿 (프롬프트 렌더링) |
| `yaml_elixir ~> 2.12` | WORKFLOW.md 프론트매터 파싱 |
| `nimble_options ~> 1.1` | 스키마 검증 설정 파싱 |
| `dialyxir ~> 1.4` | 정적 타입 분석 (dev) |

**OTP 패턴**: `GenServer` (Orchestrator, WorkflowStore, StatusDashboard), `Task.Supervisor` (에이전트 태스크), `Phoenix.PubSub` (대시보드 리얼타임 업데이트), `Port` (Codex 서브프로세스 stdio 통신)

### 3.2 핵심 모듈 구조

```
SymphonyElixir.Application [symphony_elixir.ex:15]
  Supervisor (:one_for_one)
  ├── Phoenix.PubSub
  ├── Task.Supervisor (SymphonyElixir.TaskSupervisor)
  ├── WorkflowStore [GenServer] — WORKFLOW.md 핫리로드, 1초 폴링
  ├── Orchestrator [GenServer] — 에이전트 스케줄링 핵심
  ├── HttpServer [선택적] — Phoenix/Bandit LiveView + JSON API
  └── StatusDashboard [GenServer] — ANSI TUI 렌더러, 16ms 간격
```

**핵심 모듈별 책임** (file:line 참조):

| 모듈 | 파일 | 책임 |
|------|------|------|
| `WorkflowStore` | `workflow_store.ex:1` | WORKFLOW.md 파싱 + 캐시, `{mtime,size,phash2}` 변경 감지 |
| `Workflow` | `workflow.ex:1` | YAML 파싱 + Liquid 템플릿 분리, 순수 함수 |
| `Config` | `config.ex:1` | NimbleOptions 스키마 검증, `$VAR` 환경변수 간접 참조, ~30개 타입 게터 |
| `Orchestrator` | `orchestrator.ex:1` | 5초 폴링, 상태 기계, 태스크 디스패치, 모니터링 |
| `AgentRunner` | `agent_runner.ex:1` | 워크스페이스 생성, AppServer 세션, 턴 루프 |
| `Codex.AppServer` | `codex/app_server.ex:1` | JSON-RPC 2.0 over stdio Port, 스트리밍 파싱 |
| `Codex.DynamicTool` | `codex/dynamic_tool.ex:1` | `linear_graphql` 도구 주입 |
| `Linear.Adapter` | `linear/adapter.ex:1` | GraphQL CRUD: 이슈 페치, 상태 전환, 댓글 |
| `Linear.Client` | `linear/client.ex:1` | Req HTTP 클라이언트, cursor 페이징 (50건/페이지) |
| `Workspace` | `workspace.ex:1` | 경로 안전 검증, 심볼릭 링크 탈출 방지, 훅 실행 |
| `PromptBuilder` | `prompt_builder.ex:1` | Solid Liquid strict 렌더링, Issue 구조체 → 템플릿 컨텍스트 |
| `SpecsCheck` | `specs_check.ex:1` | Elixir AST 파서, 공개 함수 `@spec` 강제 검사기 |
| `StatusDashboard` | `status_dashboard.ex:1` | ANSI TUI, sparkline 그래프 (10분 윈도우), PubSub 스냅샷 |

### 3.3 데이터 플로우

```
1. 시작
   CLI.main → set workflow path → Application.ensure_all_started
   WorkflowStore → WORKFLOW.md 로드 → YAML 파싱
   Config.validate! → tracker_kind/token/project/codex_command 필수 확인

2. 폴링 사이클 (기본 30초)
   :tick → :run_poll_cycle
   Tracker.fetch_candidate_issues()
     → Linear GraphQL (project_slug + active_states 필터)
     → [%Linear.Issue{id, identifier, title, description, state, branch_name, blocked_by}]
   choose_issues(issues, state)
     → 미실행 + 미클레임 + 미완료 + 슬롯 여유 + 상태 슬롯 여유 필터
     → 정렬: priority → created_at → identifier
   dispatch_issue(state, issue)
     → claimed + running에 추가
     → Task.Supervisor.start_child(AgentRunner.run)
     → Process.monitor(task.pid)

3. 에이전트 실행 (이슈당 독립 Task)
   AgentRunner.run(issue, orchestrator_pid, opts)
   Workspace.create_for_issue(issue)
     → {workspace_root}/{sanitized_identifier}
     → 심볼릭 링크 탈출 검증, 빌드 아티팩트 정리
   AppServer.start_session(workspace)
     → Port.open("bash", ["-lc", codex_command], cwd: workspace)
     → JSON-RPC: initialize → thread/start (with approval_policy, thread_sandbox)
   loop (turn 1..max_turns, 기본 20):
     PromptBuilder.build_prompt(issue, attempt: n)
       → 1번 턴: 전체 WORKFLOW.md Liquid 렌더링
       → 2번+ 턴: 짧은 continuation guidance
     AppServer.run_turn(session, prompt, issue)
       → JSON-RPC turn/start → 스트리밍 수신
       → DynamicTool.execute("linear_graphql", args) (동기 처리)
       → {:codex_worker_update, issue_id, msg} → Orchestrator
     Tracker.fetch_issue_states_by_ids([issue.id])
       → 활성 상태 → 다음 턴 계속
       → 종료 상태 → 루프 중단

4. 완료/실패
   {:DOWN, ref, :process, pid, :normal} → continuation retry (1초 후)
   {:DOWN, ref, :process, pid, reason} → 지수 백오프 재시도
   reconcile 루프 → 종료 상태 감지 → 워커 종료 + 워크스페이스 정리
```

---

## 4. 에이전트 오케스트레이션

### 4.1 에이전트 생명주기

**스폰** (`orchestrator.ex:598-647`):
```
Task.Supervisor.start_child(TaskSupervisor, fn → AgentRunner.run(issue, ...) end)
Process.monitor(task.pid)
claimed MapSet + running Map에 추가
```

**작업 실행** (`agent_runner.ex:11-33`):
1. `Workspace.create_for_issue/1` — 이슈별 디렉터리 생성/재사용
2. `Workspace.run_before_run_hook/2` — `before_run` 셸 훅 실행
3. `run_codex_turns/4` — AppServer 세션 + 턴 루프 (최대 `max_turns`회)
4. `after` 블록에서 `Workspace.run_after_run_hook/2` 항상 실행 (보장)

**완료 (정상)**: `{:DOWN, ref, :process, _pid, :normal}` → continuation retry 예약 (1초, `@continuation_retry_delay_ms`, `orchestrator.ex:111`)

**완료 (비정상)**: 지수 백오프 재시도 → `delay = min(10_000 × 2^(attempt-1), max_retry_backoff_ms)` (`orchestrator.ex:829-832`)

### 4.2 스케줄링 알고리즘

**`should_dispatch_issue?/4`의 통과 조건** (`orchestrator.ex:473-487`) — 모두 참이어야 함:
- 유효한 이슈 필드 + 활성 상태 + `issue_routable_to_worker?` (담당자 필터)
- todo 상태면 모든 blocker가 terminal 상태여야 함
- claimed/running/completed에 없음
- 글로벌 슬롯 여유 (`max_concurrent_agents - running_count > 0`)
- 상태별 슬롯 여유 (`max_concurrent_agents_by_state`)

**디스패치 전 재검증** (`orchestrator.ex:578-596`): 스폰 직전 Linear API를 다시 호출하여 폴 데이터 staleness 방지.

**정렬**: `{priority_rank, created_at_us, identifier}` — 우선순위 1..4 우선, 동점이면 오래된 이슈 먼저 (`orchestrator.ex:453-471`).

**동시성 제어**:
- 글로벌: `max_concurrent_agents` (기본 10)
- 상태별: `max_concurrent_agents_by_state` 맵 (예: `{"Merging": 1}`)

### 4.3 Codex 통합 (AppServer + DynamicTool)

**AppServer 프로토콜** (`codex/app_server.ex`): Erlang Port로 `bash -lc <codex_command>` 프로세스 실행, JSON-RPC 2.0 over stdio.

```
initialize (id=1) → initialized
thread/start (id=2) → thread 생성
  params: approvalPolicy, sandbox, cwd, dynamicTools
turn/start (id=3) → 스트리밍 수신
  params: threadId, input, turn_sandbox_policy, title
```

**타임아웃 계층** (`app_server.ex`):
- `read_timeout_ms` (5초): 핸드셰이크 응답 대기
- `turn_timeout_ms` (1시간): 전체 턴 완료 대기
- `stall_timeout_ms` (5분): Orchestrator 재조정이 감시, 출력 없는 교착 상태 감지

**DynamicTool** (`codex/dynamic_tool.ex`): `linear_graphql` 도구를 `thread/start`에 주입. Codex가 `item/tool/call`을 보내면, `receive_loop`에서 동기적으로 실행 후 결과를 Port로 즉시 반환. 에이전트가 Symphony 인증 자격증명으로 Linear API를 직접 호출 가능.

**자동 승인**: `approval_policy: never` 설정 시 모든 명령/패치 승인 요청을 자동으로 수락 (`app_server.ex:52`).

### 4.4 실패 처리와 재시도

| 실패 유형 | 감지 | 처리 |
|-----------|------|------|
| 비정상 종료 | `{:DOWN, ref, ..., reason}` | 지수 백오프 재시도 |
| 교착 (stall) | `elapsed > stall_timeout_ms` | 워커 종료 + 재시도 (`orchestrator.ex:367-424`) |
| 이슈 상태 종료 | reconcile 루프 배치 조회 | 워커 종료 + 워크스페이스 정리 (`orchestrator.ex:236-323`) |
| 스폰 실패 | `{:error, reason}` | 로그 + 재시도 (`orchestrator.ex:638-646`) |
| WorkflowStore 오류 | 파싱 실패 | 마지막 정상 워크플로 유지, 디스패치 차단 |

**워크스페이스 훅 실패 시맨틱**:
- `after_create` 실패 → 워크스페이스 생성 중단 → 재시도
- `before_run` 실패 → 실행 중단 → 재시도
- `after_run` / `before_remove` 실패 → 로그만, 무시 (`agent_runner.ex:16-27`)

---

## 5. CI/PR/Linear 통합

### 5.1 Codex Skill 시스템

Symphony는 `.codex/skills/` 아래 6개 스킬을 제공한다. 각각 YAML 프론트매터 + 마크다운 절차 본문으로 구성된다:

| 스킬 | 용도 | 핵심 특징 |
|------|------|-----------|
| `commit` | 커밋 생성 | `Co-authored-by: Codex <codex@openai.com>` 트레일러 강제 (`commit/SKILL.md:43`) |
| `debug` | 교착/실패 디버깅 | 스택 진단 절차 |
| `land` | PR 랜딩 | `land_watch.py` 비동기 감시 위임, 14점 리뷰 처리 프로토콜 (`land/SKILL.md:51-54`) |
| `linear` | Linear GraphQL 직접 조작 | `attachmentLinkGitHubPR` mutation 선호 (`linear/SKILL.md:264-280`) |
| `pull` | origin/main 동기화 | `rerere.enabled` + `merge.conflictstyle=zdiff3` 설정 (`pull/SKILL.md:16-29`) |
| `push` | 브랜치 push + PR 생성 | `make -C elixir all` 선행 필수 (`push/SKILL.md:29,65`) |

**스킬 합성**: `land`가 `commit`, `push`, `pull`을 서브스킬로 명시 호출 (`land/SKILL.md:36-42`). 에이전트는 git 명령을 직접 구성하지 않고, 검증된 절차 스킬을 호출한다.

### 5.2 Linear 이슈 생명주기

**이슈 구조** (`linear/issue.ex:6-21`):
```elixir
%Issue{
  id: "UUID",          # Linear 내부 ID
  identifier: "MT-686",# 인간이 읽는 키
  state: "In Progress",
  branch_name: "...",  # Linear이 미리 생성, 즉시 git branch로 사용
  blocked_by: [%{id, identifier, state}],
  assigned_to_worker: boolean
}
```

**폴링** (`linear/client.ex:107-122`): GraphQL cursor 페이징 (50건/페이지), `inverseRelations`로 blocker 이슈 포함. 담당자 필터: `"me"` → `SymphonyLinearViewer` 쿼리로 동적 해석 (`client.ex:457-474`).

**상태 전환** (`linear/adapter.ex:61-74`):
1. `resolve_state_id/2` — 상태 이름 → UUID 변환 (GraphQL 조회, 하드코딩 방지)
2. `SymphonyUpdateIssueState` mutation

**PR 연결**: `attachmentLinkGitHubPR` mutation으로 GitHub PR을 Linear 이슈에 typed attachment로 연결 (plain URL이 아님, `linear/SKILL.md:264-280`).

### 5.3 Proof-of-Work 계층

Symphony는 에이전트 작업의 품질을 5단계 검증 체인으로 강제한다:

**Layer 1 — 로컬 게이트** (`push/SKILL.md:29,65`):
```sh
make -C elixir all  # push 전 필수 실행
```

**Layer 2 — CI 게이트** (`.github/workflows/make-all.yml`):
- `mise` + 캐시 → `make all` (format + compile + dialyzer + tests + specs.check)
- 모든 PR과 main push에 적용

**Layer 3 — PR 본문 린트** (`.github/workflows/pr-description-lint.yml`):
PR 본문을 `mix pr_body.check`로 검사 (`pr_body.check.ex`):
- 모든 템플릿 헤딩 존재 및 순서 확인 (`lines 109-121`)
- `<!-- ... -->` 플레이스홀더 제거 확인 (`lines 123-128`)
- 빈 섹션 금지 (`lines 140-141`)
- 불릿/체크박스 템플릿 섹션은 최소 1개 이상 필수 (`lines 151-168`)

**Layer 4 — Codex AI 리뷰** (`land_watch.py`):
`## Codex Review — <persona>` 댓글 감지 → `[codex]` 답장 미확인 시 종료 코드 2 반환 → 에이전트가 리뷰 처리 (`land_watch.py:330-347`).

**Layer 5 — Async 감시자** (`land_watch.py`): 3개 비동기 태스크 병렬 실행:
- `wait_for_checks`: CI 체크 폴링 (10초 간격), 실패 시 코드 3
- `wait_for_codex`: 리뷰 댓글 폴링, 미승인 시 코드 2
- `head_monitor`: PR HEAD SHA 변경 감지 (CI autofix), 코드 4

출구 코드가 에이전트 지시다: 2=리뷰 처리, 3=CI 수정, 4=pull+push, 5=충돌 해결.

### 5.4 Land/Push/Pull 워크플로

```
push skill
  1. make -C elixir all (로컬 게이트)
  2. git push -u origin HEAD
  3. non-fast-forward → pull skill → re-validate → push
  4. gh pr create / gh pr edit (PR 본문 템플릿 작성)
  5. mix pr_body.check (로컬 린트)
  ↓
land skill
  1. 클린 working tree 확인
  2. gh pr view --json mergeable → CONFLICTING → pull → push
  3. python3 land_watch.py (비동기 감시 시작)
  4. exit 2 → 리뷰 처리 (per-comment 14점 프로토콜)
  5. exit 3 → CI 수정 → commit + push → watcher 재시작
  6. exit 4 → pull/amend/force-push
  7. 모두 통과 → gh pr merge --squash
```

**`[codex]` 프리픽스**: 에이전트가 생성하는 모든 댓글은 `[codex]`로 시작. `land_watch.py`가 이를 감지하여 리뷰 승인 상태를 추적 (`land_watch.py:290-303`).

### 5.5 Specs Check 시스템

`SymphonyElixir.SpecsCheck` (`specs_check.ex`): Elixir AST 파서로 `lib/` 내 모든 공개 함수의 `@spec` 선언을 강제하는 정적 분석기.

**알고리즘** (`missing_public_specs/2`):
1. `.ex` 파일 수집 → `Code.string_to_quoted/2` (AST)
2. `Macro.prewalk`로 `defmodule` 노드 탐색
3. 상태 기계 (`consume_form/5`): `@spec` 누적 → `@impl` 표시 → `def` 검사 → 미준수 기록

**준수 조건** (`compliant?/3`, `specs_check.ex:140-146`):
- 인접 `@spec` 있음, 또는
- `@impl` 표시 (콜백), 또는
- 면제 파일에 명시

`make all` 포함 → CI에서 강제 — 에이전트가 추가한 모든 공개 함수는 `@spec`을 가져야 한다.

---

## 6. Claw 생태계와의 비교

### 6.1 공유 패턴

#### SKILL.md 컨벤션의 수렴 — 가장 강력한 증거

| 구현체 | 형식 | 호출 방식 |
|--------|------|-----------|
| NanoClaw (원조) | 헤더 없음, 순수 마크다운 | 컨테이너 IPC |
| ClawWork | `always: true` 헤더 | 시스템 프롬프트 상시 주입 |
| Symphony (공식화) | YAML 프론트매터 (`name`, `description`) | WORKFLOW.md에서 명시적 참조 |

Symphony의 SKILL.md는 단순 절차 문서를 넘어 실행 가능한 자동화 플레이북이다 (`land/SKILL.md`). **진화 방향**: 비형식 → 주입 메타데이터 → 공식 YAML + 교차 참조 + 외부 스크립트.

#### WORKFLOW.md — SKILL.md의 오케스트레이터 버전

Symphony의 WORKFLOW.md는 "에이전트 정책을 코드와 함께 버전 관리"하는 NanoClaw SKILL.md 철학의 상위 확장이다. 단일 파일에 런타임 설정(YAML)과 에이전트 프롬프트(Liquid)가 공존하며, 1초 폴링으로 재시작 없이 핫리로드된다.

#### 비침습 위임 원칙

Symphony `SPEC.md:53-55`의 비즈니스 로직 위임 원칙은 ClawWork의 7가지 비침습 확장 기법과 동일한 철학이다: 핵심 로직을 건드리지 않고 레이어를 삽입한다. `tracker.ex:7-12`의 `@callback` 기반 어댑터 패턴은 IronClaw의 Rust trait, Nanobot의 Python ABC와 동일한 추상화다.

#### 공통 인프라 패턴

| 패턴 | Symphony | Claw 대응체 |
|------|----------|-----------|
| 어댑터 behaviour | `Tracker` + `Linear.Adapter` | IronClaw trait, Nanobot ABC |
| 훅 시스템 | 4개 워크스페이스 훅 | OpenClaw 24개 플러그인 훅 |
| 폴링 루프 | GenServer `:tick` | PicoClaw goroutine |
| 이슈별 격리 | 워크스페이스 디렉터리 | NanoClaw Docker 컨테이너 |
| 지수 백오프 재시도 | `10_000 × 2^(attempt-1)` | ZeroClaw 재시도 메커니즘 |
| 구조화 로그 | `:logger_disk_log_h` | ZeroClaw structured logging |

### 6.2 핵심 차이점

**Symphony에 있고 Claw에 없는 것:**

| 기능 | Symphony | Claw 상태 |
|------|----------|-----------|
| 이슈 트래커 자동 디스패치 | Linear GraphQL 폴링 + 에이전트 스폰 | 전무 (ClawWork: BLS 데이터셋만) |
| 동시 에이전트 풀 스케줄링 | 10개 동시, 상태별 한도 | 전무 |
| 공식 언어 독립 스펙 | `SPEC.md` (Draft v1) | 전무 |
| PR 랜딩 자동화 플레이북 | `land/SKILL.md` + `land_watch.py` | 전무 |
| 상태 기반 오케스트레이터 | `running/claimed/completed/retry_attempts` | 전무 |
| 워크스페이스 심볼릭 링크 방어 | `workspace.ex:230-255` | Docker/WASM으로만 |
| `@spec` 강제 정적 분석 | `specs_check.ex` | 전무 |

**Claw에 있고 Symphony에 없는 것:**

| 기능 | Claw 구현체 |
|------|-----------|
| 벡터/하이브리드 메모리 검색 | OpenClaw, IronClaw |
| 브라우저 자동화 | OpenClaw, ZeroClaw, NanoClaw |
| 에이전트 비용 추적/잔액 관리 | ClawWork |
| 에이전트 신원 (SOUL) | OpenClaw |
| MCP 서버 직접 관리 | 5개 프레임워크 |
| 멀티모달(비전) | OpenClaw, NanoClaw |
| 에이전트 소셜 플랫폼 | Moltbook |

### 6.3 Symphony가 채우는 공백

**공백 1: 이슈 트래커 → 에이전트 자동 디스패치**
현재 Claw 에이전트를 실제 프로젝트에 투입하려면 인간이 수동으로 지시해야 한다. Symphony의 폴링 루프는 이 인간 개입을 제거한다. Linear 이슈가 활성 상태로 진입하는 순간 에이전트가 자동 스폰된다.

**공백 2: 에이전트 턴 지속 프로토콜**
`agent_runner.ex:74-100`의 `continue_with_issue?` 패턴:
```elixir
case continue_with_issue?(issue, issue_state_fetcher) do
  {:continue, refreshed_issue} when turn_number < max_turns → 다음 턴
  {:continue, refreshed_issue} → Orchestrator에 반환 (max_turns 도달)
  {:done, _} → 완료
end
```
이슈 상태를 매 턴마다 실시간 확인하여 계속/중단을 결정. Claw 어느 프레임워크에도 없는 패턴.

**공백 3: PR 랜딩 감시 자동화**
`land_watch.py`의 3개 비동기 태스크(CI 체크, Codex 리뷰, HEAD 변경)가 PR 랜딩의 전체 생명주기를 감시한다. Claw 에이전트들은 PR을 생성할 수 있지만, 랜딩까지 자동 완주하는 플레이북이 없다.

**공백 4: 상태별 동시성 한도**
`max_concurrent_agents_by_state`로 "Merging 상태는 동시 1개만" 같은 세밀한 리소스 제어가 가능하다. 이는 CI 서버 포화 방지에 실용적이다.

---

## 7. 결론 및 열린 질문

### 핵심 결론

1. **Claw 생태계에 3번째 레이어가 존재한다.** repos/(프레임워크) + repos_applied/(응용)에 더해, 이슈 트래커와 에이전트 풀을 연결하는 "운영 자동화 계층"이 별도로 필요하고, Symphony가 그것의 구체적 구현체다.

2. **SKILL.md는 생태계를 초월한 de facto 인터페이스 컨벤션이 됐다.** NanoClaw(원조) → ClawWork(주입) → Moltbook API(`skill.md` URL) → Symphony(YAML 공식화)의 진화 경로가 확인된다. OpenAI가 독립적으로 동일한 컨벤션을 채택했다는 사실이 이 패턴의 수렴 압력을 입증한다.

3. **Symphony의 5단계 Proof-of-Work 체인이 자율 에이전트의 품질 제어 레퍼런스다.** 로컬 게이트 → CI → PR 본문 린트 → AI 리뷰 → squash-merge. 각 단계가 기계적으로 강제되며, 인간의 수동 확인 없이 코드가 main에 랜딩된다. 이것이 "24시간 자율 에이전트"의 실용적 안전 메커니즘이다.

4. **Codex `app-server` 프로토콜(JSON-RPC over stdio)이 에이전트 런너 표준이 될 수 있다.** Symphony가 Codex를 서브프로세스로 제어하는 방식은 Claw 프레임워크들이 이 프로토콜을 구현하면 Symphony의 스케줄링 레이어와 결합 가능함을 시사한다 (`SPEC.md:133`).

5. **워크스페이스 파일이 에피소딕 메모리를 대체한다.** Symphony는 벡터 DB나 SQL 메모리 없이, 이슈별 격리 디렉터리의 파일 시스템 자체가 에이전트의 컨텍스트 저장소다. 장기 메모리를 의도적으로 설계에서 배제했다.

### 열린 질문

**Q22. WORKFLOW.md가 SKILL.md보다 상위 컨벤션인가, 아니면 별도 진화인가?**
두 형식 모두 "YAML 헤더 + 마크다운 본문 = 에이전트 정책 파일" 구조를 공유한다. 이 패턴이 단일 컨벤션으로 수렴하고 있는가? Moltbook API의 `skill.md` URL 반환까지 합치면 이 패턴의 확산이 생태계 전체 표준 수립 과정인가?

**Q23. Claw 프레임워크가 Codex `app-server` 프로토콜을 구현하면 Symphony와 결합 가능한가?**
`SPEC.md:133`은 "JSON-RPC-like app-server mode over stdio를 지원하는 에이전트"를 요구한다. OpenClaw, Nanobot이 이 인터페이스를 구현하면 Symphony의 스케줄링 레이어를 Claw 에이전트 풀에 적용할 수 있다. 생태계 간 결합의 현실적 경로인가?

**Q24. 단일 오케스트레이터의 단일 실패 지점 문제**
`SPEC.md:85`는 "단일 권위 있는 오케스트레이터"를 요구한다. GenServer 하나가 in-memory 상태를 전담하면, 프로세스 충돌 시 실행 중 에이전트 상태가 소실된다. `SPEC.md:47`은 "재시작 복구"를 요구하지만, 실행 중 에이전트는 어떻게 되는가? ClawPort의 "Zero Own Key" 단일 게이트웨이 문제(Q21)와 동일한 구조적 취약점이다.

**Q25. Linear 이슈 상태가 AI 자율성의 경계선이 될 수 있는가?**
Symphony의 `WORKFLOW.md`에서 `Human Review` 상태는 명시적 에이전트 개입 중단점이다. 이 "트래커 상태 = 인간 개입 게이트" 패턴은 24시간 자율 에이전트를 운영하는 Claw 생태계에서 안전 장치로 도입 가능한가? 현재 Claw 에이전트들에는 이런 명시적 중단점이 없다.

**Q26. WORKFLOW.md 동적 리로드와 실행 중 에이전트의 충돌**
`SPEC.md:506-523`은 "in-flight 에이전트 세션은 자동 재시작 불필요"라고 명시하지만, `max_concurrent_agents`를 10→5로 낮추면 이미 실행 중인 10개 워커는 어떻게 되는가? 설정과 실제 상태의 일시적 불일치가 발생한다. ZeroClaw의 Soul Snapshot (Git 기반 상태 복원)보다 운영상 위험한가?

**Q27. Symphony + Claw 프레임워크의 완전한 자율 개발 파이프라인 가능성**
Symphony(이슈 → 에이전트 스폰 → PR 랜딩) + OpenClaw(브라우저 자동화 + 하이브리드 메모리) + ClawWork(품질 평가) + ClawPort(팀 관찰)의 조합이 인간 개입 없는 소프트웨어 개발 파이프라인을 구성할 수 있는가? 각 레이어가 담당하는 역할이 상호 보완적이다. 실용적 결합의 기술적 장벽은 무엇인가?
