# MetaClaw 상세 분석 보고서

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub URL** | https://github.com/aiming-lab/MetaClaw |
| **Stars** | 2,700+ |
| **Forks** | 278 |
| **언어** | Python 97.1%, TypeScript 2.3% |
| **LOC** | ~64,000 (Python 소스 기준) |
| **버전** | v0.4.0 |
| **라이선스** | MIT |
| **개발 팀** | aiming-lab |
| **기술 보고서** | arxiv.org/abs/2603.17187 |
| **설명** | "OpenClaw skill injection and RL training — one-click deployment" |
| **태그라인** | "Just talk to your agent — it learns and EVOLVES" |
| **로컬 경로** | repos_applied/metaclaw/ |

---

## 2. 핵심 특징

MetaClaw는 **에이전트 메타 레이어**다. 자체 LLM 추론 엔진을 가지지 않고, 기존 Claw 런타임(OpenClaw, CoPaw, IronClaw, PicoClaw, ZeroClaw, NanoClaw, NemoClaw, Hermes) 앞에 **OpenAI-compatible 프록시**를 위치시킨다. 이 프록시가 세 가지 기능을 투명하게 수행한다: (1) 스킬을 매 프롬프트에 주입, (2) 장기 메모리를 세션 간에 지속, (3) 대화 데이터를 수집해 RL로 에이전트를 지속 개선.

중요한 점은 GPU 없이 시작 가능하다는 것이다. `skills_only` 모드는 단순 프록시+스킬 주입만 수행하고, RL 훈련은 Tinker 클라우드 LoRA로 아웃소싱해 로컬 GPU를 요구하지 않는다. 어떤 Claw를 쓰든 `metaclaw setup` + `metaclaw start` 두 명령으로 배포된다.

---

## 3. 아키텍처

### 디렉토리 구조

```
metaclaw/
├── metaclaw/                    # 핵심 패키지
│   ├── api_server.py            # FastAPI 프록시 서버 (2124줄)
│   ├── cli.py                   # CLI 인터페이스 + 설정 마법사 (4101줄)
│   ├── claw_adapter.py          # 멀티-Claw 자동 설정 어댑터 (403줄)
│   ├── trainer.py               # GRPO RL 훈련 루프 (642줄)
│   ├── skill_manager.py         # SKILL.md 포맷 스킬 로더 (587줄)
│   ├── skill_evolver.py         # LLM 기반 스킬 자동 생성 (401줄)
│   ├── scheduler.py             # MadMax 유휴 창 스케줄러 (254줄)
│   ├── calendar_client.py       # Google Calendar 통합 (290줄)
│   ├── data_formatter.py        # ConversationSample 포맷 변환
│   ├── openclaw_env_rollout.py  # 롤아웃 데이터 수집 (360줄)
│   ├── prm_scorer.py            # Process Reward Model 스코어러
│   ├── launcher.py              # 프로세스 기동 오케스트레이터 (471줄)
│   ├── config.py                # MetaClawConfig 스키마
│   ├── weaver_compat.py         # Weaver 호환 레이어 (247줄)
│   ├── bedrock_client.py        # AWS Bedrock 클라이언트
│   └── memory/                  # 장기 메모리 서브시스템 (~14,000줄)
│       ├── manager.py           # 메모리 파사드 (5064줄)
│       ├── store.py             # 스토리지 엔진 (1798줄)
│       ├── consolidator.py      # 배경 통합기 (315줄)
│       ├── retriever.py         # keyword/embedding/hybrid 검색 (299줄)
│       ├── policy.py            # 메모리 정책 정의
│       ├── policy_optimizer.py  # 정책 자동 최적화
│       ├── policy_store.py      # 정책 상태 저장
│       ├── promotion.py         # 9-metric 정책 승격 게이트
│       ├── self_upgrade.py      # 자기 업그레이드 로직 (876줄)
│       ├── replay.py            # 리플레이 버퍼 (558줄)
│       ├── embeddings.py        # 임베딩 백엔드 추상화
│       ├── upgrade_worker.py    # 백그라운드 업그레이드 워커 (435줄)
│       ├── metrics.py           # 메모리 품질 지표
│       ├── models.py            # MemoryUnit, MemoryQuery 등
│       ├── scope.py             # 메모리 스코프 (user/project)
│       └── telemetry.py         # 검색 텔레메트리 기록
├── memory_data/                 # 런타임 메모리 + 스킬 저장소
│   └── skills/                  # SKILL.md 포맷 스킬 디렉토리
├── openclaw-metaclaw-memory/    # OpenClaw 플러그인 연동 모듈
├── extensions/                  # 확장 모듈
├── benchmark/                   # 벤치마크 스크립트
├── examples/                    # 사용 예제
└── pyproject.toml               # 패키지 메타데이터 (v0.4.0)
```

### 실행 흐름

```
metaclaw start (--mode madmax/rl/skills_only)
    |
launcher.py -> api_server.py (FastAPI :30000) 기동
    |
claw_adapter.py -> openclaw/ironclaw/... config 자동 패치 + 재시작
    |
클라이언트 요청 (OpenAI-compatible 또는 Anthropic-native)
    |
api_server.py -> skill_manager.py에서 관련 스킬 조회 (키워드/임베딩)
    |
memory/manager.py -> 세션 간 기억 검색 + 프롬프트 주입
    |
대상 Claw 백엔드로 포워딩
    |
응답 수신 -> ConversationSample 생성 (skill_generation 태그 포함)
    |
[skills_only 종료] OR [rl/madmax: 배치 축적 -> 훈련 트리거]
    |
trainer.py (GRPO) -> Tinker 클라우드 LoRA -> 가중치 핫스왑
```

---

## 4. 3가지 운영 모드

| 모드 | 기본값 | 동작 | GPU/Tinker 필요 |
|------|--------|------|----------------|
| `skills_only` | | 프록시 + 스킬 주입. 배치 축적 후 스킬 자동 요약. | 불필요 |
| `rl` | | 스킬 + GRPO 훈련. 배치가 차면 즉시 훈련. | 필요 |
| `madmax` | [기본] | 스킬 + GRPO + 스마트 스케줄러. RL 업데이트는 유휴/수면/회의 창에만 실행. | 필요 |

### MadMax 모드 상세

RL 가중치 핫스왑은 에이전트를 수분간 중단시킨다. MadMax는 `SlowUpdateScheduler`가 `asyncio.Event`(`trigger_event`, `pause_event`)를 통해 `MetaClawTrainer`를 제어한다:

```
사용자 활성 -> pause_event 설정 -> 훈련 대기
사용자 유휴/수면 감지 OR Google Calendar 회의 창 시작 -> trigger_event 설정 -> 훈련 재개
```

Google Calendar API(`calendar_client.py`)로 회의 일정을 사전 조회해 훈련 창을 스케줄링한다.

---

## 5. 멀티-Claw 어댑터 시스템

`claw_adapter.py`는 **플러그형 런타임 스위처**다. 각 Claw별 설정 경로와 재시작 명령을 알고 있고, `metaclaw config claw_type <name>` 한 줄로 백엔드를 교체한다.

| Claw | 설정 방식 | 재시작 명령 |
|------|-----------|-------------|
| openclaw | `openclaw config set ...` | `openclaw gateway restart` |
| copaw | `~/.copaw/config.json` 패치 | 데몬 핫 리로드 |
| ironclaw | `~/.ironclaw/.env` 패치 | `ironclaw service restart` |
| picoclaw | `~/.picoclaw/config.json` 패치 | `picoclaw gateway restart` |
| zeroclaw | `~/.zeroclaw/config.toml` 패치 | `zeroclaw service restart` |
| hermes/nemoclaw/nanoclaw | 동일 패턴 | 각자 재시작 |
| none | 스킵 | (수동 연결) |

새 Claw 추가는 `_configure_<name>` 함수 구현 + `_ADAPTERS` 딕셔너리 등록으로 완결된다.

---

## 6. 스킬 시스템

### SKILL.md 포맷

```
memory_data/skills/
    debug-systematically/
        SKILL.md    <- YAML frontmatter + 마크다운 본문
    code-review/
        SKILL.md
```

```yaml
---
name: debug-systematically
description: Use when diagnosing a bug...
category: coding
---
# Debug Systematically
...
```

카테고리: `general`, `coding`, `research`, `data_analysis`, `security`, `communication`, `automation`, `agentic`, `productivity`, `common_mistakes`

### 스킬 매칭

대화 내용을 키워드 분류기 또는 임베딩 유사도로 태스크 카테고리 탐지 -> 해당 카테고리 스킬을 프롬프트에 주입.

### 스킬 진화 (SkillEvolver)

실패한 `ConversationSample`을 LLM(기본: gpt-5.2)이 분석 -> Claude 스킬 포맷으로 새 스킬 자동 생성:

```python
# 생성 포맷
{"name": str, "description": str, "content": str, "category": str}
```

핵심 메커니즘: 스킬 생성 시 `skill_generation` 버전 번호가 증가하고, **RL 롤아웃 버퍼가 플러시**된다. 진화 이전 샘플과 이후 샘플이 섞이지 않도록 분리 — MAML의 support/query set 분리와 동일한 원리.

---

## 7. 메모리 시스템 (v0.4.0 "Contexture Layer")

v0.4.0(2026-03-25)에서 추가된 장기 메모리 레이어. 스킬이 "어떻게 하는가"를 저장한다면, 메모리는 "무슨 일이 있었나"를 저장한다.

### 구성 요소

| 컴포넌트 | 역할 |
|----------|------|
| `MemoryStore` | SQLite 기반 영구 저장소 |
| `MemoryRetriever` | keyword / embedding / hybrid 3모드 검색 |
| `MemoryConsolidator` | 배경 통합 (유사 기억 병합, 오래된 기억 정리) |
| `MemoryPolicyOptimizer` | 검색 정책 자동 최적화 |
| `MemoryTelemetryStore` | 검색 품질 텔레메트리 누적 |
| `MemoryPolicyStore` | 정책 상태 영구 저장 |

### 정책 승격 게이트 (9-metric)

`promotion.py`의 `should_promote()`는 후보 정책이 기준 정책보다 아래 9개 지표 모두에서 우위여야 승격을 허용한다:

```
avg_query_overlap_delta        >= 0.0
avg_continuation_overlap_delta >= 0.0
avg_response_overlap_delta     >= 0.0
avg_specificity_delta          >= -0.05
avg_focus_score_delta          >= 0.0
avg_value_density_delta        >= 0.0
avg_grounding_score_delta      >= 0.0
avg_coverage_score_delta       >= 0.0
zero_retrieval_delta           <= max_zero_retrieval_increase (기본 2)
```

`min_sample_count=10` 이상 수집돼야 승격 평가 대상이 됨. 모든 13개 Claw 프레임워크 중 메모리 정책을 데이터 기반으로 자동 승격하는 유일한 구현.

### 임베딩 백엔드

- `hashing` 모드 (기본): 의존성 없음, 즉시 사용 가능
- `embedding` 모드: `sentence-transformers` (all-MiniLM-L6-v2)
- `hybrid` 모드: 키워드 + 임베딩 결합

---

## 8. RL 훈련 파이프라인

```
AsyncRolloutWorker (대화 데이터 수집)
    |
MetaClawTrainer.training_loop()
    1. 롤아웃 워커 재개 -> 배치 수집
    2. 롤아웃 워커 일시정지
    3. GRPO-style 어드밴티지 계산 (compute_advantages)
    4. Tinker Datum 포맷 변환 (batch_to_datums)
    5. forward_backward_async -> optim_step_async (back-to-back 비동기)
    6. save_weights_and_get_sampling_client -> 롤아웃 워커에 새 가중치 전달
    7. 롤아웃 워커 재개
    8. (선택) SkillEvolver.evolve() -> 새 스킬 생성 + 버퍼 플러시
```

**Tinker 클라우드 LoRA**: OpenJarvis(로컬 파인튜닝, R11)와 달리 MetaClaw는 훈련을 클라우드 서비스에 위임해 사용자 하드웨어 의존성을 제거한다. `pip install -e ".[rl]"` 시 `tinker` 패키지가 설치된다.

**PRMScorer**: 프로세스 보상 모델. 대화 단계별 품질을 평가해 훈련 신호로 활용. OpenClaw-RL(R6)의 Conversation-to-Gradient와 유사하나 별도 스코어러 모듈로 분리됨.

---

## 9. 신규 패턴 (R-번호)

### **R35: Swap-Runtime Proxy (교체 가능 런타임 프록시)**

단일 프록시 레이어가 7개+ Claw 런타임을 교체 가능한 백엔드로 추상화. `claw_adapter.py`는 각 Claw의 설정 경로와 재시작 명령을 알고 있어, `metaclaw config claw_type <name>` 한 줄 변경으로 백엔드를 교체하고 프록시는 그대로 유지된다.

구현: `repos_applied/metaclaw/metaclaw/claw_adapter.py`
원리: 프록시가 OpenAI-compatible 엔드포인트를 고정 노출하고 백엔드를 플러그로 관리. 스킬/메모리/RL 로직은 프록시 레이어에 귀속되므로 어떤 Claw를 쓰든 동일하게 작동.
시사점: 기능 비교 실험에 이상적 — 동일 스킬/메모리 조건에서 Claw 백엔드만 교체해 성능 A/B 테스트 가능.

### **R36: MadMax 유휴-창 RL 스케줄링**

RL 가중치 핫스왑을 사용자 활성 시간에 실행하지 않고, Google Calendar API + 시스템 유휴 감지로 훈련 창(수면/유휴/회의 시간)을 식별해 그 안에서만 실행.

구현: `repos_applied/metaclaw/metaclaw/scheduler.py` + `calendar_client.py`
원리: `SlowUpdateScheduler`가 `asyncio.Event` 쌍(`trigger_event`/`pause_event`)으로 `MetaClawTrainer`를 제어. 사용자 활성 감지 시 훈련 중단, 유휴 창 진입 시 재개.
시사점: R9(Sleep Consolidation Loop, 30분 메모리 통합)과 다른 관심사 — R9은 메모리 품질 유지, R36은 RL 훈련 타이밍 최적화. 둘 다 인간 수면 패턴 활용이라는 공통점.

### **R37: 실패 기반 스킬 진화 + MAML 버퍼 분리**

실패한 대화 샘플을 LLM이 분석해 새 스킬을 자동 생성한 뒤, `skill_generation` 버전 번호를 증가시키고 RL 롤아웃 버퍼를 플러시한다. 진화 전 데이터와 진화 후 데이터가 혼재하지 않도록 분리 — MAML의 support/query set 구분과 동일한 원리.

구현: `repos_applied/metaclaw/metaclaw/skill_evolver.py` + `data_formatter.py`
원리: `ConversationSample.skill_generation`이 스킬 세대를 추적. `SkillEvolver.evolve()` 호출 시 세대 번호 증가 + 이전 세대 샘플 무효화.
시사점: R11(OpenJarvis Trace->LoRA)이 가중치 레벨 자기 개선이라면, R37은 스킬 공간 레벨 자기 개선. 두 루프를 결합하면 (a) 스킬 진화 -> 새 스킬로 더 좋은 대화 -> (b) 더 좋은 대화로 LoRA 훈련 가속 -> (a)로 피드백하는 이중 루프.

---

## 10. 비교 테이블

| 기능 | MetaClaw | OpenJarvis | Hermes Agent | OpenClaw-RL |
|------|----------|------------|--------------|-------------|
| RL 훈련 | GRPO + Tinker LoRA | 로컬 LoRA (R11) | Atropos/OPD | GRPO (R6) |
| 스킬 시스템 | SKILL.md + LLM 자동 생성 (R37) | 없음 | agentskills.io 신뢰 정책 (R20) | 없음 |
| 메모리 | SQLite + 9-metric 정책 승격 | 3-tier 벡터 DB | MEMORY.md Frozen Snapshot (R17) | 없음 |
| 멀티-런타임 지원 | 7개 Claw 교체 (R35) | 없음 | 없음 | 없음 |
| 훈련 스케줄링 | MadMax 유휴 창 (R36) | 없음 | 없음 | 배치 즉시 실행 |
| GPU 필요 여부 | 불필요 (LoRA 아웃소싱) | 필요 | 선택적 | 필요 |
| 보안 Tier | 미분류 (응용 레이어) | Tier 2+ | Tier 2+ | 해당 없음 |

---

## 11. 한계

- **프록시 단일 장애점**: MetaClaw 프록시가 다운되면 모든 Claw 기능 중단. 고가용성 구성 없음.
- **Tinker 클라우드 의존성**: RL 훈련이 외부 Tinker 서비스에 종속. 셀프호스팅 GPU 환경 미지원.
- **보안 위임**: MetaClaw 자체에는 샌드박스, 암호화, 프롬프트 인젝션 방어 없음. 하위 Claw 런타임의 보안에 전적으로 의존.
- **단일 사용자 로컬**: 로컬 프록시 구조로 멀티테넌시 미지원. 팀/서버 환경 부적합.
- **스킬 진화 LLM 비용**: `SkillEvolver`는 외부 OpenAI-compatible API 필요. 오프라인 환경 미지원.
- **메모리 스코프 제한**: `scope_id` 기반 scoping이지만 사용자 간 격리 메커니즘 미흡.

---

## 12. 참고 링크

- **GitHub**: https://github.com/aiming-lab/MetaClaw
- **기술 보고서**: https://arxiv.org/abs/2603.17187
- **관련 보고서**:
  - OpenJarvis (로컬 LoRA, R11): `reports/repos/details/openjarvis_report.md`
  - Hermes Agent (Skills Trust, R17-R22): `reports/repos/details/hermes_agent_report.md`
  - OpenClaw-RL (Conversation-to-Gradient, R6): `reports/repos_research/openclaw_rl_report.md`
  - Autoresearch (Fixed-Budget Loop, R3): `reports/repos_research/research_tools_report.md`
