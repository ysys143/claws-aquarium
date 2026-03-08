# Symphony + Claw + ClawPort 통합 구체 계획

> 작성일: 2026-03-08
> 기반 보고서: `symphony_integration_report.md`, `symphony_report.md`
> 목표: Symphony 스케줄링 + Claw 에이전트 실행 + ClawPort 관찰을 단일 자율 개발 파이프라인으로 결합

---

## 목차

1. [전체 아키텍처](#1-전체-아키텍처)
2. [핵심 기술 결정](#2-핵심-기술-결정)
3. [Phase 1: 기반 결합 (3개월)](#3-phase-1-기반-결합-3개월)
4. [Phase 2: 스펙 준수 통합 (6개월)](#4-phase-2-스펙-준수-통합-6개월)
5. [Phase 3: 단일 제어 평면 (12개월)](#5-phase-3-단일-제어-평면-12개월)
6. [핵심 기술 장벽 및 해결 방안](#6-핵심-기술-장벽-및-해결-방안)
7. [Go/No-Go 체크리스트](#7-gonogo-체크리스트)

---

## 1. 전체 아키텍처

### 의존성 구조 (단방향, 순환 없음)

```
Linear 이슈 (또는 ClawPort 이슈)
    | (폴링)
    v
Symphony Orchestrator (스케줄러/디스패처)
    | (HTTP API Phase 1 -> JSON-RPC stdio Phase 2)
    v
Claw Framework (Nanobot / OpenClaw 등)
    | (에이전트 실행 + 상태 콜백)
    v
ClawPort (관찰/추적 + 비용 계산 + Tracker 소스)
```

### 두 가지 배포 모드

**Mode A — Phase 1-2 (중간 단계)**
Linear가 이슈 소스, ClawPort는 읽기 전용 관찰

```
+-------+     +----------+     +---+     +--------+
|Linear |---->|Symphony  |---->|Claw|---->|ClawPort|
|Tracker| poll|Orchestr. | HTTP|Frame|POST|(Observe)|
+-------+     +----------+     +---+     +--------+
```

**Mode B — Phase 3 (최종 목표)**
ClawPort가 이슈 소스 및 추적, Linear 독립

```
+--------------------------------------------------+
| ClawPort (Source + Observer)                    |
| +----------------------------------------------+|
| | Issues Table (Linear 대체)                  ||
| | Episodes Table (에이전트 실행 기록)         ||
| | Token/Cost Ledger (비용 추적)               ||
| +----------------------------------------------+|
+--------------------------------------------------+
      | (Tracker API)        | (HTTP API)
    +-+---------+----------+-+
    |Symphony Orchestrator |
    |(poll -> dispatch)    |
    +-+------------------+-+
      | (app-server protocol)
    +-+------------------+-+
    | Claw Framework     |
    | (LLM + tools)      |
    +-------------------+
```

### 레이어별 책임

| 레이어 | 컴포넌트 | 역할 |
|--------|----------|------|
| **스케줄링** | Symphony | 이슈 폴링, 에이전트 디스패치, 재시도 정책, 상태 동기화 |
| **실행** | Claw Framework | LLM 호출, 도구 실행, 메모리 관리, 턴 스트리밍 |
| **관찰** | ClawPort Episodes | 에피소드 기록, 턴 기록, 토큰/비용 추적 |
| **이슈 관리** | ClawPort Issues | 이슈 생성/수정/상태 업데이트 (최종) |

---

## 2. 핵심 기술 결정

### 결정 1: 에이전트 메모리 생명주기 (이슈 ID 네임스페이스)

**문제**
Symphony가 workspace를 이슈 완료 시 정리하면, Claw의 장기 메모리(LanceDB, pgvector)와 로컬 MEMORY.md가 소실될 수 있음.

**해결 방안**
각 Claw 에이전트 세션이 `issue_id`를 메모리 네임스페이스로 사용:

- 각 이슈마다 namespace 생성: `namespace: "MT-686"`
- 이슈 재시도 시 동일 namespace에서 이전 컨텍스트 복원
- 이슈 완료 시 namespace는 아카이브 (삭제 안 함)
- ClawPort가 namespace 메타데이터(생성/갱신 시각, 턴 수) 보관

**메모리 격리 구조**

```
.memory/
├── issue_MT-686/
│   ├── MEMORY.md          # 현재 상태
│   ├── turns/
│   │   ├── turn_0001.json # {prompt, response, tool_calls}
│   │   ├── turn_0002.json
│   │   └── summary.json   # {key_decisions, current_state, final_status}
│   └── metadata.json      # {namespace, issue_id, issue_identifier, created_at}
└── issue_MT-687/
```

**영향도**
- Nanobot: `MEMORY.md` 로드/저장 경로 수정 (session_manager.py)
- OpenClaw: LanceDB 스키마에 `issue_id` 파티션 추가
- ZeroClaw: pgvector 쿼리에 `issue_id` 필터 추가

---

### 결정 2: Claw 통합 인터페이스 (HTTP -> app-server 진화)

**Phase 1: HTTP 브릿지**
Claw 프레임워크가 4개 REST 엔드포인트 노출:

```
POST   /sessions              # {issue_id, cwd} -> {session_id}
POST   /turns                 # {session_id, input} -> {turn_id, status}
GET    /turns/{id}            # -> {status, output, token_count}
DELETE /sessions/{id}         # -> 204
```

각 Claw 프레임워크가 200-400 LOC로 구현 (ASGI/Flask/FastAPI 등).

**Phase 2: JSON-RPC 2.0 app-server**
HTTP를 제거하고, Symphony SPEC.md의 app-server 프로토콜을 직접 구현.

```
Protocol:  JSON-RPC 2.0 over stdio
File:line: Symphony SPEC.md:130-160
```

최소 메서드:
```json
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {"capabilities": {}},
  "id": 1
}
```

이벤트 (응답):
```json
{
  "jsonrpc": "2.0",
  "method": "turn/delta",
  "params": {"delta": "..."},
  "id": 1
}
```

---

### 결정 3: ClawPort Tracker 어댑터 API

Symphony의 Tracker 콜백 5개를 ClawPort REST API로 매핑:

| Symphony Tracker 메서드 | ClawPort 엔드포인트 | 파일:라인 |
|------------------------|-------------------|---------:|
| `fetch_candidate_issues()` | `GET /api/issues?state=active` | tracker.ex:45 |
| `fetch_issues_by_states([...])` | `GET /api/issues?states[]=In+Progress` | tracker.ex:52 |
| `fetch_issue_states_by_ids([...])` | `GET /api/issues/batch?ids[]=id1` | tracker.ex:60 |
| `create_comment(issue_id, text)` | `POST /api/issues/{id}/comments` | tracker.ex:68 |
| `update_issue_state(issue_id, state)` | `PATCH /api/issues/{id}/state` | tracker.ex:75 |

각 엔드포인트의 응답 스키마는 Symphony의 기대값과 정확히 일치해야 함 (파일:라인 확인 필수).

---

## 3. Phase 1: 기반 결합 (3개월)

**목표**
Symphony + Nanobot HTTP 브릿지 + ClawPort 관찰 연동 작동 증명

**산출물**
- Nanobot HTTP 게이트웨이 (400 LOC)
- Symphony HttpAgent 통합 (200 LOC)
- ClawPort Episodes API (300 LOC)
- 통합 테스트 시나리오

---

### Task 1-1: Nanobot HTTP 게이트웨이 (2주)

**대상 파일** (Nanobot 레포 기준)

| 파일 | 역할 | LOC |
|-----|------|-----|
| `nanobot_http_gateway.py` (신규) | REST 게이트웨이 (FastAPI 또는 Flask) | 200 |
| `session_manager.py` (신규) | 세션 상태 + 이슈 ID 네임스페이스 | 150 |
| `memory_namespace.py` (신규) | MEMORY.md 로드/저장 (issue_id 기반) | 100 |

**핵심 구현 로직**

```python
# session_manager.py
from pathlib import Path
import json

class SessionManager:
    def start_session(self, issue_id: str, cwd: str) -> str:
        """issue_id를 메모리 네임스페이스로 사용하여 세션 시작."""
        session_id = f"{issue_id}_{uuid4().hex[:8]}"

        # issue_id 기반 메모리 로드 (존재하면 재사용)
        memory_path = Path(cwd) / ".memory" / f"issue_{issue_id}" / "MEMORY.md"
        if memory_path.exists():
            # 이전 컨텍스트 복원
            self.sessions[session_id] = {
                "issue_id": issue_id,
                "memory_path": memory_path,
                "turns": self._load_previous_turns(memory_path)
            }
        else:
            self.sessions[session_id] = {
                "issue_id": issue_id,
                "memory_path": memory_path,
                "turns": []
            }

        return session_id

    def run_turn(self, session_id: str, input_text: str) -> dict:
        """이슈 네임스페이스 내에서 턴 실행."""
        session = self.sessions[session_id]

        # Nanobot 에이전트 실행
        result = nanobot_agent(
            input=input_text,
            memory_path=session["memory_path"],
            cwd=session["cwd"]
        )

        # 턴 메타데이터 기록
        session["turns"].append({
            "turn_id": len(session["turns"]) + 1,
            "input": input_text,
            "output": result["output"],
            "tokens": result["tokens"]
        })

        return {
            "status": "completed",
            "output": result["output"],
            "token_count": result["tokens"],
            "turns_so_far": len(session["turns"])
        }
```

**REST 엔드포인트**

```python
# nanobot_http_gateway.py
from fastapi import FastAPI
from session_manager import SessionManager

app = FastAPI()
sm = SessionManager()

@app.post("/sessions")
async def start_session(issue_id: str, cwd: str):
    """POST /sessions?issue_id=MT-686&cwd=/tmp/workspace"""
    session_id = sm.start_session(issue_id, cwd)
    return {"session_id": session_id}

@app.post("/turns")
async def run_turn(session_id: str, input: str):
    """POST /turns?session_id=MT-686_abc123&input=...turnprompt..."""
    result = sm.run_turn(session_id, input)
    return result

@app.get("/turns/{turn_id}")
async def get_turn_status(turn_id: str):
    """GET /turns/MT-686_abc123"""
    result = sm.get_turn_status(turn_id)
    return result

@app.delete("/sessions/{session_id}")
async def close_session(session_id: str):
    """DELETE /sessions/MT-686_abc123"""
    sm.close_session(session_id)
    return {"status": "closed"}
```

**검증 기준**

```bash
# 세션 시작
curl -X POST http://localhost:8000/sessions \
  -d '{"issue_id":"MT-686","cwd":"/tmp/workspace"}' \
  -H 'Content-Type: application/json'
# -> {"session_id":"MT-686_abc123"}

# 턴 실행
curl -X POST http://localhost:8000/turns \
  -d '{"session_id":"MT-686_abc123","input":"Implement a counter"}' \
  -H 'Content-Type: application/json'
# -> {"status":"completed","output":"...","token_count":245}
```

---

### Task 1-2: Symphony HttpAgent 어댑터 (2주)

**대상 파일** (Symphony 레포 기준)

| 파일 | 역할 | LOC |
|-----|------|-----|
| `lib/symphony_elixir/http_agent.ex` (신규) | HTTP 클라이언트 래퍼 | 150 |
| `lib/symphony_elixir/agent_runner.ex` (수정) | AppServer -> HttpAgent 전환 | +80 |
| `config/config.exs` (수정) | HTTP 게이트웨이 URL 설정 | +5 |

**구현 (Elixir)**

```elixir
# lib/symphony_elixir/http_agent.ex
defmodule SymphonyElixir.HttpAgent do
  require Logger

  @spec start_session(String.t(), String.t()) ::
    {:ok, String.t()} | {:error, term()}
  def start_session(api_url, workspace) do
    issue_id = Path.basename(workspace)

    case Req.post(
      api_url <> "/sessions",
      json: %{
        issue_id: issue_id,
        cwd: workspace
      }
    ) do
      {:ok, response} ->
        {:ok, response.body["session_id"]}

      {:error, reason} ->
        Logger.error("Failed to start session: #{inspect(reason)}")
        {:error, reason}
    end
  end

  @spec run_turn(String.t(), String.t(), String.t()) ::
    {:ok, map()} | {:error, term()}
  def run_turn(api_url, session_id, prompt) do
    case Req.post(
      api_url <> "/turns",
      json: %{
        session_id: session_id,
        input: prompt
      }
    ) do
      {:ok, response} ->
        body = response.body
        {:ok, %{
          status: body["status"],
          output: body["output"],
          token_count: body["token_count"]
        }}

      {:error, reason} ->
        {:error, reason}
    end
  end

  @spec close_session(String.t(), String.t()) :: :ok | {:error, term()}
  def close_session(api_url, session_id) do
    case Req.delete(api_url <> "/sessions/#{session_id}") do
      {:ok, _} -> :ok
      {:error, reason} -> {:error, reason}
    end
  end
end
```

**agent_runner.ex 수정 (개요)**

```elixir
# lib/symphony_elixir/agent_runner.ex
defmodule SymphonyElixir.AgentRunner do
  # 기존: run_codex_turns/4 (AppServer 호출)
  # 신규: run_http_turns/4 (HttpAgent 호출)

  def run_http_turns(issue, workspace, turn_count, context) do
    api_url = Application.get_env(:symphony_elixir, :http_agent_url)

    {:ok, session_id} = HttpAgent.start_session(api_url, workspace)

    for turn <- 1..turn_count do
      prompt = make_prompt(issue, context, turn)
      {:ok, result} = HttpAgent.run_turn(api_url, session_id, prompt)

      # ClawPort 콜백 (Task 1-3에서 정의)
      ClawPort.record_turn(
        session_id,
        turn,
        result[:token_count]
      )
    end

    HttpAgent.close_session(api_url, session_id)
  end
end
```

**config.exs 추가**

```elixir
# config/config.exs
config :symphony_elixir, http_agent_url: "http://localhost:8000"
```

**검증 기준**

1. Symphony가 Linear 이슈 폴링
2. Nanobot HTTP 게이트웨이로 에이전트 스폰
3. 완료 후 session 정리 (DELETE 호출)
4. 에러 발생 시 재시도 로직 작동

---

### Task 1-3: ClawPort Episodes API (3주)

**대상 파일** (ClawPort 레포 기준)

| 파일 | 역할 | LOC |
|-----|------|-----|
| `src/models/Episode.ts` (신규) | 에피소드 스키마 | 80 |
| `src/routes/episodes.ts` (신규) | CRUD 엔드포인트 | 200 |
| `src/db/schema.ts` (수정) | 데이터베이스 스키마 추가 | +50 |
| `src/routes/index.ts` (수정) | 라우트 등록 | +5 |

**에피소드 스키마**

```typescript
// src/models/Episode.ts
export interface Episode {
  id: string;                      // UUID
  issue_id: string;                // "MT-686"
  issue_identifier?: string;       // Linear 이슈 키 (선택)
  issue_title?: string;            // "Implement auth flow"
  framework: "nanobot" | "openclaw" | "zeroclaw";

  started_at: Date;
  ended_at: Date | null;
  status: "running" | "completed" | "failed";

  turns: number;                   // 턴 수
  tokens_total: number;            // 총 토큰
  cost_usd: number;                // 예상 비용

  retries: number;                 // 재시도 횟수
  session_id: string;              // Claw 세션 ID

  metadata?: {
    memory_namespace?: string;     // ".memory/issue_MT-686"
    workspace_path?: string;
    final_output?: string;
    error_message?: string;
  };

  created_at: Date;
  updated_at: Date;
}
```

**CRUD 엔드포인트**

```typescript
// src/routes/episodes.ts
import { Router } from "express";
import { Episode } from "../models/Episode";
import { db } from "../db";

const router = Router();

// 에피소드 생성
router.post("/", async (req, res) => {
  const {
    issue_id,
    issue_identifier,
    framework,
    session_id,
    metadata
  } = req.body;

  const episode: Episode = {
    id: crypto.randomUUID(),
    issue_id,
    issue_identifier,
    framework,
    session_id,
    started_at: new Date(),
    ended_at: null,
    status: "running",
    turns: 0,
    tokens_total: 0,
    cost_usd: 0,
    retries: 0,
    metadata,
    created_at: new Date(),
    updated_at: new Date()
  };

  await db.episodes.insert(episode);
  res.json({ episode });
});

// 에피소드 조회
router.get("/:id", async (req, res) => {
  const episode = await db.episodes.findOne({ id: req.params.id });
  res.json({ episode });
});

// 이슈별 에피소드 목록
router.get("/issue/:issue_id", async (req, res) => {
  const episodes = await db.episodes.find({
    issue_id: req.params.issue_id
  });
  res.json({ episodes });
});

// 턴 추가 (호출당 1번, Claw -> ClawPort)
router.post("/:id/turns", async (req, res) => {
  const { tokens, output } = req.body;

  const episode = await db.episodes.findOne({ id: req.params.id });
  episode.turns += 1;
  episode.tokens_total += tokens;
  episode.cost_usd = calculate_cost(episode.tokens_total);
  episode.updated_at = new Date();

  await db.episodes.update(episode);
  res.json({ episode });
});

// 에피소드 완료
router.patch("/:id/complete", async (req, res) => {
  const { status, error_message } = req.body;

  const episode = await db.episodes.findOne({ id: req.params.id });
  episode.status = status; // "completed" | "failed"
  episode.ended_at = new Date();
  if (error_message) episode.metadata.error_message = error_message;

  await db.episodes.update(episode);
  res.json({ episode });
});

export default router;
```

**데이터베이스 마이그레이션**

```sql
-- src/db/migrations/003_create_episodes.sql
CREATE TABLE episodes (
  id VARCHAR(36) PRIMARY KEY,
  issue_id VARCHAR(50) NOT NULL,
  issue_identifier VARCHAR(100),
  framework VARCHAR(20) NOT NULL,
  session_id VARCHAR(50) NOT NULL,
  status VARCHAR(20) DEFAULT 'running',
  started_at TIMESTAMP NOT NULL,
  ended_at TIMESTAMP,
  turns INT DEFAULT 0,
  tokens_total INT DEFAULT 0,
  cost_usd DECIMAL(10,4) DEFAULT 0,
  retries INT DEFAULT 0,
  metadata JSON,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL,

  INDEX (issue_id),
  INDEX (status),
  INDEX (created_at)
);
```

**검증 기준**

```bash
# 에피소드 생성
curl -X POST http://localhost:3000/api/episodes \
  -d '{
    "issue_id":"MT-686",
    "framework":"nanobot",
    "session_id":"MT-686_abc123"
  }' \
  -H 'Content-Type: application/json'
# -> {"episode": {...}}

# 이슈별 에피소드 조회
curl http://localhost:3000/api/episodes/issue/MT-686
# -> {"episodes": [{...}, {...}]}

# ClawPort 대시보드에서 이슈별 에이전트 실행 이력 표시
```

---

### Task 1-4: Phase 1 통합 테스트 (1주)

**시나리오**

1. Linear에 테스트 이슈 5개 생성 (상태: "In Progress")
   - MT-686: "Implement counter component"
   - MT-687: "Add unit tests"
   - MT-688: "Write documentation"
   - MT-689: "Fix edge cases"
   - MT-690: "Performance optimization"

2. Symphony 기동 (config: http_agent_url, tracker: Linear)

3. 5개 이슈가 순차적으로 Nanobot HTTP 게이트웨이로 디스패치

4. 각 이슈 완료 시 ClawPort Episodes 테이블에 기록

5. 1시간 후 결과 검증

**테스트 체크리스트**

```bash
# 1. Linear 이슈 상태 확인
curl -H "Authorization: Bearer $LINEAR_API_KEY" \
  https://api.linear.app/graphql \
  -d '{"query":"query { issues(first: 5) { nodes { id identifier title state } } }"}'
# -> "state": "Done" (5건 모두)

# 2. ClawPort Episodes 레코드 확인
curl http://localhost:3000/api/episodes/issue/MT-686
# -> 1개 에피소드, status: "completed", turns: N, cost_usd: $X.XX

# 3. 메모리 네임스페이스 확인
ls -la .memory/issue_MT-686/
# -> MEMORY.md, turns/ 디렉토리 존재

# 4. 비용 집계 확인
curl http://localhost:3000/api/episodes/issue/MT-686 \
  | jq '.episodes | map(.cost_usd) | add'
# -> $15.47 (이슈별 합계)
```

**Go/No-Go 기준** (Pass 조건)

| 항목 | 기준 | 검증 방법 |
|------|------|----------|
| 이슈 자동 완료 | 5/5 (100%) | Linear API 확인 |
| Episodes 기록 | 5/5 에피소드 | ClawPort DB 쿼리 |
| 메모리 복원 (재시도) | 성공률 >90% | 1개 이슈 강제 재시도, MEMORY.md 비교 |
| 비용 집계 | 이슈별 + 합계 가능 | ClawPort 대시보드 |
| 무인 운영 | 7일 이상 | 로그 검토 (수동 개입 0) |

---

## 4. Phase 2: 스펙 준수 통합 (6개월)

**목표**
Claw 프레임워크가 Symphony SPEC.md의 app-server 프로토콜을 직접 구현. HTTP 브릿지 제거.

**산출물**
- app-server 프로토콜 스펙 (중립화)
- Nanobot app-server 모드 (600 LOC)
- Symphony HttpAgent 제거, AppServer 복귀

---

### Task 2-1: app-server 프로토콜 정리 (2주)

**대상 스펙**
Symphony SPEC.md:130-160 (file:line 확인)

**중립화 목표**
Codex 전용 이벤트 타입을 제거하고 공통 스펙 정의. 모든 Claw 프레임워크가 동일 스펙 구현.

**최소 프로토콜 (JSON-RPC 2.0 over stdio)**

```
요청 흐름:

C->A: initialize(capabilities)
A->C: initialized({})

C->A: thread/start(approvalPolicy, sandbox, cwd, dynamicTools)
A->C: {thread: {id: "T-1"}}

C->A: turn/start(threadId, input, title)
A->C: [streaming events]
    -> {method: "turn/delta", params: {delta: "Assistant is thinking..."}}
    -> {method: "turn/toolUse", params: {toolUseId, toolName, input}}
    -> {method: "turn/toolResult", params: {toolUseId, result}}
    -> {method: "turn/completed", params: {output, stopReason, usage}}

C->A: shutdown()
A->C: {}
```

**메서드 정의**

```typescript
// Protocol: JSON-RPC 2.0

method: "initialize"
params: {
  capabilities: {
    memoryManagement?: boolean;
    dynamicToolRegistration?: boolean;
  }
}
result: {
  capabilities: {
    memoryManagement: boolean;
    dynamicToolRegistration: boolean;
    appServerVersion: string;
  }
}

method: "thread/start"
params: {
  threadId?: string;
  approvalPolicy?: "accept_all" | "manual" | "tool_filter";
  sandbox?: { type: "docker" | "none" };
  cwd: string;
  dynamicTools?: Array<{
    name: string;
    description: string;
    inputSchema: object;
  }>;
}
result: {
  thread: {
    id: string;
    createdAt: string;
  }
}

method: "turn/start"
params: {
  threadId: string;
  input: string;
  title?: string;
}
events: [turn/delta, turn/toolUse, turn/toolResult, turn/completed]

method: "shutdown"
params: {}
result: {}
```

---

### Task 2-2: Nanobot app-server 모드 (4주)

**대상 파일** (Nanobot 레포)

| 파일 | 역할 | LOC |
|-----|------|-----|
| `app_server.py` (신규) | JSON-RPC 2.0 stdio 서버 | 400 |
| `thread_manager.py` (신규) | 스레드/턴/메모리 상태 관리 | 200 |

**구현 (Python)**

```python
# app_server.py
import sys
import json
from pathlib import Path
from typing import Optional, Dict, Any

class NanobotAppServer:
    def __init__(self):
        self.threads: Dict[str, "Thread"] = {}
        self.thread_counter = 0

    def handle_message(self, message: Dict[str, Any]) -> Optional[Dict]:
        """JSON-RPC 메서드 디스패치."""
        method = message.get("method")
        params = message.get("params", {})
        msg_id = message.get("id")

        try:
            if method == "initialize":
                return self._handle_initialize(params, msg_id)
            elif method == "thread/start":
                return self._handle_thread_start(params, msg_id)
            elif method == "turn/start":
                return self._handle_turn_start(params, msg_id)
            elif method == "shutdown":
                return self._handle_shutdown(params, msg_id)
            else:
                return {"error": {"code": -32601, "message": "Method not found"}}
        except Exception as e:
            return {"error": {"code": -32603, "message": str(e)}}

    def _handle_initialize(self, params, msg_id) -> Dict:
        """initialize 요청 처리."""
        return {
            "jsonrpc": "2.0",
            "result": {
                "capabilities": {
                    "memoryManagement": True,
                    "dynamicToolRegistration": True,
                    "appServerVersion": "1.0.0"
                }
            },
            "id": msg_id
        }

    def _handle_thread_start(self, params, msg_id) -> Dict:
        """thread/start 요청 처리 (issue_id 네임스페이스)."""
        cwd = params.get("cwd")
        issue_id = Path(cwd).name  # workspace 디렉토리명 = issue_id

        self.thread_counter += 1
        thread_id = f"T-{self.thread_counter}"

        thread = Thread(
            id=thread_id,
            issue_id=issue_id,
            cwd=cwd,
            sandbox=params.get("sandbox", {}).get("type", "none")
        )
        self.threads[thread_id] = thread

        return {
            "jsonrpc": "2.0",
            "result": {
                "thread": {
                    "id": thread_id,
                    "createdAt": thread.created_at.isoformat()
                }
            },
            "id": msg_id
        }

    def _handle_turn_start(self, params, msg_id) -> Dict:
        """turn/start 요청 처리 (스트리밍 이벤트)."""
        thread_id = params.get("threadId")
        input_text = params.get("input")

        thread = self.threads[thread_id]

        # 메모리 네임스페이스 로드
        memory_path = Path(thread.cwd) / ".memory" / f"issue_{thread.issue_id}" / "MEMORY.md"

        # Nanobot 에이전트 실행 (스트리밍)
        # 각 이벤트를 stdout으로 전송

        # 1. delta 이벤트
        yield {
            "jsonrpc": "2.0",
            "method": "turn/delta",
            "params": {"delta": "Processing your request..."}
        }

        # 2. 에이전트 실행
        result = nanobot_agent_stream(
            input=input_text,
            memory_path=memory_path
        )

        for output_chunk in result:
            yield {
                "jsonrpc": "2.0",
                "method": "turn/delta",
                "params": {"delta": output_chunk}
            }

        # 3. 완료 이벤트
        return {
            "jsonrpc": "2.0",
            "method": "turn/completed",
            "params": {
                "output": result.full_output,
                "stopReason": "endTurn",
                "usage": {
                    "input_tokens": result.input_tokens,
                    "output_tokens": result.output_tokens
                }
            },
            "id": msg_id
        }

class Thread:
    def __init__(self, id: str, issue_id: str, cwd: str, sandbox: str):
        self.id = id
        self.issue_id = issue_id
        self.cwd = cwd
        self.sandbox = sandbox
        self.created_at = datetime.now()

def main():
    """stdin에서 JSON-RPC 메시지 읽고 처리."""
    server = NanobotAppServer()

    for line in sys.stdin:
        try:
            message = json.loads(line)
            response = server.handle_message(message)

            if response:
                print(json.dumps(response), flush=True)
        except json.JSONDecodeError:
            print(json.dumps({
                "error": {"code": -32700, "message": "Parse error"}
            }), flush=True)

if __name__ == "__main__":
    main()
```

**CLI 진입점**

```bash
# Python 모듈로 실행
python -m nanobot app-server
```

**검증 기준**

```bash
# app-server 시작
python -m nanobot app-server &
SERVER_PID=$!

# initialize 요청
echo '{"jsonrpc":"2.0","method":"initialize","params":{},"id":1}' | nc localhost 9000
# -> {"jsonrpc":"2.0","result":{...},"id":1}

# thread/start 요청
echo '{"jsonrpc":"2.0","method":"thread/start","params":{"cwd":"/tmp/MT-686"},"id":2}' | nc localhost 9000
# -> {"jsonrpc":"2.0","result":{"thread":{"id":"T-1",...}},"id":2}

kill $SERVER_PID
```

---

### Task 2-3: Symphony HttpAgent 제거, AppServer 복귀 (1주)

**수정 파일** (Symphony 레포)

| 파일 | 변경 | 영향 |
|-----|------|------|
| `lib/symphony_elixir/agent_runner.ex` | `run_http_turns` 제거, `run_codex_turns` 복귀 | AppServer 로직 사용 |
| `lib/symphony_elixir/http_agent.ex` | 삭제 (2주 Task 1-2에서 추가한 파일) | HTTP 브릿지 제거 |
| `config/config.exs` | codex_command 설정으로 Nanobot 지정 | "python -m nanobot app-server" |

**config.exs**

```elixir
# config/config.exs

import Config

config :symphony_elixir,
  # Phase 1 (HTTP 브릿지)
  # http_agent_url: "http://localhost:8000"

  # Phase 2 (app-server)
  codex_command: "python -m nanobot app-server",

  # Tracker 설정
  tracker_kind: "linear",

  # Linear API
  linear_api_key: System.get_env("LINEAR_API_KEY"),
  linear_team_id: "DEV"
```

**app-server 프로토콜로 보고 Codex 방언 제거**

```elixir
# lib/symphony_elixir/agent_runner.ex (변경 요약)

# 기존 (Phase 1)
def run_http_turns(issue, workspace, turn_count, context) do
  # HTTP POST 호출
end

# 신규 (Phase 2)
def run_codex_turns(issue, workspace, turn_count, context) do
  # AppServer 스트림 읽기
  # JSON-RPC 메서드 호출

  {:ok, app_server} = AppServer.start(codex_command())
  {:ok, thread_id} = AppServer.start_thread(app_server, %{cwd: workspace})

  for turn <- 1..turn_count do
    prompt = make_prompt(issue, context, turn)

    # turn/start 이벤트 스트림
    stream = AppServer.start_turn(app_server, %{
      threadId: thread_id,
      input: prompt
    })

    # 스트림 이벤트 처리 (delta, toolUse, completed 등)
    result = process_stream(stream)

    # ClawPort 콜백
    ClawPort.record_turn(thread_id, turn, result.token_count)
  end

  AppServer.stop(app_server)
end
```

**검증 기준**

Phase 2 후 Phase 1과 동일한 기능이 app-server 프로토콜로 작동:
- Linear 이슈 폴링 [O]
- Nanobot 에이전트 스폰 (HTTP 대신 stdio) [O]
- ClawPort 에피소드 기록 [O]
- 메모리 네임스페이스 복원 [O]

---

## 5. Phase 3: 단일 제어 평면 (12개월)

**목표**
ClawPort가 이슈 관리 및 추적 소스가 되어 Linear 의존 제거 (선택적).

**산출물**
- ClawPort Tracker 어댑터 API (5개 엔드포인트)
- Symphony ClawPort 어댑터 (Tracker 구현)
- ClawPort 이슈 관리 UI

---

### Task 3-1: ClawPort Tracker 어댑터 API (3-4주)

**대상 파일** (ClawPort 레포)

| 파일 | 역할 | LOC |
|-----|------|-----|
| `src/models/Issue.ts` (신규) | 이슈 스키마 | 80 |
| `src/routes/tracker.ts` (신규) | Tracker API (5개 엔드포인트) | 250 |
| `src/db/schema.ts` (수정) | issues 테이블 추가 | +60 |

**이슈 스키마**

```typescript
// src/models/Issue.ts
export interface Issue {
  id: string;
  identifier: string;              // "MT-686"
  title: string;
  description?: string;
  state: IssueState;               // "todo" | "in_progress" | "done" | "canceled"

  priority?: "low" | "medium" | "high" | "urgent";
  assignee?: string;               // 에이전트 이름

  // 추적
  created_at: Date;
  updated_at: Date;
  started_at?: Date;               // "in_progress" 시작 시각
  completed_at?: Date;

  // 메타
  metadata?: {
    episode_ids?: string[];        // 관련 ClawPort 에피소드
    attempt_count?: number;
  };
}
```

**Tracker API 엔드포인트**

```typescript
// src/routes/tracker.ts
import { Router } from "express";
import { Issue, IssueState } from "../models/Issue";
import { db } from "../db";

const router = Router();

// 1. fetch_candidate_issues: 활성 이슈 조회
router.get("/issues", async (req, res) => {
  const state = req.query.state || "in_progress";

  const issues = await db.issues.find({
    state: state
  });

  // Symphony 기대값 형식
  res.json({
    issues: issues.map(issue => ({
      id: issue.id,
      identifier: issue.identifier,
      title: issue.title,
      state: issue.state
    }))
  });
});

// 2. fetch_issues_by_states: 특정 상태들의 이슈 조회
router.get("/issues/by-states", async (req, res) => {
  const states = req.query.states as string[];

  const issues = await db.issues.find({
    state: { $in: states }
  });

  res.json({
    issues: issues.map(issue => ({
      id: issue.id,
      identifier: issue.identifier,
      title: issue.title,
      state: issue.state
    }))
  });
});

// 3. fetch_issue_states_by_ids: 특정 이슈들의 상태 조회
router.post("/issues/batch-states", async (req, res) => {
  const { ids } = req.body;

  const issues = await db.issues.find({
    id: { $in: ids }
  });

  const states = Object.fromEntries(
    issues.map(issue => [issue.id, issue.state])
  );

  res.json({ states });
});

// 4. create_comment: 이슈에 코멘트 추가
router.post("/issues/:id/comments", async (req, res) => {
  const { content, author } = req.body;

  const issue = await db.issues.findOne({ id: req.params.id });

  if (!issue.comments) issue.comments = [];
  issue.comments.push({
    id: crypto.randomUUID(),
    author,
    content,
    created_at: new Date()
  });

  await db.issues.update(issue);
  res.json({ issue });
});

// 5. update_issue_state: 이슈 상태 업데이트
router.patch("/issues/:id/state", async (req, res) => {
  const { state } = req.body;

  const issue = await db.issues.findOne({ id: req.params.id });
  issue.state = state;
  issue.updated_at = new Date();

  if (state === "in_progress" && !issue.started_at) {
    issue.started_at = new Date();
  }
  if (state === "done") {
    issue.completed_at = new Date();
  }

  await db.issues.update(issue);
  res.json({ issue });
});

export default router;
```

**데이터베이스 스키마**

```sql
-- src/db/migrations/004_create_issues.sql
CREATE TABLE issues (
  id VARCHAR(36) PRIMARY KEY,
  identifier VARCHAR(50) NOT NULL UNIQUE,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  state VARCHAR(20) DEFAULT 'todo',
  priority VARCHAR(20),
  assignee VARCHAR(100),

  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL,
  started_at TIMESTAMP,
  completed_at TIMESTAMP,

  metadata JSON,
  comments JSON,

  INDEX (state),
  INDEX (created_at),
  INDEX (assignee)
);
```

---

### Task 3-2: Symphony ClawPort Tracker 어댑터 (1주)

**대상 파일** (Symphony 레포)

| 파일 | 역할 | LOC |
|-----|------|-----|
| `lib/symphony_elixir/trackers/clawport.ex` (신규) | ClawPort Tracker 구현 | 180 |
| `config/config.exs` (수정) | tracker_kind 선택 | +2 |

**구현 (Elixir)**

```elixir
# lib/symphony_elixir/trackers/clawport.ex
defmodule SymphonyElixir.Tracker.ClawPort do
  @behaviour SymphonyElixir.Tracker

  require Logger

  @impl true
  def fetch_candidate_issues do
    api_url = Config.clawport_url()

    case Req.get(api_url <> "/api/tracker/issues?state=in_progress") do
      {:ok, response} ->
        response.body["issues"]

      {:error, reason} ->
        Logger.error("Failed to fetch issues from ClawPort: #{inspect(reason)}")
        []
    end
  end

  @impl true
  def fetch_issues_by_states(states) do
    api_url = Config.clawport_url()
    query = URI.encode_query(%{"states" => Enum.join(states, ",")})

    case Req.get(api_url <> "/api/tracker/issues/by-states?#{query}") do
      {:ok, response} ->
        response.body["issues"]

      {:error, reason} ->
        Logger.error("Failed to fetch issues: #{inspect(reason)}")
        []
    end
  end

  @impl true
  def fetch_issue_states_by_ids(ids) do
    api_url = Config.clawport_url()

    case Req.post(
      api_url <> "/api/tracker/issues/batch-states",
      json: %{ids: ids}
    ) do
      {:ok, response} ->
        response.body["states"]

      {:error, reason} ->
        Logger.error("Failed to fetch issue states: #{inspect(reason)}")
        %{}
    end
  end

  @impl true
  def create_comment(issue_id, content, author \\ "Symphony") do
    api_url = Config.clawport_url()

    case Req.post(
      api_url <> "/api/tracker/issues/#{issue_id}/comments",
      json: %{
        content: content,
        author: author
      }
    ) do
      {:ok, _response} ->
        :ok

      {:error, reason} ->
        Logger.error("Failed to create comment: #{inspect(reason)}")
        {:error, reason}
    end
  end

  @impl true
  def update_issue_state(issue_id, state_name) do
    api_url = Config.clawport_url()

    case Req.patch(
      api_url <> "/api/tracker/issues/#{issue_id}/state",
      json: %{state: state_name}
    ) do
      {:ok, _response} ->
        :ok

      {:error, reason} ->
        Logger.error("Failed to update issue state: #{inspect(reason)}")
        {:error, reason}
    end
  end
end
```

**config.exs 업데이트**

```elixir
# config/config.exs

config :symphony_elixir,
  # Tracker 선택 (기본값: "linear")
  tracker_kind: System.get_env("TRACKER_KIND", "linear"),

  # ClawPort URL (tracker_kind == "clawport" 일 때만 사용)
  clawport_url: System.get_env("CLAWPORT_URL", "http://localhost:3000")
```

**tracker.ex 어댑터 선택 로직**

```elixir
# lib/symphony_elixir/tracker.ex
defmodule SymphonyElixir.Tracker do
  def adapter do
    case Config.tracker_kind() do
      "memory"    -> SymphonyElixir.Tracker.Memory
      "clawport"  -> SymphonyElixir.Tracker.ClawPort  # 신규
      _           -> SymphonyElixir.Tracker.Linear
    end
  end
end
```

---

### Task 3-3: ClawPort 이슈 관리 UI (4주)

**구현 항목**

- 이슈 생성 폼 (제목, 설명, 우선도)
- 이슈 목록 뷰 (상태별 필터, 검색)
- 이슈 상세 뷰 (상태 전환, 코멘트 추가)
- 대시보드 통합 (에피소드 + 이슈 연결 뷰)

**UI 스크린샷 (개요)**

```
ClawPort Dashboard
|-- Issues
|   |-- Create Issue (+ button)
|   |-- Filter (state: todo, in_progress, done)
|   +-- List
|       |-- MT-686 | Implement counter | in_progress | ...
|       |-- MT-687 | Add tests | todo | ...
|       +-- ...
+-- Episodes
    |-- Filter (issue_id, framework, status)
    +-- List
        |-- EP-001 | MT-686 | nanobot | 3 turns | $5.23 | [DONE]
        |-- EP-002 | MT-686 | nanobot | 1 turns | $1.02 | [FAIL]
        +-- ...
```

**검증 기준**

```bash
# ClawPort UI에서 이슈 생성
POST http://localhost:3000/api/tracker/issues \
  -d '{"title":"New feature","description":"...","priority":"high"}'
# -> {"issue": {id, identifier: "MT-691", ...}}

# Symphony이 ClawPort 이슈를 폴링 및 처리
TRACKER_KIND=clawport CLAWPORT_URL=http://localhost:3000 \
  mix ecto.start symphony_service

# 이슈 완료 후 상태 업데이트
PATCH http://localhost:3000/api/tracker/issues/MT-691/state \
  -d '{"state":"done"}'

# 대시보드에서 선형 파이프라인 가시성 확인
```

---

## 6. 핵심 기술 장벽 및 해결 방안

| 장벽 | 설명 | 해결 방안 | Phase | 영향 |
|------|------|----------|-------|------|
| **메모리 생명주기** | Symphony workspace 정리 시 Claw 메모리(MEMORY.md, LanceDB) 소실 | issue_id 네임스페이스, workspace 외부 .memory/ 저장 | 1 | HIGH |
| **에피소드 vs 세션** | ClawPort의 세션 뷰와 Symphony의 에피소드 개념 혼동 | EpisodeView 신규 추가, 세션=에피소드 통일 | 1 | MEDIUM |
| **프로토콜 방언** | Codex 전용 이벤트 타입이 앱서버 스펙에 섞여 있음 | app-server 중립 스펙 정의 (JSON-RPC 2.0) | 2 | HIGH |
| **ClawPort 쓰기 권한** | ClawPort는 현재 읽기 전용 관찰 도구 | Tracker 어댑터 API (CRUD) 신규 추가 | 3 | MEDIUM |
| **비용 단위 불일치** | 세션당 vs 이슈당 vs 턴당 비용 집계 방식 다름 | 에피소드 단위 집계, 조정 가능 환율 정의 | 1 | LOW |
| **메모리 격리 (멀티 에이전트)** | 여러 에이전트가 동일 issue_id에서 경쟁 | 에이전트 ID + issue_id 복합 네임스페이스 | 3+ | MEDIUM |

---

## 7. Go/No-Go 체크리스트

### Phase 1 Go 조건 (3개월 후)

**기능 검증**

- [O] Linear 이슈 20건 자동 완료 (7일 무인 운영, 수동 개입 0)
- [O] 모든 에이전트 실행이 ClawPort Episodes 테이블에 기록됨
- [O] 이슈별 비용($) 집계 가능 (이슈 -> 모든 에피소드 -> 합계)
- [O] 재시도 시 메모리 복원 성공률 >90% (issue_id 네임스페이스)

**운영 검증**

- [O] 에러율 <5% (에이전트 추적 실패 건수)
- [O] 평균 턴 지연 <2초
- [O] ClawPort 대시보드 로드 시간 <500ms

---

### Phase 2 Go 조건 (6개월 후)

**기능 검증**

- [O] Nanobot app-server 모드가 Phase 1과 동일한 기능 제공
  - 이슈 폴링 [O]
  - 에이전트 디스패치 [O]
  - 메모리 복원 [O]
  - Episodes 기록 [O]
- [O] HTTP 브릿지 레이어 완전 제거
- [O] 30일 무인 운영 (수동 개입 0)

**성능 검증**

- [O] app-server 시작 시간 <1초
- [O] 턴 지연 Phase 1과 동일 (<2초 평균)

---

### Phase 3 Go 조건 (12개월 후)

**기능 검증**

- [O] ClawPort Tracker 어댑터로 Linear 없이 30일 무인 운영 가능
- [O] ClawPort 이슈 생성/수정/상태 전환 UI 완성
- [O] 에이전트가 ClawPort 이슈 -> 완료 -> 상태 업데이트 전체 사이클 자동화

**비용/성능 검증**

- [O] 에이전트 비용 예측 오차 <15%
- [O] ClawPort 단일 대시보드에서 전체 파이프라인 가시성 100% (이슈 -> 에피소드 -> 턴 -> 토큰/비용)
- [O] 대시보드 로드 시간 <1초 (300+ 에피소드 조회)

---

## 부록: 산출물 체크리스트

| Phase | 산출물 | 파일 | 상태 |
|-------|--------|------|------|
| 1 | Nanobot HTTP 게이트웨이 | `nanobot_http_gateway.py` | 계획 |
| 1 | 세션 관리자 | `session_manager.py` | 계획 |
| 1 | 메모리 네임스페이스 | `memory_namespace.py` | 계획 |
| 1 | Symphony HttpAgent | `lib/symphony_elixir/http_agent.ex` | 계획 |
| 1 | ClawPort Episodes API | `src/routes/episodes.ts` | 계획 |
| 1 | Episodes 모델 | `src/models/Episode.ts` | 계획 |
| 1 | 통합 테스트 | `tests/phase1_integration_test.sh` | 계획 |
| 2 | app-server 프로토콜 스펙 | `docs/app-server-spec.md` | 계획 |
| 2 | Nanobot app-server | `app_server.py` | 계획 |
| 2 | 스레드 관리자 | `thread_manager.py` | 계획 |
| 3 | ClawPort Tracker API | `src/routes/tracker.ts` | 계획 |
| 3 | Issues 모델 | `src/models/Issue.ts` | 계획 |
| 3 | Symphony ClawPort 어댑터 | `lib/symphony_elixir/trackers/clawport.ex` | 계획 |
| 3 | ClawPort 이슈 UI | `src/pages/Issues.tsx` | 계획 |

---

**작성**: 2026-03-08
**기반**: symphony_integration_report.md, symphony_report.md
**다음 단계**: Phase 1 Task 1-1 (Nanobot HTTP 게이트웨이) 시작
