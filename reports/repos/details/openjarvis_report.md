# OpenJarvis 심층 분석 보고서 — 로컬 우선 개인 AI 프레임워크

> **조사 일자**: 2026-03-14
> **조사 방법**: GitHub API를 통한 소스코드 직접 분석 (agent, learning, security, scheduler, sessions, intelligence 전체)
> **대상 레포**: `repos/openjarvis/` (open-jarvis/OpenJarvis, squash subtree)
> **배경**: Stanford Hazy Research + Scaling Intelligence Lab의 연구 프레임워크. "Personal AI, On Personal Devices."
> **핵심 질문**: "OpenJarvis는 기존 9개 프레임워크 대비 어떤 차별점을 갖는가? 특히 로컬 학습 루프와 Intelligence Per Watt 메트릭은 24시간 에이전트 설계에 무엇을 시사하는가?"

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. [비교 매트릭스 (9 + 1 프레임워크)](#2-비교-매트릭스)
3. [Agent Architecture 분석](#3-agent-architecture-분석)
4. [Learning Loop 분석](#4-learning-loop-분석)
5. [Security 분석](#5-security-분석)
6. [Intelligence / Model Catalog 분석](#6-intelligence--model-catalog-분석)
7. [Scheduler 분석](#7-scheduler-분석)
8. [Session Management 분석](#8-session-management-분석)
9. [기존 보고서와의 Cross-validation](#9-cross-validation)
10. [신규 패턴 (R10–R14)](#10-신규-패턴-r10r14)
11. [결론 및 열린 질문](#11-결론-및-열린-질문)

---

## 1. Executive Summary

OpenJarvis는 스스로를 **"로컬 우선 개인 AI 스택"**이라 명명한다. Stanford의 두 연구실(Hazy Research, Scaling Intelligence Lab)이 주도하며, 핵심 명제는 단순하다: **"클라우드 API는 필요할 때만. 개인 기기에서 실행되는 AI가 기본이어야 한다."**

**가장 주목할 발견 5가지:**

1. **Trace→LoRA 로컬 자기개선 루프가 처음 등장한다.** 상호작용 traces를 수집하고, 품질 필터링 후 SFT pairs를 추출하고, 에이전트 TOML config를 자동 진화시키며, 선택적으로 LoRA 파인튜닝을 수행한다. Eval gate(최소 2% 개선)가 없으면 자동 reject. 9개 Claw 중 어느 것도 로컬 파인튜닝 루프를 구현하지 않는다.

2. **RLM Agent는 긴 컨텍스트를 프롬프트가 아닌 REPL 변수로 저장한다.** arxiv:2512.24601 기반. `llm_query()` / `llm_batch()`로 재귀적 서브-LM 호출, 임의 길이 입력을 코드로 분해·처리. 9개 Claw의 컨텍스트 관리와 근본적으로 다른 접근법.

3. **Intelligence Per Watt가 새로운 평가 축이다.** 정확도 외에 에너지·FLOPs·레이턴시·비용을 1등급 제약조건으로 측정. "로컬 모델이 88.7%의 단일 턴 쿼리를 처리 가능"이 연구 데이터로 존재한다. 9개 Claw 중 ZeroClaw만 $5/day 비용 한도를 갖는데, OpenJarvis는 하드웨어 효율 자체를 메트릭화한다.

4. **AgentConfigEvolver가 에이전트 설정을 자동 진화시킨다.** traces를 분석해 쿼리 클래스별로 최적 에이전트·도구 목록·max_turns를 도출하고 TOML 파일로 저장(.history/ 버전 관리 + rollback 지원). 다음 세션은 진화된 config로 실행된다.

5. **OrchestratorAgent의 structured mode가 훈련 포맷과 정렬된다.** `THOUGHT:/TOOL:/INPUT:/FINAL_ANSWER:` 텍스트 프로토콜이 에이전트 실행 형식인 동시에 SFT/GRPO 훈련 데이터 형식이다. 에이전트가 실행될수록 그 자체로 훈련 데이터가 된다.

---

## 2. 비교 매트릭스

### 2.1 5개 축 통합 비교 (9 → 10 프레임워크)

| 프레임워크 | 에이전트 수 | Security Tier | Memory Tier | 학습 루프 | 로컬 우선 |
|-----------|------------|--------------|------------|---------|---------|
| **OpenJarvis** | 7종 | Tier 1~2 | Tier 2 | **LoRA 파인튜닝** | **[O] 핵심 설계 원칙** |
| IronClaw | 1 | Tier 1 | Tier 1 (pgvector+RRF) | [X] | [X] 클라우드 API |
| ZeroClaw | 1 | Tier 1 | Tier 1 (FTS5+Soul Snapshot) | [X] | [X] 클라우드 API |
| OpenClaw | 1 | Tier 2 | Tier 1 (LanceDB+MMR+decay) | GRPO (클라우드) | [X] |
| PicoClaw | 1 | Tier 3 | Tier 2 (MEMORY.md) | [X] | [X] |
| Nanobot | 1 | Tier 3 | Tier 2 (MEMORY.md) | [X] | [X] |
| NanoClaw | 1+sub | Tier 2 | Tier 3 (CLAUDE.md 위임) | [X] | [X] |
| TinyClaw | 1 | Tier 4 | Tier 3 (write-only) | [X] | [X] |
| OpenFang | 1+Hands | Tier 1+ (16 layers) | Tier 2 | [X] | [X] |
| always-on (Google ADK) | 4 (외부 참조) | [X] | Tier 2 (SQLite LLM-only) | [X] | [X] |

### 2.2 고유 기능 비교

| 기능 | OpenJarvis | 기존 9개 중 구현체 |
|------|-----------|----------------|
| **로컬 LoRA 파인튜닝** | [O] | 없음 |
| **Trace→Config 자동 진화** | [O] AgentConfigEvolver | 없음 |
| **RLM (REPL 컨텍스트)** | [O] arxiv:2512.24601 | 없음 |
| **Intelligence Per Watt 메트릭** | [O] | 없음 |
| **하드웨어 자동 감지 + 추론 백엔드 선택** | [O] Ollama/vLLM/SGLang/llama.cpp | 없음 |
| **훈련 포맷 = 에이전트 포맷 정렬** | [O] structured mode | 없음 |
| **Cron/Interval 스케줄러** | [O] TaskScheduler | 없음 |
| **A2A 프로토콜** | [O] | OpenFang만 |
| **RBAC Capability 10종** | [O] Python+Rust 이중 구현 | IronClaw(18종), OpenFang(18종) |
| **Taint Tracking** | [O] 4-label | OpenFang(5-label) |
| **Prompt Injection Scanner** | [O] regex 기반 | 없음 (명시적) |
| **Cross-channel Session** | [O] channel_ids 매핑 | OpenClaw, OpenFang |
| **Soul Snapshot** | [X] | ZeroClaw만 |
| **벡터 검색** | [X] | IronClaw, ZeroClaw, OpenClaw |

---

## 3. Agent Architecture 분석

### 3.1 7종 에이전트 타입

```
AgentRegistry에 등록된 에이전트들:
├── simple          — 단순 Q&A, 단일 턴
├── react           — ReAct 루프 (Thought→Action→Observation)
├── orchestrator    — 멀티 턴 tool-calling (function_calling + structured 모드)
├── rlm             — Recursive Language Model (REPL 기반)
├── native_react    — ReAct 네이티브 구현체
├── native_openhands— OpenHands 네이티브 통합
├── claude_code     — Claude Code runner 통합
├── openhands       — OpenHands 프레임워크 연결
└── monitor_operative — 모니터링 전용
```

### 3.2 OrchestratorAgent — function_calling vs structured

OrchestratorAgent는 두 가지 실행 모드를 지원한다:

**function_calling 모드** (기본):
- OpenAI 포맷 tool definitions 사용
- `tool_calls` 응답 파싱
- `parallel_tools=True`시 ThreadPoolExecutor로 병렬 실행
- LoopGuard: 반복 tool call 탐지 + context 압축

**structured 모드** (훈련 정렬):
```
THOUGHT: 현재 상황을 분석하면...
TOOL: web_search
INPUT: {"query": "..."}
Observation: [도구 결과]
FINAL_ANSWER: 최종 응답
```
이 포맷이 SFT/GRPO 훈련 데이터 형식과 동일하다. **에이전트를 실행하면 그 자체로 훈련 데이터가 생성된다.**

### 3.3 RLM Agent (R12)

```python
# 전통적 방법: 긴 컨텍스트를 프롬프트에 직접 주입
prompt = f"다음 10만자 문서를 요약해라:\n{long_document}"  # 컨텍스트 폭발

# RLM 방법: 컨텍스트를 REPL 변수로 저장
repl.set_variable("context", long_document)
# LLM이 Python 코드를 작성해 context를 처리
# llm_query(chunk) 로 재귀적 서브-LM 호출
# FINAL(answer) 로 종료
```

**핵심 구조**:
- `RLMRepl`: 영속 Python REPL (json, re, math, collections 등 허용)
- `llm_query(prompt)`: 단일 서브-LM 호출 (도구 실행 1라운드 포함)
- `llm_batch(prompts)`: 다중 서브-LM 병렬 호출
- `FINAL(value)` / `FINAL_VAR(var_name)` / `answer["ready"]=True`: 종료 신호

**적용 시나리오**: 수만 자 문서 요약, 코드베이스 전체 분석, 긴 대화 히스토리 처리.

### 3.4 병렬 도구 실행

OrchestratorAgent의 `parallel_tools=True` (기본값):

```python
with concurrent.futures.ThreadPoolExecutor(max_workers=len(tool_calls)) as pool:
    futures = {pool.submit(_exec_tool, tc): tc for tc in tool_calls}
    # LoopGuard 체크 후 병렬 실행
```

PicoClaw(goroutine+WaitGroup)에 이어 두 번째로 병렬 도구 실행을 구현한 프레임워크.

---

## 4. Learning Loop 분석

OpenJarvis의 학습 루프는 기존 9개 Claw에서 발견된 어떤 패턴과도 다르다. **상호작용이 곧 훈련 데이터가 되고, 모델이 스스로 개선된다.**

### 4.1 전체 파이프라인

```
사용자 상호작용
    │
    ▼
TraceStore (SQLite)
    │ traces 저장 (steps, feedback score, outcome)
    ▼
LearningOrchestrator.run()
    ├── TrainingDataMiner.extract_sft_pairs()     → SFT (input, output) pairs
    ├── TrainingDataMiner.extract_routing_pairs() → 쿼리 클래스별 최적 모델
    └── TrainingDataMiner.extract_agent_config_pairs() → 최적 에이전트·도구
    │
    ├── [Eval] baseline_score = eval_fn()         (선택)
    │
    ├── AgentConfigEvolver.analyze()              → TOML config 진화
    │   └── .history/ 버전 관리 + rollback 지원
    │
    ├── [선택] LoRATrainer.train(sft_pairs)        (torch + min 10 pairs)
    │
    └── [Eval] post_score = eval_fn()
        └── improvement >= 0.02 → accept / reject
```

### 4.2 TrainingDataMiner

품질 필터링 기준:
- `feedback >= 0.7` (기본)
- `outcome == "success"`

3가지 추출 모드:

| 모드 | 출력 | 용도 |
|------|------|------|
| `extract_sft_pairs` | `{input, output, tools_used, agent}` | LoRA 파인튜닝 |
| `extract_routing_pairs` | 클래스별 최고 성능 모델 | 모델 라우터 업데이트 |
| `extract_agent_config_pairs` | 클래스별 최적 에이전트+도구 | AgentConfigEvolver 입력 |

### 4.3 AgentConfigEvolver (R13)

traces를 분석해 쿼리 클래스(code, search, reasoning 등)별로 최적 설정을 도출하고 TOML 파일로 저장:

```toml
# 자동 생성된 agent config 예시
[agent]
type = "orchestrator"
max_turns = 8

[tools]
enabled = ["web_search", "code_interpreter", "file_read"]
```

- `.history/` 디렉토리에 이전 버전 자동 보관
- rollback: 이전 버전으로 복원 가능
- Autoresearch의 `train.py` 코드 수정과 유사하지만 **config 레벨** 자동화

### 4.4 LoRA 파인튜닝

```python
# LearningOrchestrator 설정
orchestrator = LearningOrchestrator(
    trace_store=store,
    config_dir="./configs",
    lora_config=LoRATrainingConfig(...),
    model_name="Qwen/Qwen3-0.6B",  # 기본값: 로컬 소형 모델
    min_sft_pairs=10,               # 최소 훈련 데이터 수
    min_improvement=0.02,           # 최소 eval 개선폭
)
```

조건: `torch` 설치 + `min_sft_pairs` 이상 + `min_improvement` 충족.
충족 안 되면 config 진화만 진행하고 LoRA는 skip.

**Autoresearch와의 비교**: Autoresearch는 코드(train.py)를 수정하고 실행 결과(val_bpb)로 accept/reject. OpenJarvis는 상호작용 traces에서 SFT pairs를 추출하고 eval 개선폭으로 accept/reject. 자동화 원칙은 동일하나 **훈련 데이터 출처**가 다르다.

---

## 5. Security 분석

### 5.1 보안 레이어 구성

```
src/openjarvis/security/
├── capabilities.py    — RBAC 10종 capability
├── taint.py           — 정보 흐름 제어 (4-label taint)
├── injection_scanner.py — 프롬프트 인젝션 탐지 (regex)
├── audit.py           — 감사 로그
├── file_policy.py     — 파일 접근 정책
├── guardrails.py      — 안전 가드레일
├── rate_limiter.py    — 속도 제한
├── scanner.py         — 통합 스캐너
├── signing.py         — (서명 기능)
├── ssrf.py            — SSRF 방지
└── subprocess_sandbox.py — 서브프로세스 격리
```

### 5.2 RBAC Capability (10종)

```python
class Capability(str, Enum):
    FILE_READ      = "file:read"
    FILE_WRITE     = "file:write"
    NETWORK_FETCH  = "network:fetch"
    CODE_EXECUTE   = "code:execute"
    MEMORY_READ    = "memory:read"
    MEMORY_WRITE   = "memory:write"
    CHANNEL_SEND   = "channel:send"
    TOOL_INVOKE    = "tool:invoke"
    SCHEDULE_CREATE= "schedule:create"
    SYSTEM_ADMIN   = "system:admin"
```

**Python + Rust 이중 구현**:
```python
self._rust_impl = _rust.CapabilityPolicy(default_deny=default_deny)
# grant/deny 호출 시 Python dict + Rust impl 동시 업데이트
```

기본값: `default_deny=False` (허용 기본). `default_deny=True`로 거부 기본 전환 가능.

**IronClaw(18종) / OpenFang(18종)과 비교**: 종류는 적으나 `schedule:create`처럼 OpenJarvis 고유 능력이 포함됨.

### 5.3 Taint Tracking (4-label)

```python
class TaintLabel(str, Enum):
    PII          = "pii"      # 개인식별정보
    SECRET       = "secret"   # 비밀/자격증명
    USER_PRIVATE = "user_private"
    EXTERNAL     = "external" # 외부 출처 데이터

# Sink 정책: 특정 도구에 taint된 데이터 전달 금지
SINK_POLICY = {
    "web_search":        {TaintLabel.PII, TaintLabel.SECRET},
    "channel_send":      {TaintLabel.SECRET},
    "code_interpreter":  {TaintLabel.SECRET},
}
```

자동 탐지 패턴: SSN, 이메일, 신용카드, 전화번호.

OpenFang(5-label: PII/SECRET/USER_PRIVATE/EXTERNAL + AGENT_GENERATED)과 비교하면 AGENT_GENERATED 레이블이 없다.

### 5.4 Prompt Injection Scanner

정규식 기반, 4단계 위협 수준(LOW/MEDIUM/HIGH/CRITICAL):

| 패턴 | 위협 수준 |
|------|---------|
| "ignore all previous instructions" | HIGH |
| "you are now a different AI" | HIGH |
| `execute("...")`  / `eval("...")` | HIGH |
| shell injection (`;`, `\|`, `&&` + 명령어) | HIGH |
| 데이터 탈취 시도 (send to https://) | HIGH |

**9개 Claw 중 명시적 Prompt Injection Scanner를 구현한 첫 번째 프레임워크.**

### 5.5 보안 평가

| 항목 | OpenJarvis | 비고 |
|------|-----------|------|
| RBAC Capability | [O] 10종, Python+Rust | IronClaw/OpenFang 대비 종류 적음 |
| Taint Tracking | [O] 4-label | OpenFang과 유사 |
| Prompt Injection | [O] regex scanner | 9개 Claw 중 유일 |
| SSRF 방지 | [O] ssrf.py | 명시적 구현 |
| 서브프로세스 격리 | [O] subprocess_sandbox.py | |
| 감사 로그 | [O] audit.py | |
| WASM 격리 | [X] | IronClaw만 |
| Ed25519 서명 | signing.py (확인 필요) | OpenFang 대비 미확인 |
| Docker 컨테이너 | [X] | NanoClaw, OpenClaw |

---

## 6. Intelligence / Model Catalog 분석

### 6.1 Intelligence Per Watt (R10)

OpenJarvis의 배경 연구 [intelligence-per-watt.ai]는 다음을 제시한다:

- 로컬 모델이 **88.7%의 단일 턴 쿼리를 처리 가능** (클라우드 불필요)
- 모델 지능 효율이 **2023→2025 사이 5.3배 향상**
- 평가 지표: 정확도 + **에너지 (Watt)** + **FLOPs** + **레이턴시** + **달러 비용**

이 연구가 프레임워크 설계 원칙의 근거가 된다: "클라우드는 진짜 필요할 때만."

### 6.2 Model Catalog

```python
# 로컬 Dense 모델
ModelSpec(model_id="qwen3:8b",   parameter_count_b=8.2,  context_length=32768)
ModelSpec(model_id="qwen3:32b",  parameter_count_b=32.0, min_vram_gb=20.0)

# 로컬 MoE 모델 (active params 별도 표기)
ModelSpec(model_id="qwen3.5:3b",  active_parameter_count_b=0.6, context_length=131072)
ModelSpec(model_id="qwen3.5:8b",  active_parameter_count_b=1.0, context_length=131072)
ModelSpec(model_id="qwen3.5:14b", active_parameter_count_b=2.0, context_length=131072)
# + 클라우드 모델 (GPT-4, Claude 등 fallback)
```

`min_vram_gb` 필드가 있어 하드웨어 제약을 모델 선택에 반영한다.

### 6.3 Learned Router

traces 기반으로 쿼리 클래스(code/search/reasoning/chat)별 최적 모델을 학습:

```
쿼리 입력 → classify_query() → 클래스 결정 → LearnedRouter → 최적 모델 선택
```

ZeroClaw의 소형/대형 모델 라우팅과 유사하지만 **클래스 분류 + trace 기반 학습**이 추가됨.

---

## 7. Scheduler 분석

### 7.1 TaskScheduler

```python
scheduler = TaskScheduler(store, system, poll_interval=60)
scheduler.start()  # daemon thread 시작

# 세 가지 스케줄 타입
scheduler.create_task("리포트 생성", "cron",     "0 9 * * 1")   # 매주 월요일 9시
scheduler.create_task("메모리 정리", "interval", "3600")         # 1시간마다
scheduler.create_task("온보딩",     "once",     "2026-04-01T09:00:00Z")
```

**실행 흐름**:
```
daemon thread (poll_interval=60s)
    └── _poll_loop()
        └── store.get_due_tasks(now)
            └── _execute_task(task)
                ├── system.ask(prompt, agent=..., tools=...)
                └── store.log_run(started_at, finished_at, success, result)
```

- 상태 관리: active / paused / cancelled / completed
- cron 파싱: `croniter` 사용, 없으면 기본 파서 fallback
- 실행 로그: started_at, finished_at, success, result, error 저장

### 7.2 "24시간 상주" 관점에서

TaskScheduler는 백그라운드 daemon thread로 동작하지만, **요청 처리기에 스케줄러가 붙은 구조**다. always-on-memory-agent처럼 에이전트가 항상 처리 중인 것이 아니라, 예약된 시간에 에이전트를 깨우는 방식. OpenClaw-RL의 상시 학습 루프와도 다르다.

그러나 **Learning Loop + Scheduler 조합**은 가능하다:
```python
# 매일 자정에 학습 루프 실행
scheduler.create_task("일일 학습 루프", "cron", "0 0 * * *")
# 1시간마다 config 진화 체크
scheduler.create_task("config 진화", "interval", "3600")
```

---

## 8. Session Management 분석

### 8.1 Cross-channel Session

```python
@dataclass
class SessionIdentity:
    user_id: str
    display_name: str
    channel_ids: Dict[str, str]  # channel_type → channel_user_id
    # 예: {"slack": "U12345", "telegram": "@user", "web": "sess_abc"}
```

하나의 사용자 정체성이 여러 채널을 통합한다. OpenFang(40개 채널 어댑터)과 목표는 같지만 구현 규모가 다르다.

### 8.2 Session Store (SQLite)

```python
SessionStore(
    max_age_hours=24.0,               # 24시간 후 만료
    consolidation_threshold=100,      # 100메시지 후 통합
)
```

- consolidation: 100 메시지 임계값 (Nanobot과 동일한 숫자)
- decay: max_age_hours 기반 자동 정리
- cross-channel: channel_ids dict로 다채널 사용자 통합

---

## 9. Cross-validation

### 9.1 memory_architecture_report.md 와의 연결

| Q11 (consolidation 주기) | OpenJarvis 데이터포인트 |
|------------------------|----------------------|
| Nanobot: 100 메시지 | SessionStore: 100 메시지 (동일) |
| ZeroClaw: 12h | - |
| OpenClaw: 5초 debounce | - |
| always-on: 30분 (수면 비유) | - |
| **OpenJarvis**: 24시간 session 만료 + 100 메시지 consolidation | 시간+분량 이중 기준 |

### 9.2 security_report.md 와의 연결

| Tier 기준 | OpenJarvis 위치 |
|----------|---------------|
| Tier 1 (WASM/Docker+proxy) | WASM [X], Docker [X] → Tier 1 제외 |
| Tier 2 (컨테이너, 도구 허용 목록) | subprocess_sandbox + RBAC → **Tier 2** |
| Tier 3 (인-프로세스, deny 패턴) | — |

**Tier 2 상단** 배치. Prompt Injection Scanner는 9개 Claw 중 유일한 구현.

### 9.3 research_tools_report.md 와의 연결

| 패턴 | OpenJarvis 구현 |
|------|--------------|
| R3: Fixed-Budget Loop (Autoresearch) | eval gate + min_improvement 유사 |
| R11: Trace→LoRA 로컬 파인튜닝 | 직접 구현 |
| R13: AgentConfigEvolver | 직접 구현 |

Autoresearch가 "코드 수정→실행→val_bpb 비교→accept/reject"라면, OpenJarvis는 "상호작용→traces→SFT→LoRA→eval 비교→accept/reject". **자동화 철학은 동일, 훈련 데이터 출처가 다름.**

### 9.4 openclaw_rl_patterns_analysis.md 와의 연결

| 축 | OpenClaw-RL | OpenJarvis |
|-----|------------|-----------|
| 훈련 방식 | GRPO (policy gradient) | LoRA SFT |
| 데이터 출처 | 다음 턴 신호 (PRM 판정) | 피드백 점수 (사용자 평가) |
| 실행 환경 | 클라우드 GPU (Ray+SGLang) | 로컬 기기 |
| 비동기성 | 완전 비동기 (4-component) | 동기 루프 (LearningOrchestrator) |

---

## 10. 신규 패턴 (R10–R14)

### R10: Intelligence Per Watt 메트릭

**정의**: 정확도 외에 에너지·FLOPs·레이턴시·달러 비용을 1등급 평가 지표로 취급. "충분히 좋은 성능"에서 클라우드 대신 로컬 모델을 선택하는 의사결정 근거.

**적용**: 모델 카탈로그의 `min_vram_gb`, `parameter_count_b`, `active_parameter_count_b` 필드가 하드웨어 제약을 반영. 24시간 상주 에이전트 설계 시 "항상 켜져 있는 비용"을 측정하는 틀을 제공.

**9개 Claw와 차이**: ZeroClaw는 $5/day 예산 한도(사후 제한). OpenJarvis는 에너지·FLOPs를 사전 설계 메트릭으로 사용(사전 최적화).

---

### R11: Trace→LoRA 로컬 자기개선 루프

**정의**: 상호작용 traces → 품질 필터링 → SFT pairs 추출 → LoRA 파인튜닝 → eval gate → accept/reject.

**구조**:
```
TraceStore → TrainingDataMiner → LoRATrainer → eval_fn → accept/reject
               (feedback >= 0.7)   (torch 필요)  (개선폭 >= 0.02)
```

**유사 패턴과의 차이**:
- Autoresearch R3: 코드 수정 → 실행 → BPB 비교 (훈련 코드 자동화)
- OpenClaw-RL R6: 다음 턴 신호 → GRPO (실시간 RL)
- OpenJarvis R11: 상호작용 축적 → SFT LoRA (지연 학습)

---

### R12: RLM Agent — Context-as-Variable

**정의**: 긴 입력을 프롬프트에 직접 주입하는 대신 Python REPL 변수로 저장. LLM이 코드를 작성해 컨텍스트를 재귀적으로 분해·처리.

**구조**:
```python
repl.set_variable("context", long_input)
# LLM → Python 코드 생성 → REPL 실행
# llm_query(chunk) / llm_batch(chunks) → 재귀 서브-LM 호출
# FINAL(answer) → 종료
```

**적용 시나리오**: 수만 자 문서, 코드베이스 전체, 긴 대화 히스토리. 컨텍스트 창 폭발 없이 임의 길이 처리.

---

### R13: AgentConfigEvolver — Config 자동 진화

**정의**: traces 분석 → 쿼리 클래스별 최적 에이전트·도구·파라미터 도출 → TOML config 자동 갱신 (버전 관리 + rollback 포함).

**구조**:
```
traces → classify_query() → 클래스별 그룹핑 →
    최고 성능 agent + tools + max_turns 집계 →
        .history/ 백업 → 새 TOML 저장
```

**Autoresearch R3과의 차이**: Autoresearch는 코드(train.py) 수정. OpenJarvis는 설정(TOML config) 수정. 수정 범위와 위험도가 다르다.

---

### R14: 훈련 포맷 = 에이전트 포맷 정렬

**정의**: OrchestratorAgent의 structured mode 실행 포맷(`THOUGHT:/TOOL:/INPUT:/FINAL_ANSWER:`)이 SFT/GRPO 훈련 데이터 포맷과 동일.

**함의**: 에이전트를 structured mode로 운용하면 실행 로그가 그대로 훈련 데이터가 된다. 별도의 데이터 수집 파이프라인 불필요. **에이전트 운용과 모델 훈련이 동일한 루프 안에 있다.**

---

## 11. 결론 및 열린 질문

### 핵심 결론

1. **로컬 파인튜닝 루프는 9개 Claw와 완전히 다른 차원이다**: 기존 9개는 모두 "모델을 고정하고 도구와 메모리로 에이전트를 향상". OpenJarvis는 "에이전트 사용이 모델 자체를 개선". 이것이 OpenJarvis를 단순 프레임워크가 아닌 연구 플랫폼으로 만드는 핵심.

2. **Intelligence Per Watt는 24시간 에이전트 설계에 필수 메트릭이다**: 항상 켜져 있는 에이전트는 비용·에너지가 누적된다. "클라우드 API는 필요할 때만"이라는 철학은 ZeroClaw의 $5/day 한도보다 더 근본적인 접근.

3. **RLM은 긴 컨텍스트 문제의 다른 해법이다**: OpenFang의 3-Layer Context Management(Guard 75% 임계값), ZeroClaw의 3일 윈도우와는 다른 접근. 컨텍스트를 압축하거나 자르는 대신 **코드로 분해**.

4. **AgentConfigEvolver + LoRA 조합은 아직 실험적이다**: min_sft_pairs=10 조건과 torch 의존성이 있어 실제 로컬 기기에서 동작하려면 상당한 상호작용 데이터 축적이 선행되어야 한다.

5. **보안은 Tier 2이나 Prompt Injection Scanner는 독보적이다**: WASM/Docker 격리는 없지만, regex 기반 주입 탐지 + Taint Tracking + RBAC 조합은 9개 Claw 중 가장 다층적인 소프트 보안.

### 신규 열린 질문

**Q31. LoRA 파인튜닝의 실효성**: 로컬 기기의 연산 제약 하에 min_sft_pairs=10으로 의미 있는 개선이 가능한가? Qwen3-0.6B 기본값은 실용적인 선택인가?

**Q32. AgentConfigEvolver rollback 트리거**: config 진화 후 성능이 저하됐을 때 자동 rollback 로직이 있는가? 현재 코드는 accept/reject를 eval_fn에 위임하는데, eval_fn이 없으면 항상 accept.

**Q33. RLM의 보안 경계**: REPL에서 `json, re, math` 등은 허용하지만 `os, sys, subprocess`는 막아야 한다. 현재 제한 목록이 충분한가? 샌드박스 없이 REPL을 실행하면 탈출 가능한가?

**Q34. Intelligence Per Watt 측정 방법론**: 에너지(Watt)를 실제로 측정하는가, 아니면 FLOPs에서 추정하는가? 하드웨어 다양성(M2 Mac vs x86 GPU)에서 일관된 측정이 가능한가?

**Q35. 훈련 포맷 정렬의 실용성**: structured mode가 SFT 데이터 생성을 자동화하지만, function_calling mode 대비 응답 품질 차이는? 사용자 경험을 희생하고 훈련 데이터를 얻는 트레이드오프가 합리적인가?

---

## 부록: 코드 참조 인덱스

### 핵심 파일

| 파일 | 기능 |
|------|------|
| `src/openjarvis/agents/orchestrator.py` | OrchestratorAgent (function_calling + structured mode, 병렬 도구) |
| `src/openjarvis/agents/rlm.py` | RLMAgent (REPL 기반 재귀 LM, arxiv:2512.24601) |
| `src/openjarvis/agents/rlm_repl.py` | RLMRepl (영속 REPL, llm_query/llm_batch 콜백) |
| `src/openjarvis/learning/learning_orchestrator.py` | 전체 학습 루프 오케스트레이터 |
| `src/openjarvis/learning/training/data.py` | TrainingDataMiner (SFT/routing/agent-config 추출) |
| `src/openjarvis/learning/training/lora.py` | LoRATrainer |
| `src/openjarvis/learning/agents/agent_evolver.py` | AgentConfigEvolver (TOML 진화 + .history/) |
| `src/openjarvis/learning/routing/learned_router.py` | 쿼리 클래스별 모델 라우터 |
| `src/openjarvis/security/capabilities.py` | RBAC 10종 (Python+Rust 이중 구현) |
| `src/openjarvis/security/taint.py` | Taint Tracking 4-label + SINK_POLICY |
| `src/openjarvis/security/injection_scanner.py` | Prompt Injection Scanner (regex, 4 위협 수준) |
| `src/openjarvis/scheduler/scheduler.py` | TaskScheduler (cron/interval/once daemon) |
| `src/openjarvis/sessions/session.py` | SessionStore (SQLite, cross-channel, consolidation) |
| `src/openjarvis/intelligence/model_catalog.py` | ModelSpec 카탈로그 (Qwen3, MoE, VRAM 제약) |
| `src/openjarvis/intelligence/router.py` | 지능형 모델 라우터 |

### 에이전트 타입 등록

| 에이전트 | registry 키 | 파일 |
|---------|------------|------|
| Simple | `simple` | `simple.py` |
| ReAct | `react` | `react.py` |
| Orchestrator | `orchestrator` | `orchestrator.py` |
| RLM | `rlm` | `rlm.py` |
| Native ReAct | `native_react` | `native_react.py` |
| Claude Code | `claude_code` | `claude_code.py` |
| OpenHands | `openhands` | `openhands.py` |
| Monitor Operative | `monitor_operative` | `monitor_operative.py` |

---

*본 보고서는 repos/openjarvis/를 10번째 프레임워크로 추가하며 작성된 최초 분석이다. 기존 9개 Claw와의 비교는 session_context_report.md, security_report.md, memory_architecture_report.md, research_tools_report.md를 교차 참조.*
