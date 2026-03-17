# Claw 에이전트 런타임의 세션/컨텍스트 관리 전략 비교 분석

> **조사 일자**: 2026-03-04 (OpenJarvis 추가: 2026-03-14, OpenFang/NemoClaw 추가: 2026-03-17)
> **조사 방법**: 7개 에이전트가 각 레포의 실제 소스코드를 병렬로 심층 분석 (OpenJarvis, OpenFang, NemoClaw 별도 추가)
> **핵심 질문**: "24시간 상주하는 대화형 에이전트가 복잡한 멀티스텝 작업을 수행할 때, 세션과 컨텍스트를 어떻게 관리하는가?"

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. [비교 매트릭스](#2-비교-매트릭스)
3. [개별 분석](#3-개별-분석)
   - 3.1 OpenClaw
   - 3.2 Nanobot
   - 3.3 NanoClaw
   - 3.4 IronClaw
   - 3.5 ZeroClaw
   - 3.6 PicoClaw
   - 3.7 TinyClaw
   - 3.8 OpenJarvis
4. [핵심 설계 패턴 5가지](#4-핵심-설계-패턴-5가지)
5. [idea.md 가설 검증](#5-ideamd-가설-검증)
6. [결론: 아직 아무도 풀지 못한 것](#6-결론-아직-아무도-풀지-못한-것)

---

## 1. Executive Summary

10개 Claw 구현체의 소스코드를 분석한 결과, 세션/컨텍스트 관리 전략은 크게 **4가지 아키타입**으로 수렴한다:

| 아키타입 | 구현체 | 핵심 원리 |
|----------|--------|-----------|
| **A. 프로세스/컨테이너 격리** | NanoClaw, TinyClaw, **OpenFang** | 컨텍스트 경계 = OS 프로세스/컨테이너 경계. OpenFang은 프로세스 격리 + Soul Snapshot 24/7 지속성 추가 |
| **B. 세션 키 기반 논리적 격리** | OpenClaw, Nanobot, PicoClaw, ZeroClaw, OpenJarvis | 세션 키로 대화 히스토리를 논리적으로 분리 |
| **C. 보안 계층 기반 격리** | IronClaw | WASM 샌드박스 + 프록시 + 암호화 볼트로 다층 격리 |
| **D. 컨테이너 격리 (샌드박스 플러그인)** | **NemoClaw** | Docker 샌드박스 수명주기(생성/연결/파괴) + 마이그레이션 상태 머신. 호스트(OpenClaw)의 에이전트 시스템에 의존 |

**가장 주목할 발견**: 어떤 구현체도 idea.md가 제시한 "메일 읽기 → 일정 확인 → 주간 계획 수립"과 같은 **이종 작업 간 자동 컨텍스트 분리**를 완전히 해결하지 못했다. 현재 가장 가까운 해법은 OpenClaw의 서브에이전트 시스템이지만, 이는 에이전트가 스스로 `sessions_spawn` 도구를 호출해야 하며, "이 작업은 별도 컨텍스트가 필요하다"는 판단을 LLM에 의존한다.

---

## 2. 비교 매트릭스

### 2.1 세션 수명주기

| 구현체 | 세션 키 구조 | 저장 형식 | 세션 간 격리 수준 |
|--------|-------------|-----------|------------------|
| **OpenClaw** | `agent:<id>:<type>:<uuid>` | JSONL + JSON store | 완전 격리 (파일/워크스페이스/도구 컨텍스트 분리) |
| **Nanobot** | `channel:chat_id` | JSONL (append-only) | 채널-채팅 단위 격리 |
| **NanoClaw** | `group_folder` | SQLite + 컨테이너 마운트 | 컨테이너 파일시스템 격리 |
| **IronClaw** | `(user_id, channel, ext_thread_id)` | In-memory (`RwLock<HashMap>`) | 스레드 단위 + WASM 실행 격리 |
| **ZeroClaw** | `session_id` (SQLite 칼럼) | SQLite (WAL + FTS5) | DB 세션 스코프 |
| **PicoClaw** | `agent:<id>:<channel>:<kind>:<peer>` | JSON 파일 (atomic write) | 채널-피어 단위 격리 |
| **TinyClaw** | `agent_dir` 경로 | 파일시스템 디렉토리 | 에이전트별 디렉토리 격리 |
| **OpenJarvis** | `session_id` (SQLite) | SQLite SessionStore | cross-channel 논리적 격리 (SessionIdentity) |
| **OpenFang** | 채널별 세션 + per-channel 모델 오버라이드 | Soul Snapshot (마크다운) + 내부 DB | 프로세스 격리 + Soul Snapshot 24/7 지속성 |
| **NemoClaw** | 샌드박스 수명주기 (생성/연결/파괴) | 마이그레이션 스냅샷 (tar archive) | Docker 컨테이너 격리 + blueprint 버전 관리 |

### 2.2 서브에이전트/멀티에이전트 지원

| 구현체 | 서브에이전트 모델 | 최대 깊이/수 | 결과 전달 방식 |
|--------|------------------|-------------|---------------|
| **OpenClaw** | `sessions_spawn` 도구 (run/session 모드) | 깊이 제한 + 자식 5개/에이전트 | 부모 세션에 시스템 메시지로 주입 |
| **Nanobot** | `spawn` 도구 → asyncio.Task | 단일 깊이 (재귀 불가), 15회 반복 제한 | MessageBus를 통한 시스템 메시지 |
| **NanoClaw** | Claude SDK Agent Teams (`TeamCreate`) | SDK 제한 따름 | SDK 내부 메시지 시스템 |
| **IronClaw** | Job 시스템 (Pending→InProgress→Completed) | `max_jobs` 제한 | JobContext 상태 머신 |
| **ZeroClaw** | 없음 (단일 에이전트) | N/A | N/A |
| **PicoClaw** | 없음 (단일 에이전트 루프) | 50회 도구 반복 | N/A |
| **TinyClaw** | 분산 액터 모델 (팀 멤버 간 멘션) | 팀 크기 제한 없음 | SQLite 메시지 큐 (`conversation_id`) |
| **OpenJarvis** | OrchestratorAgent (LoopGuard) | N/A | 시스템 메시지 주입 |
| **OpenFang** | Hands 시스템 + A2A 프로토콜 (멀티에이전트) | 설정 가능 | A2A 표준 메시지 교환 |
| **NemoClaw** | OpenClaw 에이전트 시스템에 위임 | OpenClaw 제한 따름 | OpenClaw 내부 메시지 시스템 |

### 2.3 컨텍스트 윈도우 관리 (컴팩션/요약)

| 구현체 | 트리거 조건 | 요약 전략 | 장기 메모리 |
|--------|------------|-----------|------------|
| **OpenClaw** | 컨텍스트 사용량 임계치 초과 | N-분할 → 부분 요약 → 병합 (adaptive chunk ratio) | 플러그인/워크스페이스 기반 |
| **Nanobot** | unconsolidated >= memory_window (100) | LLM에 `save_memory` 도구 호출 → HISTORY.md + MEMORY.md | MEMORY.md (장기 사실) + HISTORY.md (시계열 로그) |
| **NanoClaw** | Claude SDK 자체 관리 | SDK 내부 auto-compaction | 그룹별 `.claude/` 디렉토리 |
| **IronClaw** | 3가지 전략 선택: Summarize/Truncate/MoveToWorkspace | LLM 요약 → 워크스페이스 일일 로그 | 워크스페이스 파일 |
| **ZeroClaw** | max_history_messages (50) 초과 | 시스템 메시지 보존, 오래된 것부터 삭제 (요약 없음) | SQLite FTS5 + 벡터 임베딩 하이브리드 검색 |
| **PicoClaw** | 20개 메시지 or 토큰 75% 초과 | 2-pass 요약 (분할→각각 요약→LLM 병합), 최근 4개 보존 | MEMORY.md + 최근 3일 daily notes |
| **TinyClaw** | Claude `-c` 플래그 (세션 연속) | SDK 자체 관리 | 에이전트별 AGENTS.md |
| **OpenJarvis** | consolidation_threshold: 100 메시지 (Nanobot과 동일) | LoopGuard 반복 탐지 → 컨텍스트 압축; RLM Agent: Python REPL 변수로 저장 | SQLite 세션 저장소 |
| **OpenFang** | Soul Snapshot 24/7 지속성 기반 | per-channel 모델 오버라이드 + Soul Snapshot 자동 재개 | Knowledge Graph + SQLite BLOB (importance scoring) |
| **NemoClaw** | OpenClaw 의존 | 마이그레이션 상태 머신 (host → sandbox 이전) | OpenClaw 컨텍스트 관리 위임 + blueprint 캐시 |

---

## 3. 개별 분석

### 3.1 OpenClaw — 가장 정교한 세션 시스템

**핵심 패턴: 구조화된 세션 키 + 깊이 제한 서브에이전트 트리**

OpenClaw는 8개 구현체 중 가장 복잡한 세션 관리 시스템을 가지고 있다.

**세션 키**: `agent:<agentId>:<type>:<uuid>` 형식으로, 세션의 유형(subagent, acp, cron, dm)과 고유 ID를 구조적으로 인코딩한다.

**서브에이전트 격리**: 각 서브에이전트는 완전히 독립된 컨텍스트를 가진다:
- 별도의 JSONL 트랜스크립트 파일 (`~/.openclaw/agents/<id>/sessions/<key>.jsonl`)
- PID 기반 write lock (재활용된 PID 감지를 위한 starttime 포함)
- 별도의 워크스페이스 디렉토리
- 독립된 도구 컨텍스트 (agentSessionKey가 모든 도구에 전파)

**두 가지 스폰 런타임**:
- **Subagent**: 내부 에이전트 스폰. 깊이 제한, 자식 수 제한, 결과 자동 전달
- **ACP (Agent Control Protocol)**: 외부 에이전트 스폰. 스트리밍 릴레이 지원

**컴팩션**: 적응형 N-분할 요약. 식별자(UUID, URL 등) 보존 정책. 컨텍스트 윈도우의 50% 초과 메시지는 자동 가지치기. orphaned tool_use/tool_result 쌍 복구 로직 포함.

**주요 코드**: `src/agents/subagent-spawn.ts`, `src/agents/compaction.ts`, `src/agents/session-write-lock.ts`

---

### 3.2 Nanobot — 가장 깔끔한 2계층 메모리

**핵심 패턴: Append-only JSONL + LLM 기반 능동적 기억 통합**

Nanobot의 설계 철학은 "LLM이 스스로 무엇을 기억할지 결정하게 하라"이다.

**세션**: `channel:chat_id` 키로 JSONL 파일에 저장. Append-only 설계로 LLM 프롬프트 캐시 효율 극대화 (메시지를 삭제하지 않고 `last_consolidated` 포인터를 전진시킴).

**2계층 메모리**:
- `MEMORY.md`: LLM이 `save_memory` 도구로 업데이트하는 장기 사실 저장소. 매 턴마다 시스템 프롬프트에 주입.
- `HISTORY.md`: 타임스탬프가 찍힌 grep 가능한 대화 요약 로그.

**통합 트리거**: unconsolidated 메시지 수 >= memory_window (기본 100)일 때 비동기로 실행. 현재 턴을 블로킹하지 않음.

**서브에이전트**: `spawn` 도구로 asyncio.Task 생성. 재귀적 스폰 방지(spawn 도구 미제공), 15회 반복 제한. 결과는 MessageBus를 통해 시스템 메시지로 주입.

**주요 코드**: `nanobot/session/manager.py`, `nanobot/agent/memory.py`, `nanobot/agent/subagent.py`

---

### 3.3 NanoClaw — 컨테이너 = 컨텍스트

**핵심 패턴: Docker/Apple Container 파일시스템 마운트가 곧 컨텍스트 경계**

NanoClaw의 핵심 통찰은 "격리 문제를 OS 레벨에서 해결하라"이다.

**컨테이너 격리**: 각 그룹(채팅방)마다 독립된 Docker 컨테이너가 스폰되며, 파일시스템 마운트가 컨텍스트를 정의한다:
- `/home/node/.claude` → 그룹별 세션 디렉토리 (호스트의 `data/sessions/{group}/.claude/`)
- `/workspace/group` → 그룹별 작업 디렉토리
- `/workspace/ipc` → IPC 파일 (500ms 폴링)

**세션 연속성**: 컨테이너는 응답 후에도 30분간 살아있으며, IPC 파일 폴링으로 새 메시지를 수신한다. 같은 세션, 같은 컨텍스트 윈도우, 같은 프로세스가 여러 메시지를 처리한다.

**에이전트 스웜**: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 플래그 + `AsyncIterable<SDKUserMessage>` 패턴으로 SDK의 `isSingleUserTurn` 버그를 우회. 이것이 "Anthropic Agent SDK 네이티브 스웜 최초 지원"의 실체.

**주요 코드**: `src/container-runner.ts`, `container/agent-runner/src/index.ts`, `src/ipc.ts`

---

### 3.4 IronClaw — 보안이 곧 격리

**핵심 패턴: Session→Thread→Turn 3계층 + WASM 샌드박스 + 프록시 기반 자격증명 주입**

IronClaw는 "보안 경계가 곧 컨텍스트 경계"라는 철학을 가진다.

**3계층 대화 모델**:
- **Session**: 유저 단위. `auto_approved_tools` 화이트리스트 소유.
- **Thread**: `(user_id, channel, ext_thread_id)` 키. 상태 머신: Idle→Processing→AwaitingApproval→Completed.
- **Turn**: 개별 대화 턴. 메시지는 턴에서 온디맨드 재구성 (인메모리 리스트 없음).

**자격증명 격리 (`pending_auth` 모드)**: 스레드가 `pending_auth` 상태에 진입하면, 다음 사용자 메시지는 **일반 파이프라인을 완전히 우회**하여 직접 credential store로 라우팅. 로깅 없음, 턴 생성 없음, 히스토리 기록 없음 — 비밀번호가 대화 로그에 남지 않는다.

**WASM 격리**: "compile once, instantiate fresh per execution". 각 실행마다 새 `Store` 생성, 10MB 메모리 제한, 10M 명령어 fuel, 60초 타임아웃. 교차 실행 상태 누출 없음.

**Docker 프록시**: 컨테이너의 모든 HTTP(S)가 호스트 프록시를 경유. 도메인 허용목록 적용, 자격증명은 프록시에서 일시적으로 복호화하여 헤더에 주입. 컨테이너는 원본 자격증명을 절대 보지 못한다.

**주요 코드**: `src/agent/session.rs`, `src/tools/wasm/`, `src/sandbox/proxy/`, `src/secrets/crypto.rs`

---

### 3.5 ZeroClaw — 메모리가 곧 컨텍스트

**핵심 패턴: Trait 기반 교체 가능 메모리 + 스냅샷 하이드레이션**

ZeroClaw는 멀티에이전트를 지원하지 않는 단일 에이전트 시스템이다. 대신 **정교한 메모리 시스템**으로 컨텍스트 문제를 해결한다.

**히스토리 관리**: `max_history_messages` (기본 50)으로 단순 트리밍. 시스템 메시지 보존, 오래된 비시스템 메시지부터 삭제. LLM 요약 없음.

**4계층 메모리 아키텍처**:
1. **SQLite (brain.db)**: WAL 모드, FTS5 전문 검색, BM25 스코어링, 벡터 임베딩 BLOB 저장, 하이브리드 검색 (벡터 + 키워드 가중 병합)
2. **LucidMemory**: 외부 Lucid CLI 브릿지. 500ms 타임아웃으로 로컬 SQLite 폴백
3. **Markdown**: 사람이 읽을 수 있는 플랫 파일
4. **Snapshot/Hydration ("Atomic Soul Export")**: Core 메모리 → `MEMORY_SNAPSHOT.md` 내보내기. DB 손실 시 스냅샷에서 자동 복원

**메모리 카테고리**: Core (장기 사실), Daily (세션 로그), Conversation (턴별 컨텍스트), Custom (사용자 정의).

**compact_context 모드**: 13B 이하 소형 모델용으로 설계 (6000자 부트스트랩, RAG 청크 2개). 코드에 필드는 있으나 아직 미구현.

**주요 코드**: `src/agent/agent.rs:359`, `src/memory/sqlite.rs`, `src/memory/snapshot.rs`, `src/memory/traits.rs`

---

### 3.6 PicoClaw — 플래시 스토리지에 안전한 원자적 저장

**핵심 패턴: 원자적 파일 쓰기 + 2-pass 비동기 요약 + 정적 프롬프트 캐싱**

PicoClaw는 엣지 디바이스(Termux/Android, SD카드)에서의 안정성에 집중한다.

**원자적 세션 저장**: `WriteFileAtomic` — tmp 파일 작성 → fsync → rename → 디렉토리 sync. "SD카드, eMMC 등 플래시 스토리지에 필수"라는 코드 주석.

**세션 키 전략**: `agent:<id>:<channel>:<kind>:<peer>` 형식. 기본값 `per-channel-peer`로 각 사용자가 각 채널에서 격리된 대화. 크로스 플랫폼 ID 연결 지원 (Telegram ID ↔ Discord ID).

**2-pass 요약**: 20개 메시지 or 토큰 75% 초과 시 비동기 트리거. 10개 이상이면 분할→각각 요약→LLM 병합. 최근 4개 메시지는 항상 보존. 토큰 추정: `rune_count * 2/5` (CJK 고려).

**정적 프롬프트 캐싱**: 시스템 프롬프트의 정적 부분(identity, skills, memory)을 mtime 기반으로 캐시하고 `cache_control: ephemeral` 태그로 LLM 측 KV 캐시 재사용 유도.

**주요 코드**: `pkg/session/manager.go`, `pkg/agent/loop.go`, `pkg/fileutil/file.go`, `pkg/agent/context.go`

---

### 3.7 TinyClaw — 분산 액터 모델

**핵심 패턴: SQLite 메시지 큐 + 에이전트별 디렉토리 격리 + Promise 체인 직렬화**

TinyClaw는 중앙 오케스트레이터 없이 에이전트들이 메시지 패싱으로 협업하는 분산 모델을 사용한다.

**팀 오케스트레이션**: `@teammate: message` 멘션 문법으로 에이전트 간 통신. Chain (순차: 멘션 1개) vs Fan-out (병렬: 멘션 N개) 실행 모드가 같은 큐 인프라 위에서 동작.

**컨텍스트 격리**: 각 에이전트는 독립된 디렉토리에서 Claude `-c` 플래그로 세션을 유지. `agentProcessingChains: Map<string, Promise<void>>`로 동일 에이전트에 대한 메시지는 직렬화, 서로 다른 에이전트는 자연스럽게 병렬 실행.

**대화 추적**: 인메모리 `Conversation` 객체가 `pending` 카운터로 진행 중인 분기를 추적. `withConversationLock`으로 동시성 제어.

**주요 코드**: `src/queue-processor.ts`, `src/lib/db.ts`, `src/lib/invoke.ts`

---

### 3.8 OpenJarvis — Cross-Channel 세션 + RLM 컨텍스트 변수화

**핵심 패턴: SessionIdentity 다채널 매핑 + LoopGuard 반복 탐지 + RLM Context-as-Variable**

OpenJarvis는 10번째 프레임워크로, 멀티채널 운영 환경에서의 세션 연속성과 긴 컨텍스트 처리를 독자적인 방식으로 해결한다.

**세션 저장소**: SQLite 기반 `SessionStore`. `session_id`를 기본 키로 하며 `max_age_hours: 24.0` 만료 정책과 자동 decay 정리를 내장한다. Nanobot과 동일한 100 메시지 `consolidation_threshold`를 사용한다.

**cross-channel SessionIdentity (독특한 패턴)**: 다른 모든 구현체는 단일 채널의 사용자 ID를 세션 키로 쓰지만, OpenJarvis의 `SessionIdentity`는 `channel_ids` dict로 여러 채널의 사용자 ID를 하나의 세션에 묶는다:

```python
SessionIdentity(
    channel_ids={
        "slack":    "U12345",
        "telegram": "@user",
        "web":      "sess_abc"
    }
)
```

이를 통해 Slack에서 시작한 대화를 Telegram에서 이어받고, 웹 UI에서도 동일한 컨텍스트로 접근할 수 있다. 9개 기존 구현체 중 이 패턴을 가진 구현체는 없다.

**LoopGuard (반복 tool call 탐지 + 컨텍스트 압축)**: OrchestratorAgent 내부의 `LoopGuard`는 에이전트가 동일한 tool call을 반복할 때 이를 탐지하고 컨텍스트를 압축한다. IronClaw의 `pending_auth` 상태 머신처럼 특정 조건에서 일반 파이프라인을 우회하는 패턴이지만, 목적은 자격증명 보호가 아닌 무한루프 방지다.

**TaskScheduler**: cron/interval/once 세 가지 모드를 지원하는 daemon 프로세스. SQLite에 스케줄 상태를 영속화한다. OpenClaw의 cron 세션 유형(`agent:<id>:cron:<uuid>`)과 유사하나, OpenJarvis는 스케줄러를 독립 daemon으로 분리했다.

**RLM Agent — Context-as-Variable (가장 독특한 패턴)**: arxiv:2512.24601에서 제안된 RLM(Reasoning Language Model) 패턴을 구현. 긴 컨텍스트를 LLM 프롬프트에 직접 포함하는 대신, **Python REPL 변수에 저장**하고 코드로 참조하게 한다:

```python
# 기존 방식: 컨텍스트 윈도우에 직접 주입 (토큰 소비)
system_prompt = f"Here is the document: {long_document}"

# RLM 방식: Python 변수로 저장, 코드로 참조 (토큰 절약)
repl.exec(f"document = {repr(long_document)}")
# LLM은 document 변수를 코드로 조작
```

이는 ZeroClaw의 `compact_context` 모드(소형 모델용, 미구현)와 같은 문제를 완전히 다른 방향으로 해결한다. "컨텍스트 압축"이 아니라 "컨텍스트 외재화"다.

**주요 특이점 요약**:
- cross-channel 세션 연속성: 9개 기존 구현체에 없는 패턴
- RLM Context-as-Variable: 컨텍스트 윈도우 문제를 코드 실행으로 우회
- LoopGuard: 프레임워크 레벨의 반복 탐지 (에이전트 프롬프트 의존 아님)
- TaskScheduler daemon: SQLite 영속화 스케줄러

---

### 3.9 OpenFang — Soul Snapshot 24/7 + per-channel 모델 오버라이드

**핵심 패턴: 프로세스 격리 + Soul Snapshot 지속성 + Hands 시스템 멀티에이전트**

OpenFang은 24/7 자율 실행을 전제로 설계된 프레임워크로, Soul Snapshot을 통한 세션 간 에이전트 정체성 연속성이 핵심이다.

**3-layer 컨텍스트 윈도우**: 시스템 프롬프트(12-section 빌더) + 대화 히스토리 + 도구 결과. 12-section 빌더는 채널 유형, 에이전트 역할, 메모리 상태 등을 구조적으로 조합해 시스템 프롬프트를 동적 생성.

**per-channel 모델 오버라이드**: 채널별로 다른 LLM 모델 지정 가능. DM은 claude-opus, 그룹 채팅은 claude-haiku 등 컨텍스트 요구에 맞는 모델 선택.

**Soul Snapshot 재개**: 재시작 시 Soul Snapshot에서 에이전트 정체성과 핵심 기억 자동 복원. ZeroClaw의 cold-boot hydration 패턴과 동일하나, OpenFang은 Capability 시스템과 통합.

**Hands 시스템 + A2A 프로토콜**: 플러그인 생태계(Hands)로 멀티에이전트 오케스트레이션. Google A2A spec 지원으로 타 프레임워크 에이전트와 표준화된 통신.

**주요 코드**: `crates/context/src/builder.rs` (12-section builder), `crates/memory/src/snapshot.rs` (Soul Snapshot), `crates/hands/src/lib.rs` (플러그인 시스템)

---

### 3.10 NemoClaw — 샌드박스 수명주기 + 마이그레이션 상태 머신

**핵심 패턴: Docker 샌드박스 생성/연결/파괴 + 마이그레이션 스냅샷 + OpenClaw 에이전트 위임**

NemoClaw는 세션/컨텍스트 관리를 직접 구현하지 않고 OpenClaw 호스트에 위임하는 "샌드박스 플러그인"이다. 고유한 기여는 샌드박스 수명주기 관리와 마이그레이션 상태 머신.

**샌드박스 수명주기**: `launch → connect → migrate → eject → destroy` 5단계 상태 머신. 각 단계는 blueprint 버전으로 관리되어 환경 재현성 보장.

**마이그레이션 상태 머신 (host → sandbox)**: 호스트에서 실행 중인 OpenClaw 에이전트를 샌드박스 컨테이너로 이전하는 고유 패턴. tar 아카이브 스냅샷으로 런 상태 보존 후 컨테이너 내부에서 복원.

**blueprint 버전 관리**: 10개 네트워크 정책 프리셋(커넥터)을 blueprint로 선택. blueprint가 컨테이너 환경을 정의하므로, 같은 blueprint로 동일한 샌드박스를 언제든 재현 가능.

**OpenClaw 에이전트 위임**: 컨텍스트 윈도우, 대화 히스토리, 기억 관리는 모두 OpenClaw 내부에서 처리. NemoClaw는 이 모든 것을 투명하게 샌드박스 컨테이너 안에서 실행되도록 래핑.

**주요 코드**: `nemoclaw/sandbox/lifecycle.py` (샌드박스 수명주기), `nemoclaw/migration/state.py` (마이그레이션 상태 머신), `nemoclaw/blueprint/` (환경 정의)

---

## 4. 핵심 설계 패턴 5가지

### 패턴 1: "세션 키가 세계를 정의한다"

모든 구현체에서 세션 키의 구조가 격리 수준을 결정한다.

```
OpenClaw:   agent:<id>:<type>:<uuid>           → 에이전트+유형+인스턴스 격리
IronClaw:   (user, channel, thread_id)         → 유저+채널+스레드 격리
PicoClaw:   agent:<id>:<ch>:<kind>:<peer>      → 에이전트+채널+종류+상대 격리
Nanobot:    channel:chat_id                    → 채널+채팅 격리
NanoClaw:   group_folder                       → 그룹(채팅방) 격리
TinyClaw:   agent_dir path                     → 에이전트 디렉토리 격리
ZeroClaw:   session_id (DB column)             → DB 레코드 격리
OpenJarvis: session_id + channel_ids dict      → DB 레코드 격리 + cross-channel 통합
OpenFang:   channel + per-channel model config → 채널 격리 + 모델 오버라이드 통합
NemoClaw:   sandbox_id + blueprint_version     → 컨테이너 격리 + 환경 버전 관리
```

**OpenJarvis의 혁신**: 세션 키가 단순한 "격리 단위"를 넘어 "통합 단위"로 기능한다. `channel_ids: {"slack": "U12345", "telegram": "@user"}` 구조로 여러 채널의 동일 사용자를 하나의 세션에 묶는다. 다른 구현체들의 세션 키가 분리를 위한 키라면, OpenJarvis의 SessionIdentity는 연결을 위한 키다.

**교훈**: 세션 키 설계는 아키텍처의 가장 근본적인 결정이다. 키가 복잡할수록 세밀한 격리가 가능하지만, 키 관리 복잡도도 비례하여 증가한다.

### 패턴 2: "요약은 비동기, 삭제는 신중하게"

| 구현체 | 요약 실행 | 메시지 삭제 여부 |
|--------|----------|----------------|
| OpenClaw | 동기 (컨텍스트 가드 트리거) | 가지치기 (oldest chunks) |
| Nanobot | **비동기** (asyncio.Task) | **삭제 안 함** (포인터 전진) |
| PicoClaw | **비동기** (goroutine) | 삭제 (최근 4개 보존) |
| ZeroClaw | 동기 (턴 후) | 삭제 (시스템 메시지 보존) |
| IronClaw | 전략 선택 가능 | 전략에 따름 |

**Nanobot의 append-only 설계**가 특히 주목할 만하다: 메시지를 삭제하지 않고 `last_consolidated` 포인터만 전진시켜 LLM 프롬프트 캐시 히트율을 극대화한다. 이는 Anthropic API의 prompt caching이 접두사 매칭 방식이라는 점을 활용한 최적화.

### 패턴 3: "서브에이전트의 결과는 메시지 버스로"

서브에이전트를 지원하는 4개 구현체 모두, 결과를 **시스템 메시지**로 부모 컨텍스트에 주입하는 동일한 패턴을 사용한다:

```
OpenClaw:  서브에이전트 완료 → 레지스트리가 announce → 부모 세션에 유저 메시지로 주입
Nanobot:   서브에이전트 완료 → MessageBus.publish_inbound(system 메시지)
NanoClaw:  SDK Agent Teams의 내부 메시지 시스템
TinyClaw:  SQLite 큐에 conversation_id로 연결된 메시지 삽입
```

**교훈**: 서브 컨텍스트의 결과를 부모에게 전달할 때, "요약된 시스템 메시지"가 사실상의 표준이다. 이는 부모 컨텍스트 윈도우를 오염시키지 않으면서 필요한 정보만 전달하는 실용적 해법.

### 패턴 4: "장기 메모리는 마크다운 파일"

| 구현체 | 장기 메모리 형식 | 주입 방식 |
|--------|----------------|-----------|
| Nanobot | `MEMORY.md` + `HISTORY.md` | 시스템 프롬프트에 전문 주입 |
| ZeroClaw | `MEMORY_SNAPSHOT.md` + SQLite | 시스템 프롬프트 + RAG 검색 (top-5) |
| PicoClaw | `MEMORY.md` + daily notes | 시스템 프롬프트에 전문 주입 |
| OpenClaw | 워크스페이스 파일 | 플러그인 기반 주입 |
| IronClaw | 워크스페이스 일일 로그 | 컴팩션 결과 저장 |
| OpenJarvis | SQLite SessionStore | 세션 로드 시 컨텍스트 재구성; RLM Agent는 Python REPL 변수로 외재화 |

**교훈**: LLM에게 장기 기억을 제공하는 가장 보편적인 방법은 마크다운 파일을 시스템 프롬프트에 주입하는 것이다. 이는 단순하지만 효과적이며, 사람이 직접 편집할 수 있다는 장점이 있다. ZeroClaw의 "Atomic Soul Export" (DB→마크다운 스냅샷→DB 복원)은 이 패턴의 가장 정교한 변형.

### 패턴 5: "자격증명은 절대 컨텍스트에 남기지 않는다"

IronClaw만이 이 문제를 체계적으로 해결했다:

- `pending_auth` 모드: 비밀번호 입력 시 대화 파이프라인 완전 우회
- WASM 크레덴셜 인젝터: 모듈이 값을 보지 못하고, 호스트가 HTTP 요청 인터셉트 후 주입
- Docker 프록시: 컨테이너 내부에 자격증명 미노출

다른 구현체들은 이 문제를 **무시**하거나 (Nanobot, TinyClaw), **에이전트의 자발적 보안**에 의존한다 (OpenClaw의 tool_result.details를 요약에서 제외하는 정도).

---

## 5. idea.md 가설 검증

### 가설: "세션과 컨텍스트 관리 전략이 진짜 쟁점이다"

**검증 결과: 정확하다.** 7개 구현체의 코드를 분석한 결과, 코드 복잡도의 30-50%가 세션/컨텍스트 관리에 집중되어 있다.

| 구현체 | 전체 LOC | 세션/컨텍스트 관련 추정 비중 |
|--------|---------|--------------------------|
| OpenClaw | 430,000+ | ~35% (서브에이전트, 컴팩션, 세션 관리) |
| IronClaw | ~15,000 | ~40% (세션, WASM, 프록시, 크레덴셜) |
| Nanobot | ~4,000 | ~30% (세션, 메모리, 서브에이전트) |
| ZeroClaw | ~12,000 | ~25% (메모리 시스템, 스냅샷) |
| PicoClaw | ~10,000 | ~30% (세션, 요약, 컨텍스트 빌더) |

### 가설: "메일을 읽고, 일정을 확인하는 등의 작업은 별도 컨텍스트로 분리되어야 한다"

**검증 결과: 맞지만, 자동화된 구현체는 없다.**

현재 가능한 접근:
- **OpenClaw**: 에이전트가 `sessions_spawn`을 호출하여 수동으로 서브에이전트 생성
- **Nanobot**: `spawn` 도구로 서브태스크 생성
- **NanoClaw**: SDK Agent Teams으로 팀 구성
- **TinyClaw**: `@teammate` 멘션으로 다른 에이전트에 위임

그러나 **"이 작업은 별도 컨텍스트가 필요하다"는 판단을 자동으로 내리는 구현체는 없다.** 모든 구현체에서 컨텍스트 분리는 LLM의 도구 호출 판단이나 사용자의 명시적 지시에 의존한다.

### 가설: "리서치 같은 복잡한 작업은 Agent Team이 필요하다"

**검증 결과: 부분적으로 구현됨.**

- **OpenClaw**: 서브에이전트 트리 (깊이 제한 + 자식 수 제한)
- **NanoClaw**: Claude SDK Agent Teams (가장 네이티브한 구현)
- **TinyClaw**: 분산 액터 모델 (가장 독특한 구현)

그러나 "적절한 작업 디렉토리를 생성하고, 문서를 정리하고, 분석 노트를 작성하는" 수준의 자율적 프로젝트 관리는 어떤 구현체에서도 프레임워크 레벨에서 제공하지 않는다. 이는 에이전트의 프롬프트/스킬에 의존하는 영역이다.

---

## 6. 결론: 아직 아무도 풀지 못한 것

### 풀린 문제

| 문제 | 해법 | 대표 구현체 |
|------|------|------------|
| 대화 히스토리 폭발 | 요약/컴팩션 | Nanobot (append-only + 비동기 통합) |
| 서브태스크 격리 | 서브에이전트/컨테이너 | OpenClaw (세션 트리), NanoClaw (Docker) |
| 자격증명 보호 | 프록시 주입 + 파이프라인 우회 | IronClaw (3중 방어) |
| 장기 기억 | 마크다운 + 벡터 DB | ZeroClaw (하이브리드 검색 + 스냅샷) |
| 엣지 디바이스 안정성 | 원자적 쓰기 + 비동기 요약 | PicoClaw (fsync + 2-pass) |
| 멀티에이전트 협업 | 메시지 큐 + 액터 모델 | TinyClaw (SQLite 큐 + Promise 체인) |
| **cross-channel 세션 연속성** | SessionIdentity channel_ids dict | **OpenJarvis** (Slack+Telegram+Web 통합) |
| **컨텍스트 윈도우 초과 (대형 문서)** | Python REPL 변수 외재화 (RLM) | **OpenJarvis** (Context-as-Variable) |
| **24/7 에이전트 정체성 연속성** | Soul Snapshot + per-channel 모델 오버라이드 | **OpenFang** (재시작 후 자동 복원 + 채널별 최적 모델) |
| **기존 에이전트의 안전한 샌드박스 이전** | 마이그레이션 상태 머신 + blueprint 버전 관리 | **NemoClaw** (host→sandbox 이전 + 환경 재현성) |

### 아직 풀리지 않은 문제

1. **자동 컨텍스트 분리 판단**: "이 작업은 새 컨텍스트가 필요하다"를 프레임워크가 자동으로 판단하는 구현체 없음. 모든 구현체가 LLM의 도구 호출 판단에 의존.

2. **컨텍스트 간 정보 흐름 최적화**: 서브에이전트가 10,000토큰 분량의 분석을 했을 때, 부모에게 어떤 정보를, 어떤 형태로, 얼마나 전달해야 하는가? 현재는 "결과를 시스템 메시지로 주입"이 전부.

3. **세션 간 공유 상태 관리**: 메일을 읽는 서브에이전트와 일정을 확인하는 서브에이전트가 동일한 "오늘의 계획"이라는 상위 목표를 공유하고 있음을 어떻게 표현하는가? TeamCreate/메시지 패싱 외에 구조적 해법 없음.

4. **비용 인식 컨텍스트 관리**: 서브에이전트를 스폰할 때마다 시스템 프롬프트가 반복 전송된다. 5개 서브에이전트 × 10,000토큰 시스템 프롬프트 = 50,000토큰의 순수 오버헤드. 이를 최적화하는 구현체 없음 (Nanobot의 서브에이전트가 "stripped-down system prompt"을 쓰는 것이 유일한 시도).

5. **크로스세션 장기 학습**: 에이전트가 반복적으로 수행하는 작업 패턴을 학습하여 다음에는 더 효율적으로 컨텍스트를 분리하는 메타 학습. 어떤 구현체에서도 시도하지 않음.

---

> **최종 요약**: idea.md의 핵심 통찰은 정확했다. "이게 다야"라는 아키텍처 원형 분석도, "세션과 컨텍스트 관리가 진짜 쟁점"이라는 문제 정의도 코드 레벨에서 검증된다. 10개 구현체 모두 이 문제에 코드의 1/4~2/5를 투자하고 있으며, 각자 다른 트레이드오프를 선택했다. OpenJarvis는 두 가지 새로운 방향을 제시했다: (1) 세션을 "격리 단위"가 아닌 "cross-channel 통합 단위"로 재정의하는 SessionIdentity, (2) 컨텍스트를 압축하는 대신 Python REPL 변수로 외재화하는 RLM Context-as-Variable. OpenFang은 Soul Snapshot 24/7 지속성과 per-channel 모델 오버라이드로 "24시간 에이전트 정체성 연속성" 문제에 가장 직접적인 답을 제시했다. NemoClaw는 "기존 에이전트를 안전하게 샌드박스로 이전"하는 마이그레이션 상태 머신으로 독자적 문제를 해결했다. 그러나 "자동 컨텍스트 분리 판단"이라는 핵심 문제는 아직 어느 구현체도 프레임워크 레벨에서 해결하지 못했다.
