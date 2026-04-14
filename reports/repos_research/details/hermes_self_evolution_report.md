# Hermes Agent Self-Evolution 상세 분석 보고서

> **소스**: NousResearch/hermes-agent-self-evolution (ICLR 2026 Oral)
> **분석 대상**: Hermes Agent 외부 자동 최적화 파이프라인
> **작성 일자**: 2026-04-14
> **로컬 경로**: `repos_research/hermes-agent-self-evolution/`

---

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub URL** | https://github.com/NousResearch/hermes-agent-self-evolution |
| **Stars** | 1,700 |
| **Forks** | 149 |
| **Watchers** | 16 |
| **언어** | Python 100% |
| **라이선스** | MIT © 2026 Nous Research |
| **버전** | 0.1.0 |
| **Python 요구** | ≥3.10 |
| **학술 검증** | ICLR 2026 Oral (`reports/phase1_validation_report.pdf`) |
| **Hermes Agent와의 관계** | **외부 도구** — "operates ON hermes-agent — not part of it" (README L5). `HERMES_AGENT_REPO` 환경변수로 대상 레포 지정 |
| **GPU 요구** | **없음** — 전부 API 호출 기반. LoRA/파인튜닝 아님 |
| **운영 비용** | **$2–10 / 최적화 런** |
| **설명** | "Evolutionary self-improvement for Hermes Agent — optimize skills, prompts, tool descriptions, and code using DSPy + GEPA" |

---

## 2. 비전과 포지셔닝

**"Hermes Agent의 스킬·프롬프트·도구 설명·코드를 반영적(reflective) 진화 탐색으로 체계적으로 개선하는 독립 최적화 파이프라인"**

핵심 주장: 전통적 RL은 **"무엇이 실패했는지"**만 알 수 있는 스칼라 보상에 의존하는 반면, GEPA는 **execution trace를 읽어 "왜 실패했는지"를 이해**하고 타겟을 두고 변이(mutation)를 생성한다. 3개의 예시만으로도 작동하며 기존 DSPy 최적화기와 RL을 모두 능가한다고 명시.

**No GPU training required.** 이 플랜의 모든 요소가 API 호출로만 동작. 프롬프트·명령어·few-shot 예제의 **텍스트를 변이·평가**하는 것이지 모델 가중치를 건드리지 않음. DSPy의 유일한 가중치 훈련 컴포넌트(`BootstrapFinetune`)는 **플랜에서 명시적으로 제외**.

### Hermes Agent 본체와의 경계

| 측면 | Hermes Agent 본체 | Self-Evolution |
|------|------------------|----------------|
| **레포** | NousResearch/hermes-agent | NousResearch/hermes-agent-self-evolution |
| **역할** | 에이전트 런타임 | 외부 최적화 도구 |
| **통신** | — | `HERMES_AGENT_REPO` env var로 경로 참조 |
| **산출물** | 에이전트 세션 로그(`session_db`) | Hermes Agent 레포에 **Pull Request** 제출 |
| **자동 커밋** | — | **금지** (모든 변경은 인간 리뷰 필수) |

---

## 3. 3개 엔진 아키텍처

| 엔진 | 최적화 대상 | 라이선스 | 통합 방식 |
|------|------------|---------|----------|
| **DSPy + GEPA** | Skills, prompts, instructions, tool descriptions | MIT | 네이티브 Python, **주력 엔진** |
| **Darwinian Evolver** | Code files, algorithms, tool implementations | AGPL v3 | **외부 CLI만** (라이선스 격리) |
| **DSPy MIPROv2** | Few-shot examples, instruction text | MIT | 네이티브 Python, 폴백 옵티마이저 |

**GEPA가 스타**: DSPy에 통합되어 있으며 execution trace를 읽어 실패 원인 이해. 3개 예시로 작동. RL과 이전 DSPy 최적화기 모두 능가한다고 주장.

**Darwinian Evolver가 외부 CLI로 격리된 이유**: AGPL v3 라이선스 감염 방지. 코드 진화는 Git 기반 "organism" 모델로 별도 프로세스에서 실행.

### 디렉토리 구조 (실측)

```
hermes-agent-self-evolution/
├── evolution/
│   ├── core/
│   │   ├── config.py
│   │   ├── constraints.py          # 5-layer guardrails
│   │   ├── dataset_builder.py      # session_db + synthetic mining
│   │   ├── external_importers.py
│   │   └── fitness.py               # evaluation metrics
│   ├── skills/
│   │   ├── evolve_skill.py          # Phase 1 엔트리포인트
│   │   └── skill_module.py          # DSPy Module wrapper
│   ├── tools/                       # Phase 2 자리 (현재 __init__만)
│   ├── code/                        # Phase 4 (Darwinian)
│   ├── monitor/                     # Phase 5 연속 루프
│   └── prompts/                     # Phase 3 시스템 프롬프트
├── datasets/
│   ├── skills/                      # 스킬별 평가 데이터
│   └── tools/                       # 도구 선택 평가
├── reports/
│   └── phase1_validation_report.pdf # ICLR 2026 논문
├── tests/
├── generate_report.py
├── PLAN.md                          # 781 LOC 설계 명세
└── pyproject.toml
```

**의존성 (`pyproject.toml` 실측)**:
```toml
dependencies = [
  "dspy>=3.0.0",
  "openai>=1.0.0",
  "pyyaml>=6.0",
  "click>=8.0",
  "rich>=13.0",
]
[project.optional-dependencies]
darwinian = ["darwinian-evolver"]  # 선택적 — AGPL 격리
```

---

## 4. 4-Tier 최적화 대상

`PLAN.md`에 명시된 **리스크/가치 매트릭스**:

| Tier | 대상 | 가치 | 리스크 | 구현 상태 | 엔진 |
|------|------|------|--------|----------|------|
| **1** | Skill files (SKILL.md) | Highest | Lowest | **✅ Implemented** | DSPy + GEPA |
| **2** | Tool descriptions (schema `description` field) | Medium | Low | 🔲 Planned | DSPy + GEPA |
| **3** | System prompt components (persona/policies/formatting) | High | Higher (prompt caching 파손 위험) | 🔲 Planned | DSPy + GEPA |
| **4** | Tool implementation code | High | Highest | 🔲 Planned | Darwinian Evolver |

### Tier 3의 특수성 — Prompt Caching 제약

> "Must be careful not to break prompt caching — only optimize offline, deploy as new versions"

Anthropic prefix cache 보존이 Hermes Agent의 핵심 최적화 축(R17 Frozen Snapshot)이므로, 시스템 프롬프트 최적화는 **오프라인에서만** 수행하고 배포는 버전 번호를 올리는 형태로 처리.

---

## 5. 최적화 루프 + 5-Layer 가드레일

### 표준 루프 (7단계)

```
1. SELECT TARGET       → 스킬/프롬프트 섹션/도구 선택, baseline 로드
2. BUILD EVAL DATASET  → session_db 실사용 또는 합성 데이터, train/val/test 분할
3. WRAP AS DSPy MODULE → Skill text → dspy.Signature / Workflow → dspy.ReAct / Selection → dspy.Predict
4. RUN OPTIMIZER       → 주력 dspy.GEPA (reflective evolution) / 폴백 dspy.MIPROv2 (Bayesian) / 코드 Darwinian CLI
5. EVALUATE & COMPARE  → held-out test에서 정확도·비용·레이턴시 대조, 통계적 유의성 검증
6. CONSTRAINT GATES    → 5겹 가드레일 (아래)
7. PR SUBMIT           → hermes-agent 레포에 Pull Request, 인간 리뷰
```

### 5-Layer 가드레일 (README 원문)

모든 진화된 변형(variant)은 반드시 5개를 모두 통과해야 함:

1. **전체 테스트 스위트**: `pytest tests/ -q` 100%
2. **크기 제한**: Skills ≤15KB, tool descriptions ≤500 chars
3. **캐싱 호환**: Mid-conversation 변경 금지
4. **의미 보존**: 원본 목적에서 드리프트 금지
5. **인간 PR 리뷰**: 모든 변경은 인간 리뷰, **절대 직접 커밋 없음**

### 평가 데이터 소스 (2가지)

```bash
# 합성 (빠른 반복)
python -m evolution.skills.evolve_skill --skill X --eval-source synthetic

# 실제 세션 히스토리 (Claude Code, Copilot, Hermes 등에서 마이닝)
python -m evolution.skills.evolve_skill --skill X --eval-source sessiondb
```

`core/dataset_builder.py`가 두 소스를 통합 처리.

---

## 6. 기존 Claw 자기 개선 패턴과의 비교

| 차원 | R11: OpenJarvis Trace→LoRA | R13: OpenJarvis AgentConfigEvolver | R37: MetaClaw SkillEvolver | **Self-Evolution (R47 후보)** |
|------|---------------------------|-----------------------------------|---------------------------|-----------------------------|
| **최적화 레이어** | **모델 가중치** (LoRA) | TOML 설정 파일 | Skill 공간 (LLM 생성) | **텍스트 (프롬프트/스킬/도구 설명)** + Tier 4 코드 |
| **GPU 필요** | ✅ (LoRA 파인튜닝) | ❌ | ❌ | **❌ ($2–10 API only)** |
| **변이 메커니즘** | SFT pairs → 파인튜닝 | 쿼리 클래스별 최적 조합 탐색 | 실패 샘플 → LLM 즉흥 생성 | **GEPA 반영적 진화 + execution trace 원인 분석** |
| **피드백 신호** | feedback≥0.7 필터 | traces 분석 | 실패 샘플 | **왜 실패했는가 (execution trace reasoning)** |
| **안전 장치** | eval gate (개선폭≥0.02) | .history/ 롤백 | MAML 버퍼 분리 | **5-layer 가드레일 + 인간 PR 리뷰 필수** |
| **적용 범위** | 단일 에이전트 모델 | config 파일 1개 | Skill 1개/turn | **4-Tier (스킬/도구desc/시스템프롬프트/코드)** |
| **학술 검증** | — | — | — | **ICLR 2026 Oral** |
| **운영 방식** | 내장 루프 | 내장 루프 | 내장 루프 | **외부 독립 레포, PR 제출** |

### 핵심 차별점 3가지

1. **텍스트-레벨 + No-GPU** — R11(LoRA)은 GPU 필요, R37(MetaClaw)은 LLM 즉흥 생성. Self-Evolution은 **DSPy Signature/Module 포맷에 text를 래핑해 GEPA로 변이**. API 호출만으로 $2–10에 최적화 완료.
2. **Execution Trace 기반 원인 분석** — R37은 실패 샘플에서 **새 스킬을 생성**하지만, GEPA는 실패 trace를 읽어 **기존 스킬의 어느 부분이 왜 문제인지 추론**해 국소 변이. 반영적(reflective) 최적화.
3. **인간 PR 리뷰 필수 + 외부 독립 도구** — R11/R13/R37은 모두 **런타임 내부 자동화 루프**. Self-Evolution은 명시적으로 **"PR against hermes-agent"** 구조를 채택해 에이전트가 자기 소스를 자동 변경하지 못하게 설계. 감사 가능한 외부 파이프라인.

---

## 7. 신규 패턴 제안 — R47

### R47: Reflective Text-Level Evolution with PR-Gated Deploy

**정의**: 에이전트의 텍스트 자산(스킬/도구 설명/시스템 프롬프트/코드)을 execution trace 원인 분석 기반 GEPA로 진화시키되, GPU 가중치 훈련 없이 API 호출만으로 $2–10에 최적화하고, 5겹 가드레일 + 인간 PR 리뷰 게이트를 통과해야 배포되는 자기 개선 패턴.

**구현**:
- 텍스트 래핑: `evolution/skills/skill_module.py` → DSPy Module
- 변이 엔진: `dspy.GEPA` (주력), `dspy.MIPROv2` (폴백), `darwinian-evolver` (코드 외부 CLI)
- 평가 데이터: `evolution/core/dataset_builder.py` → session_db 마이닝 + 합성 생성
- 가드레일: `evolution/core/constraints.py` → 5-layer (pytest + size + cache + semantic + PR)
- 배포 게이트: `generate_report.py` → PR 본문 생성, 인간 리뷰

**원리**: "진화 = 변이 + 선택" 공식에서 **변이 단계의 지능화**에 초점. execution trace를 읽어 "왜 틀렸는지"를 이해하는 LLM-reasoning 기반 mutation operator가 랜덤 mutation 대비 수렴 속도를 단축. 평가·선택은 표준 evolutionary search.

**시사점**:
- 런타임 내장형 자기 개선(R11/R13/R37)은 감사성·되돌림이 약함 — R47은 **외부 PR 파이프라인**으로 이를 보완
- GPU 없는 소규모 팀/개인도 사용 가능 — 연구실 스킬셋·개인 에이전트에 직접 적용 가능
- DSPy + GEPA가 오픈 소스(MIT)라 Claw 생태계 전반으로 이식 가능성 큼

**R11/R13/R37과의 관계**: 상호 배타적 아님. R47의 텍스트 최적화 + R11의 LoRA 가중치 업데이트가 합쳐지면 **완전한 자기 개선 루프**가 됨 (텍스트 → 가중치 → 텍스트…).

---

## 8. 사용자 개인 에이전트 시스템과의 접점

`project_user_personal_agent.md`의 4-에이전트 파이프라인(Monitoring/Research/Writing/Publisher) 관점에서:

| 적용 지점 | 잠재 효용 |
|----------|----------|
| **Monitoring Agent 스킬** | 6개월 운영 후 session_db에서 "실험 가치 판단" 실패 케이스 마이닝 → GEPA로 판단 기준 프롬프트 최적화 |
| **Research Agent 스킬** | 벤치마크 실패(잘못된 도구 선택, 재현 실패)를 trace로 읽어 도구 선택 로직 개선 |
| **Writing Agent 스킬** | 사용자 피드백("서론 길어") 트레이스 → 문체 프롬프트 Tier 3 최적화 (prompt caching 주의 필요) |
| **Publisher Agent 스킬** | 업로드 실패 패턴 마이닝 → 포맷 변환 스킬 진화 |

**비용 예상**: 4개 스킬 × $2–10 × 분기 1회 최적화 = **분기당 $8–40** — 개인 에이전트 운영에서 무시 가능한 수준.

**전제**: Hermes Agent를 Research Agent(혹은 전 파이프라인)에 채택했을 때 즉시 딸려오는 부가 효능. 클론만 해두고 `HERMES_AGENT_REPO` 설정하면 즉시 사용 가능.

---

## 9. 한계

1. **Phase 1만 구현** — Tier 2(도구 설명), Tier 3(시스템 프롬프트), Tier 4(코드)는 PLAN.md상 설계만 존재. 4-Tier 전부가 돌아가는 것은 아님.
2. **session_db 의존** — 실 사용 로그가 없으면 합성 데이터에만 의존. 합성 데이터의 representativeness가 검증되지 않으면 최적화 방향이 왜곡 가능.
3. **DSPy 3.0+ 의존** — DSPy API 변화에 종속. DSPy 2.x 코드베이스나 비Python 프레임워크에는 이식 어려움.
4. **코드 진화의 AGPL 격리 부담** — Darwinian Evolver가 AGPL v3라 CLI로만 분리. Tier 4 활성화 시 라이선스 관리 복잡도 증가.
5. **PR 리뷰 병목** — 인간 리뷰 필수는 안전하지만, 많은 스킬을 동시 진화할 때 리뷰 부담 누적. 리뷰 자동 요약 도구 부재.
6. **Hermes Agent 특화** — 다른 Claw 프레임워크(OpenClaw, Claude Code)에는 그대로 적용 불가. 범용화는 포팅 작업 필요.

---

## 10. 참고 링크

- **GitHub**: https://github.com/NousResearch/hermes-agent-self-evolution
- **Hermes Agent 본체**: https://github.com/NousResearch/hermes-agent
- **DSPy**: https://github.com/stanfordnlp/dspy
- **GEPA**: https://github.com/gepa-ai/gepa
- **Darwinian Evolver**: https://github.com/imbue-ai/darwinian_evolver
- **Phase 1 검증 논문**: `repos_research/hermes-agent-self-evolution/reports/phase1_validation_report.pdf` (ICLR 2026 Oral)
- **내부 문서**: `PLAN.md` (781 LOC 완전 설계 명세)
- **관련 보고서**:
  - `reports/repos/details/hermes_agent_report.md` — Hermes Agent 본체 분석 (R17–R22)
  - `reports/repos_applied/details/metaclaw_report.md` — R37 SkillEvolver 비교 대상
  - `reports/repos_research/details/deepinnovator_report.md` — R1–R5 연구 자동화 대비
  - `reports/repos_research/details/autoresearch_report.md` — R3 Fixed-Budget Loop 대비

---

**최종 수정**: 2026-04-14
