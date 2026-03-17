# DeepInnovator 상세 분석 보고서

> **소스**: `reports/repos_research/research_tools_report.md` §§3-4에서 추출
> **분석 대상**: HKUDS/DeepInnovator — RL 기반 연구 아이디어 생성
> **작성 일자**: 2026-03-09

---

## 목차

1. [기본 정보](#기본-정보)
2. [에이전트 아키텍처 (7개 에이전트)](#1-에이전트-아키텍처-7개-에이전트)
3. [YAML 설정 스키마](#2-yaml-설정-스키마)
4. [오케스트레이션 루프](#3-오케스트레이션-루프)
5. [Authenticity Discriminator](#4-authenticity-discriminator-고유-혁신)
6. [계층적 메모리 시스템](#5-계층적-메모리-시스템)
7. [보상/훈련 시스템](#보상훈련-시스템)
8. [코드 참조 인덱스](#코드-참조-인덱스)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **저자** | HKUDS (홍콩대) |
| **언어** | Python |
| **규모** | ~105K LOC, 402 파일 |
| **목적** | RL 기반 연구 아이디어 생성 모델 훈련 |
| **에이전트 수** | 7개 (YAML 설정) |
| **도구** | MCP + Sandbox + 검색 |
| **기억** | JSON 파일 계층적 (4-layer) |
| **보상** | GRPO + Delta Reward + Token Amount |
| **루프** | 데이터 준비 파이프라인 (4단계) |

---

## 1. 에이전트 아키텍처 (7개 에이전트)

DeepInnovator는 학술 논문을 분석하여 연구 아이디어를 생성하는 **계층적 멀티에이전트 파이프라인**이다.

| # | 에이전트 | 파일 | 역할 | 모델 |
|---|---------|------|------|------|
| 1 | **Paper Analyzer** | `paper_analyzer.yaml` | 논문에서 구조화된 정보 추출 (제목, 요약, 도메인, 핵심 발견, 방법론, 한계, 향후 연구) | model_set |
| 2 | **Paper Router** | `paper_router.yaml` | 논문을 기존 메모리 그룹에 라우팅하거나 새 그룹 생성 결정. 내용 기반 매칭 (키워드 아님) | model_set |
| 3 | **Paper Idea Spark** | `paper_idea_spark.yaml` | 기존 연구로부터 다음 아이디어 합성. **깊이 있는 합성** 원칙 (A+B+C 단순 조합 금지) | idea_spark_model_set (더 큰 모델) |
| 4 | **Serendipity Engine** | `idea_serendipity_engine.yaml` | 크로스 도메인 우연의 연결 발견. surprise_factor 0-1 스케일 | big_model_set (가장 큰 모델) |
| 5 | **Paper Group Creator** | `paper_group_creator.yaml` | 새 메모리 그룹 생성 (소문자 하이픈 ID) | model_set |
| 6 | **Paper Group Updater** | `paper_group_updater.yaml` | 기존 그룹에 논문 매칭, 설명 업데이트 | model_set |
| 7 | **Paper Connections + Research Trending** | `idea_paper_connections.yaml`, `idea_research_trending.yaml` | 논문 간 연결 (5가지 유형) + 트렌드 신호 합성 (4단계 성숙도) | model_set |

---

## 2. YAML 설정 스키마

각 에이전트 YAML은 **완전한 에이전트 지침**을 포함한다:

```yaml
# 공통 구조 (예: paper_analyzer.yaml)
model: model_set              # 모델 세트 이름
system_prompt: |              # 완전한 시스템 프롬프트
  You are a research paper analyst...
output_schema:                # 기대 출력 JSON 스키마
  paper_title: string
  paper_summary: string
  research_domain: [string]
  key_findings: [string]
  methodology: string
  limitations: [string]
  future_work: [string]
  confidence: float (0-1)
```

**모델 할당 전략**: 작업 복잡도에 따라 다른 크기의 모델 사용
- 기본 작업 (분석, 라우팅, 그룹 관리): `model_set`
- 아이디어 합성: `idea_spark_model_set` (더 큰 모델)
- 크로스 도메인 발견: `big_model_set` (가장 큰 모델)

---

## 3. 오케스트레이션 루프

### 데이터 준비 파이프라인 (Step 1-4)

```
입력: 학술 논문들
  |
[Step 1] Paper -> JSON Lines (format_paper_content)
  - 단순화된 순차 ID (paper_1, paper_2, ...) + 양방향 매핑
  |
[Step 2] Paper Routing & Analysis (순차)
  - Paper Analyzer: 각 논문 구조화 분석
  - Paper Router: 기존 메모리/새 그룹 결정
  - Creator/Updater: 그룹 생성 또는 업데이트
  |
[Step 3] Insight Generation (병렬 가능)
  - Paper Connections: 논문 간 연결 (5가지 유형)
  - Serendipity Engine: 예상치 못한 크로스 도메인 연결
  - Research Trending: 트렌드 신호 (nascent->emerging->accelerating->mature)
  - 각각 3회 재시도 메커니즘
  |
[Step 4] Next Research Idea Synthesis
  - Paper Idea Spark: 아이디어 생성
  - Authenticity Discriminator: 진정성 검증
  |
출력:
  - inner_paper_memory.json   (개별 논문 분석)
  - inter_paper_group.json    (그룹화)
  - connections.json          (논문 간 연결)
  - serendipity.json          (우연의 연결)
  - trending.json             (트렌드 신호)
  - ideas.json                (생성된 아이디어)
```

### 런타임 Agent Loop (DeepInnovatorAgentLoop)

```
[PENDING] -> [GENERATING] -> [PROCESSING_TOOLS] -> [INTERACTING] -> [TERMINATED]
                                                       |
                                              Discriminator 호출
                                              authenticity==1 -> 종료 (reward=1.0)
                                              authenticity==0 -> 재생성 (reward=0.0)
```

---

## 4. Authenticity Discriminator (고유 혁신)

**파일**: `DeepInnovator_interation.py:31-152`

기존 9개 Claw 런타임에 없는 **완전히 새로운 패턴** (패턴 R1):

```python
# 판별기 호출 (3회 재시도)
for i in range(self.num_retries):
    response = await call_agent(
        model=self.discriminator_model,
        messages=[{"role": "user", "content": discriminator_prompt}],
    )
    if authenticity == 1:    # 실제 아이디어
        reward = 1.0         # 종료
    else:                    # 가상 아이디어
        reward = 0.0         # 재생성
```

**판별 기준** (형식이 아닌 내용 품질):
- 기술적 깊이 (용어만 나열 vs 실제 통합)
- 문제 명확성 (추상적 vs 구체적)
- `technical_approach`의 실행 가능성 (데이터 소스, 파라미터, 계산 복잡도)

**Red Flags for Fictional Ideas**:
1. 표면적 기술 깊이 (용어만 언급, 실제 통합 없음)
2. 실제 도전 무시 (계산 복잡도, 데이터 가용성)
3. 비현실적 통합 (호환성 설명 없이 여러 기법 결합)
4. 추상적 한계점만 (구체적 실제 도전 없음)
5. 야심 과다 (너무 많은 불균형한 문제)

---

## 5. 계층적 메모리 시스템

```
Layer 0: 개별 논문 분석          (inner_paper_memory.json)
Layer 1: 그룹화 + 라우팅        (inter_paper_group.json)
Layer 2: 통찰 생성               (connections + serendipity + trending)
Layer 3: 아이디어 합성           (ideas.json)
```

**Paper_Profile 클래스** (utils.py:47-155):
- JSON 파일 기반 지속성
- 증분 처리 (새 논문만 분석)
- 양방향 ID 매핑으로 추적 가능

---

## 보상/훈련 시스템

### GRPO (Group Relative Policy Optimization)

**파일**: `verl/trainer/ppo/core_algos.py:267-330`

같은 프롬프트에 대한 k개 생성물을 그룹화하여 **상대적 이점**을 계산:

```python
# 각 그룹 g에 대해:
# a_i = (r_i - mean_g) / std_g  (정규화된 이점)
for idx in id2score:
    scores_tensor = torch.stack(id2score[idx])
    id2mean[idx] = torch.mean(scores_tensor)    # mean_g
    id2std[idx] = torch.std(scores_tensor)      # std_g
```

**변형**: GRPO (기본), GRPO_VECTORIZED (최적화), GRPO_PASSK (최고 샘플만), Dr.GRPO (정규화 제거)

### 다중 메트릭 보상 시스템

| 메트릭 | 가중치 | 역할 |
|--------|-------|------|
| **Delta Reward** | 5.0 | 인접 아이디어 간 개선도 측정 (LLM 판사) |
| **Token Amount** | 0.1 | 3000-5000자 범위 유지 장려 |
| **Basic Reward** | - | turn_scores 합산 (authenticity) |

**Delta Reward 핵심**: 절대 점수가 아닌 **상대적 개선도**를 보상. 에이전트가 반복할수록 더 나은 아이디어를 생성하도록 학습.

---

## 코드 참조 인덱스

| 파일 | 라인 | 기능 |
|------|------|------|
| `recipe/DeepInnovator/reward_function.py` | 111-477 | DeepInnovatorRewardManager |
| `recipe/DeepInnovator/metrics/delta_reward.py` | 107-255 | 아이디어 개선도 메트릭 |
| `recipe/DeepInnovator/DeepInnovator_agent_loop.py` | 36-132 | 에이전트 루프 |
| `recipe/DeepInnovator/DeepInnovator_interation.py` | 31-289 | Discriminator + 보상 |
| `verl/trainer/ppo/core_algos.py` | 267-419 | GRPO 알고리즘 (4 변형) |
| `verl/tools/base_tool.py` | 24-94 | BaseTool 인터페이스 |
| `data_prepare/utils.py` | 182-299 | call_agent() + load_config() |
