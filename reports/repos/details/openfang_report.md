# OpenFang 심층 분석 보고서 -- Agent OS 비교 연구

> **조사 일자**: 2026-03-08
> **조사 방법**: 6개 specialist 에이전트가 14개 크레이트 소스코드를 병렬 심층 분석
> **대상 레포**: `repos/openfang/` (RightNow-AI/openfang, squash subtree)
> **핵심 질문**: "OpenFang은 기존 8개 프레임워크 대비 어떤 차별점을 갖는가?"

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. [비교 매트릭스 (8 + 1 프레임워크)](#2-비교-매트릭스)
3. [Tool Architecture 분석](#3-tool-architecture-분석)
4. [Security 분석](#4-security-분석)
5. [Memory Architecture 분석](#5-memory-architecture-분석)
6. [Channel Adapter 분석](#6-channel-adapter-분석)
7. [Session/Context Management 분석](#7-sessioncontext-management-분석)
8. [Hands System 분석](#8-hands-system-분석)
9. [벤치마크 주장 검증](#9-벤치마크-주장-검증)
10. [기존 4개 보고서와의 Cross-validation](#10-cross-validation)
11. [결론 및 신규 오픈 퀘스천](#11-결론-및-신규-오픈-퀘스천)

---

## 1. Executive Summary

OpenFang은 스스로를 **"Agent OS"**라고 명명한다. 137K LOC, 14개 Rust 크레이트, 단일 ~32MB 바이너리. 기존 8개 프레임워크가 "대화형 에이전트"에 집중했다면, OpenFang의 핵심 명제는 다르다: **에이전트가 사용자를 기다리는 것이 아니라, 사용자가 잠든 동안 에이전트가 일한다.**

**가장 주목할 발견 5가지:**

1. **Hands System이 진짜 혁신이다.** 7개 번들 Autonomous Capability Package (Clip/Lead/Collector/Predictor/Researcher/Twitter/Browser)는 기존 8개 프레임워크에서 발견된 어떤 개념과도 다르다. HAND.toml이 도구·설정·요구사항·에이전트 프롬프트를 단일 선언적 파일로 묶고, activate/pause/resume/deactivate 생명주기를 가진다.

2. **Channel Adapter 40개는 사실이지만 Twitter/X는 없다.** 40개 채널은 기존 8개 프레임워크의 합계보다 많다. 그러나 Twitter Hand가 별도 존재함에도 Twitter 채널 어댑터는 미구현. 채널별 모델 오버라이드와 슬라이딩 윈도우 레이트 리밋은 독보적.

3. **16-Layer 보안은 아키텍처 수준에서 검증된다.** WASM Dual Metering(fuel + epoch 독립), 5-Label Taint Tracking, 18-Type Capability, Ed25519 Manifest Signing이 소스코드에서 모두 확인됨. 단, 승인 게이트는 암호학적이 아닌 LLM 프롬프트 규율에 의존한다.

4. **Memory는 Phase 1 의도적 단순화.** SQLite + 코사인 유사도(BLOB f32). 외부 벡터 DB 없음, 하이브리드 검색 없음. 코드 주석에 Phase 2로 Qdrant 마이그레이션 계획이 명시된다.

5. **3-Layer Context 관리는 기존 프레임워크 중 가장 정교하다.** 결과당 30%, Guard 75%, Emergency 4단계 복구가 계층화되어 있다. 50회 max iteration, 5개 동시 LLM 호출 세마포어.

---

## 2. 비교 매트릭스

### 2.1 5개 축 통합 비교 (8 -> 9 프레임워크)

| 프레임워크 | Tool 수 | Security Tier | Memory Tier | Browser | Channel |
|-----------|---------|--------------|------------|---------|---------|
| **OpenFang** | 60 built-in + MCP무제한 | Tier 1+ (16 layers) | Tier 2 (Phase 1) | Native CDP | 40 adapters |
| IronClaw | ~35 | Tier 1 | Tier 1 (pgvector+RRF) | Playwright | 0 |
| ZeroClaw | ~47 | Tier 1 | Tier 1 (FTS5+Soul Snapshot) | 3 backends | 0 |
| OpenClaw | ~25+ | Tier 2 | Tier 1 (LanceDB+MMR+decay) | Playwright+CDP | 0 |
| PicoClaw | 17+MCP | Tier 3 | Tier 2 (MEMORY.md) | 없음 | 0 |
| Nanobot | 9+MCP | Tier 3 | Tier 2 (MEMORY.md+HISTORY.md) | 없음 | 다수 |
| NanoClaw | 10 IPC | Tier 2 | Tier 3 (CLAUDE.md 위임) | Playwright (HOST) | 0 |
| TinyClaw | 0 native | Tier 4 | Tier 3 (write-only) | 없음 | 0 |
| Moltbook | 32 endpoints | Tier 4 | Tier 3 (없음) | 없음 | 0 |

### 2.2 고유 기능 비교

| 기능 | OpenFang | 기존 8개 중 구현체 |
|------|---------|----------------|
| **Hands (자율 능력 패키지)** | [O] 7개 번들 | 없음 |
| **Channel 수** | 40 | 최대 수개 (Nanobot) |
| **Per-channel 모델 오버라이드** | [O] | 없음 |
| **A2A Protocol** | [O] | 없음 |
| **Taint Tracking** | [O] 5-label | 없음 |
| **Ed25519 Manifest Signing** | [O] | 없음 |
| **WASM Dual Metering** | [O] fuel+epoch | IronClaw: fuel만 |
| **Soul Snapshot** | [X] | ZeroClaw만 |
| **Parallel Tool Execution** | [X] | PicoClaw만 |
| **Knowledge Graph** | [O] entity-relation | 없음 |
| **Cost/LLM metering** | [O] 47개 모델 가격표 | 없음 |
| **FangHub Marketplace** | [O] (GitHub 기반) | 없음 |

---

## 3. Tool Architecture 분석

### 3.1 도구 수 및 분류 (총 60개 built-in)

`openfang-runtime/src/tool_runner.rs:487-900+`의 `builtin_tool_definitions()`에서 확인:

| 카테고리 | 도구 | 수 |
|---------|------|---|
| Filesystem | file_read, file_write, file_list, apply_patch | 4 |
| Web | web_fetch, web_search | 2 |
| Shell | shell_exec | 1 |
| Inter-agent | agent_send, agent_spawn, agent_list, agent_kill | 4 |
| Shared Memory | memory_store, memory_recall | 2 |
| Collaboration | agent_find, task_post, task_claim, task_complete, task_list, event_publish | 6 |
| Scheduling | schedule_create, schedule_list, schedule_delete | 3 |
| Knowledge Graph | knowledge_add_entity, knowledge_add_relation, knowledge_query | 3 |
| Media | image_analyze, media_describe, media_transcribe, image_generate | 4 |
| Audio | text_to_speech, speech_to_text | 2 |
| Docker | docker_exec | 1 |
| System | location_get, system_time | 2 |
| Cron | cron_create, cron_list, cron_cancel | 3 |
| Process | process_start, process_poll, process_write, process_kill, process_list | 5 |
| Hands | hand_list, hand_activate, hand_status, hand_deactivate | 4 |
| A2A | a2a_discover, a2a_send | 2 |
| Browser | browser_navigate, browser_click, browser_type, browser_screenshot, browser_read_page, browser_close, browser_scroll, browser_wait, browser_run_js, browser_back, browser_fill | 11 |
| Canvas | canvas_present | 1 |

외부 도구: MCP(25개 pre-configured 통합 + 무제한 커스텀), Skills(WASM/Python/Node 레지스트리)

### 3.2 Tool 인터페이스 설계

IronClaw(Rust trait) · PicoClaw(Go interface)와 달리, OpenFang은 **String 기반 직접 디스패치**를 사용:

```rust
// tool_runner.rs:155
pub async fn execute_tool(
    tool_use_id: &str,
    tool_name: &str,           // 문자열로 매칭
    input: &serde_json::Value,
    // ...
) -> ToolResult
```

Fallback chain: Built-in -> MCP (`mcp_` prefix) -> Skills registry

### 3.3 WASM Sandbox

`wasmtime = "41"` (Cargo.toml:83). Guest ABI:
- Export: `memory`, `alloc(size) -> ptr`, `execute(input_ptr, input_len) -> i64`
- Import module: `"openfang"` -> `host_call(req_ptr, req_len) -> i64`, `host_log`
- 결과: packed i64 = `(result_ptr << 32) | result_len`

`SandboxConfig` 기본값 (sandbox.rs:47-56):
- `fuel_limit`: 1,000,000 instructions
- `max_memory_bytes`: 16 MB (예약; 아직 실 강제 미적용)
- `timeout_secs`: 30초

### 3.4 MCP 양방향 지원

- **Client**: stdio/SSE 두 전송, `mcp_{server}_{tool}` 네임스페이싱, 25개 pre-configured (GitHub, PostgreSQL, Slack, Discord, Jira, AWS, GCP, Azure, MongoDB, Redis, Notion, Linear 등)
- **Server**: `openfang mcp` CLI -> stdio JSON-RPC, `POST /mcp` HTTP endpoint
- MCP 2024-11-05 스펙 준수, 10 MB 메시지 제한, SSRF 보호

### 3.5 A2A 프로토콜 (Google 스펙)

```
GET /.well-known/agent.json   -> Agent Card
POST /a2a/tasks/send          -> Task 제출
GET  /a2a/tasks/{id}          -> Status 폴링
POST /a2a/tasks/{id}/cancel   -> 취소
```

에이전트 Tool -> A2A Skill 자동 변환 (a2a.rs:62-75). 기존 8개 프레임워크 중 A2A 구현체 없음.

### 3.6 병렬 도구 실행 -- 부재

`tool_runner.rs`의 agent loop는 순차 실행. `tokio::join_all` 없음. `subagent_max_concurrent` 필드(tool_policy.rs:49)는 서브에이전트 스폰 깊이 제한이지 도구 병렬화가 아님.

**비교**: PicoClaw만이 goroutine+WaitGroup으로 병렬 도구 실행을 지원한다.

---

## 4. Security 분석

### 4.1 16 Layer 보안 아키텍처 (소스코드 교차검증)

| # | 레이어 | 구현 | 파일:라인 |
|---|--------|------|---------|
| 1 | Capability-Based Access Control | 18개 타입, deny-wins glob 패턴 | `openfang-types/src/capability.rs:106-212` |
| 2 | Approval Gates | 4단계 위험도, 10-300초 타임아웃 | `openfang-types/src/approval.rs:165-280` |
| 3 | WASM Dual Metering | Fuel(1M) + Epoch(30s) 독립 동작 | `openfang-runtime/src/sandbox.rs:170-184` |
| 4 | Information Flow Taint Tracking | 5 label, 3 sink, declassification | `openfang-types/src/taint.rs:13-158` |
| 5 | Ed25519 Manifest Signing | SHA-256 + Ed25519 서명 검증 | `openfang-types/src/manifest_signing.rs:38-107` |
| 6 | SSRF Protection | 9 hostname + 16 IP range 차단 | `openfang-runtime/src/web_fetch.rs:180-228` |
| 7 | Secret Zeroization | `Zeroizing<String>` on all API keys | channels crate |
| 8 | OFP Mutual Authentication | HMAC-SHA256 + nonce replay 방지 | `openfang-wire/src/message.rs:48-53` |
| 9 | Security Headers | CSP, HSTS, X-Frame-Options | `openfang-api/src/routes.rs` |
| 10 | GCRA Rate Limiter | Per-IP 토큰 버킷, cost-aware | `openfang-api/src/` |
| 11 | Path Traversal Prevention | `safe_resolve_path()` 모든 파일 op | `openfang-runtime/src/` |
| 12 | Subprocess Sandbox | `env_clear()` + 화이트리스트 PATH | `openfang-runtime/src/subprocess_sandbox.rs` |
| 13 | Prompt Injection Scanner | 스킬 TOML 오버라이드 패턴 탐지 | `openfang-skills/src/bundled.rs` |
| 14 | Loop Guard | 반복 도구 루프 탐지 | `openfang-runtime/src/loop_guard.rs` |
| 15 | Session Repair | 3단계 LLM 대화 히스토리 복구 | `openfang-runtime/src/` |
| 16 | Health Endpoint Redaction | 공개 `/api/health` 최소 정보만 | `openfang-api/src/routes.rs` |

### 4.2 Taint Tracking -- 기존 8개에 없는 유일한 구현

```rust
// openfang-types/src/taint.rs:13-37
pub enum TaintLabel {
    ExternalNetwork,  // HTTP 응답 바디
    UserInput,        // 직접 사용자 입력
    Pii,              // 개인식별정보
    Secret,           // API 키, 토큰, 패스워드
    UntrustedAgent,   // 비신뢰 에이전트 출력
}

// 3 Sink 차단 규칙
// shell_exec  -> ExternalNetwork, UntrustedAgent, UserInput 차단 (쉘 인젝션 방지)
// net_fetch   -> Secret, Pii 차단 (자격증명 유출 방지)
// agent_message -> Secret 차단 (에이전트 간 시크릿 누출 방지)
```

### 4.3 WASM Dual Metering의 의미

| 시나리오 | Fuel 감지 | Epoch 감지 |
|---------|----------|-----------|
| CPU 집약적 루프 | [O] (명령 수) | [X] |
| Host call blocking (네트워크 지연) | [X] | [O] (벽시계) |
| I/O 대기 | [X] | [O] |
| Host call에서 연료 낭비하는 악성 모듈 | [X] | [O] |

IronClaw의 fuel-only WASM과 달리, epoch watchdog이 I/O 기반 회피를 막는다.

### 4.4 Capability 18개 타입 (privilege escalation 방지 포함)

```rust
// capability.rs:171-187
pub fn validate_capability_inheritance(
    parent_caps: &[Capability],
    child_caps: &[Capability],
) -> Result<(), String> {
    // 부모가 없는 capability를 자식이 요청 -> 즉시 거부
}
```

경제 관련 Capability도 존재: `EconSpend(f64)`, `EconEarn`, `EconTransfer(String)` -- 기존 8개 프레임워크에서 발견된 바 없음.

### 4.5 security_report.md 교차검증

기존 Tier 1 (IronClaw, ZeroClaw) 기준으로 재분류:

| 기준 | IronClaw | ZeroClaw | OpenFang |
|------|---------|---------|------------|
| 암호화 볼트 | [O] AES-256-GCM | [O] ChaCha20 | [X] (env var만) |
| 인젝션 방어 전용 레이어 | [O] SafetyLayer 4중 | [O] PromptGuard 6패턴 | [O] Taint Tracking |
| WASM 샌드박스 | [O] fuel only | [X] Docker | [O] fuel+epoch |
| 승인 게이트 | [O] 도구별 | [O] 3단계 자율성 | [O] (LLM 규율) |
| HITL 암호학적 강제 | [~] (도구 정책) | [O] (E-Stop) | [X] (프롬프트 의존) |

-> OpenFang은 Tier 1과 Tier 2 사이의 새로운 위치. 레이어 수에서는 Tier 1을 초과하지만, 승인 게이트의 강제성(암호학적 아닌 LLM 기반)과 자격증명 격리(볼트 없음)에서 Tier 1에 미치지 못한다.

---

## 5. Memory Architecture 분석

### 5.1 스토리지 백엔드

**SQLite 단독** (rusqlite 크레이트). 외부 벡터 DB 없음. 임베딩은 f32 little-endian BLOB:

```rust
// semantic.rs:331-345
fn embedding_to_bytes(embedding: &[f32]) -> Vec<u8> {
    embedding.iter()
        .flat_map(|&v| v.to_le_bytes())
        .collect()
}
```

### 5.2 메모리 스키마 (v7, 11개 테이블)

| 테이블 | 목적 | 주요 필드 |
|-------|------|---------|
| memories | 에피소딕+시맨틱 | agent_id, content, embedding BLOB, confidence(0.0-1.0), access_count |
| entities | Knowledge Graph 노드 | id, entity_type(JSON), name, properties |
| relations | Knowledge Graph 엣지 | source_entity, relation_type, target_entity, confidence |
| canonical_sessions | 크로스채널 영속 컨텍스트 | agent_id(PK), messages BLOB, compaction_cursor, compacted_summary |
| sessions | 세션별 대화 히스토리 | agent_id, messages BLOB(msgpack), context_window_tokens |
| kv_store | 에이전트 상태 KV | agent_id + key(복합PK), value BLOB(JSON), version |
| usage_events | 비용 추적 | agent_id, model, input_tokens, output_tokens, cost_usd |

### 5.3 검색 알고리즘

```
[쿼리 수신]
    +-- embedding 있음 -> 후보 limit*10 (최소 100) fetch -> 코사인 유사도 순 -> 상위 limit 반환
    +-- embedding 없음 -> content LIKE 검색 -> 정확히 limit 반환
```

**코사인 유사도** (semantic.rs:244-266):
```rust
fn cosine_similarity(a: &[f32], b: &[f32]) -> f32 {
    let (dot, na, nb) = a.iter().zip(b.iter())
        .fold((0.0, 0.0, 0.0), |(d, na, nb), (ai, bi)| {
            (d + ai * bi, na + ai * ai, nb + bi * bi)
        });
    let denom = na.sqrt() * nb.sqrt();
    if denom < f32::EPSILON { 0.0 } else { dot / denom }
}
```

RRF, MMR, decay fusion 없음 -- "Phase 1 correctness-first" 설계.

### 5.4 메모리 Decay

```rust
// consolidation.rs:27-53
// 7일 미접근 메모리 -> confidence 감쇠
// confidence_new = MAX(0.1, confidence_old * (1.0 - decay_rate))
UPDATE memories SET confidence = MAX(0.1, confidence * ?1)
WHERE deleted = 0 AND accessed_at < ?2 AND confidence > 0.1
```

### 5.5 Knowledge Graph -- 기존 8개에 없음

```sql
-- migration.rs:147-169
CREATE TABLE entities (id, entity_type, name, properties, ...);
CREATE TABLE relations (source_entity, relation_type, target_entity, confidence, ...);
```

entities/relations는 **전체 에이전트 공유** (agent_id 컬럼 없음). 에이전트간 지식 공유가 가능하나 격리 불가.

### 5.6 memory_architecture_report.md 교차검증

| 프레임워크 | 벡터 백엔드 | 하이브리드 검색 | Soul Snapshot | Knowledge Graph |
|-----------|-----------|-------------|-------------|----------------|
| OpenFang | SQLite BLOB (Phase 1) | [X] (코사인만) | [X] | [O] entity-relation |
| IronClaw | pgvector | [O] RRF | [X] | [X] |
| ZeroClaw | SQLite+FTS5 | [X] | [O] git 기반 | [X] |
| OpenClaw | LanceDB+sqlite-vec | [O] MMR+decay+weighted | [X] | [X] |

-> OpenFang은 Knowledge Graph를 도입한 유일한 프레임워크. 벡터 검색 성숙도는 Phase 1으로 가장 낮다.

---

## 6. Channel Adapter 분석

### 6.1 40개 채널 어댑터 전체 목록 (검증 완료)

```
Telegram, Discord, WhatsApp, Slack, Signal, Matrix, Teams, Mattermost, Messenger, Viber,
Line, Threema, Bluesky, Mastodon, Reddit, Twitch, LinkedIn, Feishu, DingTalk, Flock,
Guilded, Keybase, Nextcloud, RocketChat, Webex, Zulip, Email, IRC, XMPP(stub), Webhook,
Gitter, Discourse, Gotify, Ntfy, Twist, Pumble, Nostr, Revolt, Google Chat, Mumble
```

**Twitter/X 어댑터 미구현** -- Twitter Hand는 존재하나 채널 어댑터 파일 없음.

### 6.2 브라우저 자동화 -- Native CDP (Playwright 없음)

OpenClaw·NanoClaw가 Playwright를 사용하는 반면, OpenFang은 **Chrome DevTools Protocol 직접 구현** (`browser.rs`, 1309줄):

1. Chromium 프로세스 직접 spawn
2. stderr에서 "DevTools listening on ws://..." 파싱
3. tokio-tungstenite로 CDP WebSocket 연결
4. 명령어: Navigate, Click, Type, Screenshot, ReadPage, Scroll, Wait, RunJs, Back, Fill

세션 영속성: `BrowserManager::sessions: DashMap<AgentId, Arc<Mutex<BrowserSession>>>` -- 인메모리, 재시작 시 소멸.

### 6.3 채널별 고유 기능

**Per-channel 모델 오버라이드** (`openfang-types/src/config.rs:74`):
```toml
[channels.telegram]
model = "claude-opus-4-20250514"  # 이 채널만 다른 모델 사용
rate_limit_per_user = 10          # 분당 10메시지
```

**Rate Limiting** (bridge.rs:205-241):
```rust
struct ChannelRateLimiter {
    buckets: Arc<DashMap<String, Vec<Instant>>>  // key: "{channel_type}:{platform_id}"
}
// 슬라이딩 윈도우 60초, 에이전트 라우팅 이전에 체크
```

### 6.4 browser_actions_report.md 교차검증

| 측면 | OpenFang | OpenClaw | ZeroClaw | NanoClaw |
|------|---------|---------|---------|---------|
| 브라우저 백엔드 | Native CDP | Playwright+CDP | 3 backends | Playwright HOST |
| 브라우저 도구 수 | 11 | 50+ | ~15 | ~8 |
| 채널 어댑터 | 40 | 없음 | 없음 | 없음 |
| 채널별 모델 오버라이드 | [O] | 없음 | 없음 | 없음 |
| 구매 승인 게이트 | [O] (프롬프트) | 없음 | 없음 | 없음 |

---

## 7. Session/Context Management 분석

### 7.1 Agent Loop 아키텍처

`agent_loop.rs:116` `run_agent_loop()` -- 3160줄의 핵심 실행 엔진:

- **Max iterations**: 50 (상수)
- **Tool timeout**: 120초
- **Default context**: 200K 토큰
- **Loop phases**: Thinking -> ToolUse -> Streaming -> Done/Error

### 7.2 3-Layer Context Window 관리 (기존 프레임워크 대비 가장 정교)

```
Layer 1: Per-Result Truncation (context_budget.rs:61)
  -> 결과당 컨텍스트의 30% 상한 (max 50%)
  -> 개행 경계에서 자름 + [TRUNCATED...] 마커

Layer 2: Context Guard (context_budget.rs:99)
  -> 도구 결과가 컨텍스트의 75% 초과 -> 가장 오래된 결과 압축

Layer 3: Emergency Recovery (context_overflow.rs:38)
  -> 70%: 마지막 10개 메시지 유지
  -> 90%: 마지막 4개 메시지 + 마커
  -> 전체 결과 2K chars로 강제 truncate
  -> 최후 수단: Error + /reset 안내
```

### 7.3 세션 영속성

`session.rs:40-100`:
- SQLite WAL 모드, 5초 busy timeout
- MessagePack (rmp-serde) 직렬화
- `INSERT OR CONFLICT DO UPDATE` atomic 저장
- 재시작 후 손 복원: hand_state.json -> kernel.rs:3362-3373

### 7.4 컨텍스트 요약 (compactor.rs)

- **트리거**: 메시지 30개 초과 OR 토큰 추정치 70% 초과
- **전략**: single-pass(오래된 것 요약 + 최근 10개 유지) -> chunked(대용량) -> fallback(단순 truncation)
- **토큰 추정**: chars/4 휴리스틱

### 7.5 듀얼 스케줄러

```
ResourceScheduler (scheduler.rs:44)
  -> 시간당 롤링 윈도우 쿼터 추적
  -> LLM 호출 전 체크 (kernel.rs:1362)

CronScheduler (cron.rs:66)
  -> Job 기반 크론 (cron_jobs.json 영속)
  -> Background agents: Continuous / Periodic / Proactive 3가지 모드

동시성 제한: 5개 LLM 호출 semaphore (background.rs:18)
```

### 7.6 12-Section System Prompt Builder

`prompt_builder.rs:65`에서 빌드:

1. Identity (에이전트 이름/설명)
2. Current Date (시간 인식)
3. Tool Call Behavior
4. AGENTS.md (행동 규칙)
5. Available Tools
6. Memory Protocol
7. Skills (SKILL.md 콘텐츠, 최대 2K chars)
8. MCP Servers
9. Persona (SOUL.md, IDENTITY.md)
10. Bootstrap ritual
11. Heartbeat checklist
12. Peer agents

Canonical Context는 캐시 무효화 방지를 위해 system 프롬프트가 아닌 별도 user 메시지로 주입 (prompt_builder.rs:264).

### 7.7 24/7 자율 실행

```rust
// background.rs:74-91
loop {  // Tokio 무한 루프
    match mode {
        Continuous => { agent.run(); sleep(interval).await; }
        Periodic   => { agent.run_on_cron(pattern); }
        Proactive  => { trigger_engine.wait_for_event().await; }
    }
    if shutdown.is_cancelled() { break; }
}
```

500ms staggered 시작으로 레이트 리밋 폭풍 방지.

---

## 8. Hands System 분석

### 8.1 HAND.toml 완전 스키마

```toml
[root]
id = "researcher"
name = "Researcher Hand"
description = "..."
category = "productivity"  # content/security/productivity/development/communication/data
icon = "researcher"        # 아이콘 식별자
tools = ["web_fetch", "web_search", "memory_store", "memory_recall", ...]
skills = []      # 빈 = 전체 허용
mcp_servers = [] # 빈 = 전체 허용

[[requires]]     # binary | env_var | api_key
key = "ffmpeg"
requirement_type = "binary"
[requires.install]
  macos = "brew install ffmpeg"
  windows = "winget install Gyan.FFmpeg"

[[settings]]     # select | text | toggle
key = "stt_provider"
setting_type = "select"
default = "auto"
[[settings.options]]
  value = "groq_whisper"
  provider_env = "GROQ_API_KEY"

[agent]
name = "researcher-hand"
model = "default"   # "claude-sonnet-4-20250514"
max_tokens = 8192
temperature = 0.3
max_iterations = 80
system_prompt = """500+ word procedural playbook..."""

[[dashboard.metrics]]
label = "Reports Generated"
memory_key = "researcher_reports_count"
format = "number"
```

### 8.2 7개 번들 Hands 요약

| Hand | 카테고리 | 핵심 기능 | Max Iter | Temp |
|------|---------|---------|---------|------|
| Clip | Content | 8-phase 비디오->Shorts 파이프라인, FFmpeg+5 STT 백엔드 | 40 | 0.4 |
| Lead | Data | ICP 기반 리드 발굴+농축+채점(0-100) | 50 | 0.3 |
| Collector | Data | OSINT 연속 모니터링, 지식 그래프 구축 | 60+ | TBD |
| Predictor | Data | 슈퍼포캐스팅, 대립 모드, Brier Score 추적 | 50+ | 0.5 |
| Researcher | Productivity | CRAAP 신뢰도 평가, APA 인용, 다국어 지원 | 80 | 0.3 |
| Twitter | Communication | 7가지 포맷 콘텐츠, 최적 시간 스케줄링, 승인 큐 | 50+ | 0.7 |
| Browser | Productivity | 웹 자동화, 필수 구매 승인 게이트 | 60 | 0.3 |

### 8.3 Hand 생명주기

```
activate(hand_id, config)
  -> HandRegistry.activate()  // HandInstance 생성 (UUID)
  -> resolve_settings()       // 사용자 설정 -> prompt_block + env_vars
  -> Kernel.spawn_agent()     // 에이전트 스폰
  -> registry.set_agent()     // instance_id <-> agent_id 연결
  -> persist_state()          // hand_state.json 저장 (재시작 생존)
      |
pause(instance_id)   -> status = Paused (에이전트 계속 실행, 작업 중단)
resume(instance_id)  -> status = Active
deactivate(instance_id) -> 인스턴스 제거 + 에이전트 킬 + 영속 상태 삭제
```

### 8.4 승인 게이트의 실제 구현 방식

Browser Hand HAND.toml system_prompt (lines 149-156):

```
MANDATORY RULE -- PURCHASE APPROVAL:
Before completing ANY purchase:
1. Summarize items in cart
2. Show total cost
3. List all items clearly
4. STOP COMPLETELY
5. Ask user: "Shall I proceed with this purchase? Total: $X"
6. Wait for explicit approval
NEVER click "Place Order" / "Pay Now" without approval.
```

**중요**: 이것은 암호학적 강제가 아닌 **LLM 프롬프트 규율**이다. IronClaw의 capability gate나 ZeroClaw의 E-Stop과 달리, 충분히 강력한 adversarial prompt로 우회 가능하다.

### 8.5 FangHub -- GitHub API 기반 마켓플레이스

```rust
// skills/marketplace.rs
fn search(query: &str) -> Vec<MarketplaceSkill> {
    // GET /search/repositories?q={query}+org:openfang-skills&sort=stars
}
fn install(skill_name: &str, target_dir: &Path) {
    // GET /repos/openfang-skills/{skill_name}/releases/latest -> tarball 다운로드
}
```

Phase 1: GitHub API 의존. 레이트 리밋 이슈 잠재적.

### 8.6 SKILL.md 주입 메커니즘 -- 중요한 오해 교정

**Hands의 SKILL.md**: `HandDefinition.skill_content`에 저장되나 런타임 주입 안 됨 (개발자 문서).
**Skills의 SKILL.md**: `SkillRegistry` -> `SkillManifest.prompt_context` -> 에이전트 시스템 프롬프트에 최대 2K chars 주입 (실제 런타임 주입).

---

## 9. 벤치마크 주장 검증

README가 주장하는 수치의 소스코드 근거:

| 주장 | 상태 | 근거 |
|------|------|------|
| 137K LOC | [O] 보수적 수치 | 실측 152,966줄 (테스트/주석 포함) |
| 1,767+ 테스트 | [~] 미검증 | 배지 주장; bundled.rs 15+, registry.rs 10+ 테스트 코드 확인 |
| ~32MB 바이너리 | [O] 타당 | Cargo.toml release: `lto=true`, `codegen-units=1`, `strip=true`, `opt-level=3` |
| Cold start <200ms | [WARN] 미검증 | criterion.rs 등 벤치마킹 코드 없음; Rust 단일 바이너리로 타당하나 실측 불가 |
| 40MB idle 메모리 | [WARN] 미검증 | 메모리 프로파일링 코드 없음 |
| 16 Security Systems | [O] 소스 확인 | 모든 16개 레이어 구현 코드 검증 완료 |
| 40 channel adapters | [O] 정확 | 파일 40개 직접 카운트 |
| 27 LLM providers | [~] 타당 | Cargo.toml 의존성 + metering.rs 47개 모델 가격표 확인 |

---

## 10. Cross-validation

### 10.1 security_report.md와 일치/불일치

| 발견 | 이전 보고서 | OpenFang 교차검증 |
|------|-----------|----------------|
| "암호화 볼트 구현은 IronClaw, ZeroClaw뿐" | 유지 | OpenFang은 볼트 없음 (env vars) |
| "HITL을 구현한 곳은 2개" | **수정** | OpenFang도 승인 게이트 있음 (단, LLM 기반) |
| "프롬프트 인젝션 전용 레이어는 2개뿐" | **수정** | OpenFang Taint Tracking이 3번째 구현체 |
| "Tier 1: IronClaw, ZeroClaw" | **확장 필요** | OpenFang은 Tier 1.5 신설 고려 |

### 10.2 memory_architecture_report.md와 일치/불일치

| 발견 | 이전 보고서 | OpenFang 교차검증 |
|------|-----------|----------------|
| "Knowledge Graph: 없음" | **추가** | OpenFang entity-relation SQLite KG |
| "Tier 1: 3개 프레임워크" | 유지 | OpenFang은 Tier 2 (Phase 1 의도적 단순화) |
| "Optimal combo: ZeroClaw Snapshot + OpenClaw hybrid + IronClaw identity" | 유지 | Knowledge Graph 추가 고려 |

### 10.3 browser_actions_report.md와 일치/불일치

| 발견 | 이전 보고서 | OpenFang 교차검증 |
|------|-----------|----------------|
| "4개 프레임워크에 브라우저 자동화" | **확장** | OpenFang이 5번째 (Native CDP) |
| "Playwright가 de facto" | **다양화** | OpenFang은 Playwright 없이 직접 CDP |
| "채널 어댑터는 특정 프레임워크만" | **확장** | OpenFang 40개로 단독 최다 |

### 10.4 session_context_report.md와 일치/불일치

| 발견 | 이전 보고서 | OpenFang 교차검증 |
|------|-----------|----------------|
| "PicoClaw만 병렬 도구 실행" | 유지 | OpenFang은 순차 |
| "24/7 자율 실행 프레임워크" | **추가** | OpenFang이 가장 완전한 24/7 구현 (3모드) |
| "컨텍스트 관리 성숙도" | **추가** | OpenFang 3-layer가 가장 정교 |

---

## 11. 결론 및 신규 오픈 퀘스천

### 11.1 결론

OpenFang은 기존 8개 프레임워크의 **다른 레이어를 경쟁 대상으로 삼는다**. 대화형 에이전트 보조도구가 아닌, 사용자 없이 24/7 작동하는 자율 에이전트 런타임을 목표로 한다. 핵심 차별점:

1. **Hands**: 자율 능력 패키지로서 생명주기(activate/pause/resume/deactivate)와 대시보드 메트릭을 가짐. 기존 8개 프레임워크에서 발견된 어떤 추상화와도 다름.
2. **Channel-first**: 40개 어댑터, 채널별 모델 오버라이드, 슬라이딩 윈도우 레이트 리밋.
3. **Security layering**: 16개 독립 레이어, Taint Tracking, Dual WASM Metering.
4. **Knowledge Graph**: SQLite 기반이지만 entity-relation 구조를 유일하게 도입.

### 11.2 신규 오픈 퀘스천 (Q16~Q20)

**Q16. Hands의 LLM 기반 승인 게이트는 실제 프로덕션에서 얼마나 안전한가?**
Browser Hand의 구매 승인은 프롬프트 규율에 의존한다. 충분히 복잡한 웹페이지가 adversarial prompt injection으로 승인을 우회할 수 있는가? IronClaw의 capability gate · ZeroClaw의 E-Stop과의 안전성 차이는 정량화 가능한가?

**Q17. Knowledge Graph가 메모리 아키텍처에 실질적으로 기여하는가?**
entities/relations가 전체 에이전트 공유(격리 없음)이고 Phase 1에서는 단순 join 쿼리만 지원한다. 실제 용례에서 에이전트간 지식 오염 위험 없이 Knowledge Graph 공유가 가능한가? 에이전트별 격리된 KG가 필요한가?

**Q18. 40개 채널 어댑터의 유지보수 부담은 Rust 단일 바이너리로 감당 가능한가?**
각 어댑터는 독립 구현(예: Discord 29.9 KB, Telegram 30.9 KB)이다. 플랫폼 API 변경 시 업데이트 주기가 어떻게 관리되는가? XMPP가 stub인 것처럼 일부 어댑터의 실질적 완성도 차이는?

**Q19. Phase 2 Qdrant 마이그레이션 시 SQLite BLOB 임베딩 데이터의 이전 경로는?**
코드 주석에 Qdrant 마이그레이션 계획이 명시되어 있다. openfang-migrate 크레이트가 존재하는데, 기존 사용자의 SQLite 임베딩이 어떻게 Qdrant로 이전되는가? 마이그레이션 중단 없이 가능한가?

**Q20. FangHub GitHub API 의존성이 24/7 에이전트 OS의 Hands 배포에 병목이 되는가?**
마켓플레이스가 GitHub API 레이트 리밋에 종속된다. GitHub 장애 시 새 Hands 설치가 불가능하다. 오프라인 환경이나 에어갭 배포 시 FangHub는 어떻게 동작하는가?

---

*분석 완료 일자: 2026-03-08*
*분석 에이전트: 6개 (Tool/Security/Memory/Channel/Session/Hands)*
*참조 파일: 14개 크레이트, docs/ 포함 주요 소스파일 직접 검증*
