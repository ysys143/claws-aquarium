# Hermes Agent 상세 분석 보고서

> **소스**: GitHub [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) 직접 분석
> **조사일**: 2026-03-20
> **언어**: Python

---

## 목차

1. [기본 정보](#기본-정보)
2. [핵심 특징](#핵심-특징)
3. [아키텍처](#아키텍처)
4. [메모리 시스템](#메모리-시스템)
5. [스킬 시스템](#스킬-시스템)
6. [보안 모델](#보안-모델)
7. [위임(Delegation) 아키텍처](#위임delegation-아키텍처)
8. [플랫폼 및 백엔드](#플랫폼-및-백엔드)
9. [RL 및 학습 루프](#rl-및-학습-루프)
10. [차별점 및 신규 패턴](#차별점-및-신규-패턴)
11. [한계](#한계)
12. [참고 링크](#참고-링크)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub** | [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) |
| **Stars** | 9,269 |
| **Forks** | 1,117 |
| **언어** | Python |
| **라이선스** | MIT |
| **생성일** | 2025-07-22 |
| **최종 업데이트** | 2026-03-20 |
| **빌더** | Nous Research |
| **Topics** | ai-agent, anthropic, claude, openclaw, clawdbot, moltbot, hermes, llm |

> **주목**: Topics에 `openclaw`, `clawdbot`, `moltbot` 포함 — Claw 생태계와 직접 연결된 프레임워크

---

## 핵심 특징

> "The self-improving AI agent built by Nous Research. The only agent with a built-in learning loop."

| 특징 | 설명 |
|------|------|
| **자기개선 루프** | 복잡한 작업 완료 후 스킬 자동 생성, 사용 중 스킬 자동 개선 |
| **메모리** | MEMORY.md (에이전트 노트) + USER.md (사용자 프로파일) 이중 저장소 |
| **세션 검색** | SQLite FTS5 + LLM 요약으로 과거 대화 크로스-세션 검색 |
| **위임** | 자식 AIAgent 인스턴스 병렬 스폰, ThreadPoolExecutor |
| **플랫폼** | Telegram, Discord, Slack, WhatsApp, Signal, CLI |
| **터미널 백엔드** | local, Docker, SSH, Daytona, Singularity, Modal (6종) |
| **스케줄러** | 내장 cron 스케줄러 (자연어로 설정) |
| **RL** | Atropos 환경, 궤적 압축, OPD 환경 |
| **ACP 어댑터** | VS Code / Zed / JetBrains IDE 통합 |
| **Honcho** | dialectic 사용자 모델링 통합 |

---

## 아키텍처

```
hermes-agent/
├── run_agent.py           # AIAgent 클래스 — 핵심 대화 루프 (max_iterations=90)
├── model_tools.py         # 도구 오케스트레이션, _discover_tools(), handle_function_call()
├── toolsets.py            # 툴셋 정의, _HERMES_CORE_TOOLS 목록
├── cli.py                 # HermesCLI — 인터랙티브 CLI 오케스트레이터
├── hermes_state.py        # SessionDB — SQLite 세션 저장소 (FTS5 검색)
├── agent/
│   ├── prompt_builder.py      # 시스템 프롬프트 조립
│   ├── context_compressor.py  # 자동 컨텍스트 압축
│   ├── prompt_caching.py      # Anthropic 프롬프트 캐싱
│   ├── auxiliary_client.py    # 보조 LLM 클라이언트 (비전, 요약)
│   ├── smart_model_routing.py # 키워드 기반 cheap/strong 모델 라우팅
│   ├── skill_commands.py      # 스킬 슬래시 명령어 (CLI/gateway 공유)
│   └── trajectory.py          # 궤적 저장 헬퍼
├── tools/                 # 도구 구현 (파일당 1도구, ~44개)
│   ├── registry.py            # 중앙 도구 레지스트리 (스키마, 핸들러, 디스패치)
│   ├── memory_tool.py         # 이중 메모리 저장소 (MEMORY.md + USER.md)
│   ├── skill_manager_tool.py  # 에이전트 자율 스킬 생성/편집/삭제
│   ├── delegate_tool.py       # 하위 에이전트 위임 (병렬, MAX_DEPTH=2)
│   ├── session_search_tool.py # FTS5 세션 검색 + LLM 요약
│   ├── mcp_tool.py            # MCP 클라이언트 (~1050 LOC)
│   ├── tirith_security.py     # pre-exec 보안 스캐너 (외부 바이너리)
│   ├── skills_guard.py        # 스킬 설치 보안 스캐너 (정적 분석)
│   └── environments/          # 터미널 백엔드 6종
├── gateway/               # 메시징 플랫폼 게이트웨이
│   └── platforms/             # Telegram, Discord, Slack, WhatsApp, Signal, HomeAssistant
├── cron/                  # 스케줄러 (jobs.py, scheduler.py)
├── environments/          # RL 훈련 환경 (Atropos)
│   ├── agentic_opd_env.py     # OPD 환경
│   ├── hermes_swe_env/        # SWE 환경
│   └── web_research_env.py    # 웹 리서치 환경
├── acp_adapter/           # IDE 통합 (VS Code/Zed/JetBrains)
├── honcho_integration/    # Dialectic 사용자 모델링
├── batch_runner.py        # 병렬 배치 처리
└── trajectory_compressor.py  # 훈련 데이터용 궤적 압축
```

### 의존성 체인

```
tools/registry.py  (no deps — 모든 도구 파일이 import)
       ↑
tools/*.py  (각각 registry.register() 호출)
       ↑
model_tools.py  (tools/registry import + 도구 디스커버리)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

---

## 메모리 시스템

### 이중 저장소 구조

| 저장소 | 파일 | 용량 제한 | 용도 |
|--------|------|----------|------|
| `memory` | `~/.hermes/memories/MEMORY.md` | 2,200 chars | 에이전트 노트 (환경, 규칙, 발견사항) |
| `user` | `~/.hermes/memories/USER.md` | 1,375 chars | 사용자 프로파일 (선호도, 스타일, 습관) |

> **주목**: 토큰 단위가 아닌 **문자(char) 단위** 제한 — "char counts are model-independent"

### Frozen Snapshot 패턴 (R17)

```python
def load_from_disk(self):
    # 세션 시작 시 스냅샷 캡처 (1회)
    self._system_prompt_snapshot = {
        "memory": self._render_block("memory", self.memory_entries),
        "user": self._render_block("user", self.user_entries),
    }

def format_for_system_prompt(self, target):
    # 항상 frozen snapshot 반환 (live state 아님)
    block = self._system_prompt_snapshot.get(target, "")
    return block if block else None
```

- 세션 시작 시 스냅샷을 **1회만** 캡처
- 세션 중 tool call로 메모리 수정 → 디스크에 즉시 저장, 시스템 프롬프트 **불변**
- 다음 세션에서 스냅샷 갱신
- **효과**: Anthropic prefix cache 전체 세션 유지 → 토큰 비용 절감

### 메모리 주입 탐지 (R19)

```python
_MEMORY_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'you\s+are\s+now\s+', "role_hijack"),
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET)', "exfil_curl"),
    (r'authorized_keys', "ssh_backdoor"),
    # ... 10개 패턴
]
_INVISIBLE_CHARS = {'\u200b', '\u200c', '\u202a', '\ufeff', ...}  # 비가시 유니코드
```

- 메모리 항목 추가/교체 **전** 스캔 실행
- 사용자 입력 + 에이전트 생성 항목 모두 검사
- 비가시 유니코드 탐지 포함

### 원자적 파일 쓰기

```python
# temp file + os.replace() → 독자는 항상 완전한 파일 또는 새 파일 중 하나만 봄
fd, tmp_path = tempfile.mkstemp(dir=str(path.parent), ...)
os.replace(tmp_path, str(path))  # atomic on same filesystem
```

---

## 스킬 시스템

### 개요

- **agentskills.io** 오픈 스탠다드 호환
- `~/.hermes/skills/` — 사용자 스킬 홈디렉토리
- 에이전트가 `skill_manager_tool`로 자율적으로 생성/편집/삭제

### 스킬 구조

```
~/.hermes/skills/
└── my-skill/
    ├── SKILL.md        # 스킬 정의 (절차적 지식)
    ├── references/
    ├── templates/
    ├── scripts/
    └── assets/
```

### Skills Guard 신뢰 수준 (R20)

```python
TRUSTED_REPOS = {"openai/skills", "anthropics/skills"}

INSTALL_POLICY = {
    #                  safe      caution    dangerous
    "builtin":       ("allow",  "allow",   "allow"),
    "trusted":       ("allow",  "allow",   "block"),
    "community":     ("allow",  "block",   "block"),
    "agent-created": ("allow",  "allow",   "block"),
}
```

- 6개 카테고리 정적 분석: exfiltration, injection, destructive, persistence, network, obfuscation
- Hub 설치 + 에이전트 생성 스킬 모두 스캔

### 외부 스킬 플러그인 결합

Hermes는 `~/.hermes/skills/`에 SKILL.md를 놓으면 즉시 사용 가능하므로, 외부 스킬을 **런타임 수정 없이** 설치할 수 있다.

**예: autoresearch-skill 결합**
```bash
# 설치
cp -r autoresearch-skill ~/.hermes/skills/autoresearch/
```
- `skill_manager_tool`이 스킬 수정 실행 담당
- `session_search_tool`(FTS5)이 과거 실험 기록 검색 담당
- Hermes가 **런타임**, autoresearch-skill이 **플러그인** 관계

> autoresearch-skill(olelehmann100kMRR/autoresearch-skill)은 Karpathy autoresearch를 SKILL.md 최적화에 이식한 도구로, Hermes의 스킬 자기개선 기능과 동일한 아이디어를 **명시적 실험 루프(바이너리 eval + 단일 변이 + changelog)**로 구조화한 것이다.

### 메모리 vs 스킬 구분

| 종류 | 저장 위치 | 내용 |
|------|-----------|------|
| `memory` | MEMORY.md | 선언적 지식 (사실, 환경, 규칙) |
| `user` | USER.md | 사용자 프로파일 (선호도) |
| `skill` | ~/.hermes/skills/ | 절차적 지식 (특정 태스크 수행 방법) |

---

## 보안 모델

### Tirith Pre-Exec 스캐너 (R22)

```
터미널 명령 실행 전 → tirith 바이너리 스캔 → 0=allow, 1=block, 2=warn
```

- 탐지: homograph URL, pipe-to-interpreter, terminal injection
- **자동 설치**: PATH에 없으면 GitHub releases에서 자동 다운로드
- **SHA-256 체크섬** 항상 검증
- **cosign 서명 검증** (설치 시, supply chain provenance)
- 설치는 백그라운드 스레드 → 시작 시간 블로킹 없음

### Skills Guard 정적 분석

- 6개 위협 카테고리, severity: critical/high/medium/low
- 신뢰 수준별 차등 정책 (builtin > trusted > community)

### 메모리 인젝션 탐지

- 10개 위협 패턴 + 비가시 유니코드 감지
- add/replace 액션 전 항상 실행

### 위임 차단 도구

```python
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",   # 재귀 위임 불가
    "clarify",         # 사용자 인터랙션 불가
    "memory",          # 공유 MEMORY.md 쓰기 불가
    "send_message",    # 크로스플랫폼 부작용 불가
    "execute_code",    # 코드 실행 불가
])
```

**보안 등급**: Tier 2+ (멀티레이어 보안, 외부 바이너리 스캐너)

---

## 위임(Delegation) 아키텍처

### 구조 제약 (R21)

```python
MAX_CONCURRENT_CHILDREN = 3   # 최대 동시 자식 에이전트
MAX_DEPTH = 2                  # parent(0) → child(1) → 거부(2)
DEFAULT_MAX_ITERATIONS = 50   # 자식당 최대 반복
DEFAULT_TOOLSETS = ["terminal", "file", "web"]
```

- `ThreadPoolExecutor` + `as_completed()` 병렬 실행
- 자식은 격리된 컨텍스트 (부모 히스토리 없음)
- 부모는 delegation call과 요약 결과만 봄 (자식 중간 tool call 노출 없음)

### 세션 검색 아키텍처

```
FTS5 검색 → top N 세션 선택 → 100K chars 트런케이션 → Gemini Flash 요약 → 결과 반환
```

- 비싼 주 모델의 컨텍스트 창 보호
- cheap/fast 보조 LLM으로 요약 처리

---

## 플랫폼 및 백엔드

### 메시징 플랫폼 (6종)

| 플랫폼 | 파일 |
|--------|------|
| Telegram | gateway/platforms/telegram.py |
| Discord | gateway/platforms/discord.py |
| Slack | gateway/platforms/slack.py |
| WhatsApp | gateway/platforms/whatsapp.py |
| Signal | gateway/platforms/signal.py |
| HomeAssistant | gateway/platforms/homeassistant.py |

### 터미널 백엔드 (6종)

| 백엔드 | 특징 |
|--------|------|
| local | 기본 로컬 실행 |
| Docker | 컨테이너 격리 |
| SSH | 원격 서버 |
| Daytona | 서버리스, idle 시 hibernate |
| Singularity | HPC 클러스터 |
| Modal | 서버리스, GPU 지원 |

---

## RL 및 학습 루프

### Atropos 환경

```
environments/
├── agentic_opd_env.py    # OPD (Outcome-Proximal Decomposition) 환경
├── hermes_swe_env/       # SWE-bench 환경
├── web_research_env.py   # 웹 리서치 환경
└── terminal_test_env/    # 터미널 테스트 환경
```

- `batch_runner.py`: 병렬 배치 궤적 생성
- `trajectory_compressor.py`: 훈련 데이터용 궤적 압축
- `rl_cli.py`: RL 명령줄 인터페이스

### Smart Model Routing

```python
_COMPLEX_KEYWORDS = {"debug", "implement", "refactor", "analyze", "delegate", "subagent", ...}

# 복잡한 키워드 감지 → strong model
# URL 감지 → 컨텍스트 기반 선택
# 기본 → cheap model
```

---

## 차별점 및 신규 패턴

### Hermes만의 신규 패턴 (R17–R22)

| 패턴 | 설명 | 비교 |
|------|------|------|
| **R17: Frozen Snapshot Memory** | 세션 시작 시 메모리 스냅샷 1회 캡처, 세션 중 불변. Prefix cache 안정화 | 9개 Claw 모두 세션 중 동적 프롬프트 수정 |
| **R18: Char-Limited Memory** | 토큰이 아닌 문자(char) 단위 예산 (model-agnostic) | 대부분 무제한 또는 토큰 기반 |
| **R19: Memory Injection Detection** | 메모리 항목 추가 전 regex 스캔 (10패턴 + 비가시 유니코드) | OpenFang memory는 injection 검사 없음 |
| **R20: Skills Trust Levels** | builtin/trusted/community/agent-created 4단계 신뢰 정책, agentskills.io 표준 | 10개 프레임워크 중 스킬 신뢰 시스템 유일 |
| **R21: Bounded Delegation Tree** | MAX_DEPTH=2, MAX_CONCURRENT=3, 명시적 blocked tools 목록 | PicoClaw/OpenJarvis 병렬 실행 있지만 depth 제한 없음 |
| **R22: Tirith Pre-Exec Scanner** | 외부 바이너리 스캐너 (SHA-256 + cosign 서명 검증, 자동 설치) | OpenJarvis Prompt Injection Scanner는 regex, Hermes는 외부 바이너리 |

### Claw 패턴과의 비교

| 항목 | Hermes Agent | 가장 유사한 Claw |
|------|-------------|-----------------|
| 메모리 | MEMORY.md + USER.md (frozen snapshot) | ZeroClaw Soul Snapshot (유사하지만 frozen 없음) |
| 스킬 시스템 | agentskills.io + 에이전트 자율 생성 | OpenFang Hands System (HAND.toml) |
| 보안 | Tirith 외부 바이너리 + skills_guard + memory injection | IronClaw (Tier 1 WASM) |
| RL 환경 | Atropos + OPD + SWE | OpenClaw-RL (GRPO) |
| 자기개선 | 스킬 생성 + 스킬 개선 | OpenJarvis (Trace→LoRA) |
| 플랫폼 | 6종 메시징 | NullClaw (19채널) |
| 서버리스 | Daytona + Modal | ZeroClaw ($5/day) |

---

## 한계

1. **Python 런타임 의존**: Zig(NullClaw) 같은 정적 바이너리 없음, 설치 복잡
2. **메모리 용량 제한**: 2,200 + 1,375 chars = 매우 작음 (OpenJarvis SQLite와 대조)
3. **MAX_DEPTH=2 위임 한계**: 깊은 에이전트 계층 불가
4. **Frozen Snapshot의 단점**: 세션 중 메모리 추가 시 같은 세션에서 사용 불가
5. **외부 서비스 의존**: Honcho (dialectic modeling), Daytona, Modal
6. **Tirith 외부 바이너리**: 네트워크 다운로드 필요, air-gapped 환경 어려움

---

## 참고 링크

- GitHub: https://github.com/NousResearch/hermes-agent
- Docs: https://hermes-agent.nousresearch.com/docs/
- agentskills.io: 스킬 오픈 스탠다드
- Honcho: https://github.com/plastic-labs/honcho
- Atropos: RL 훈련 프레임워크
- Tirith: https://github.com/sheeki03/tirith (pre-exec 보안 스캐너)
