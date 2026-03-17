# 연구 자동화 도구 분석: DeepInnovator & Autoresearch

> **분석 대상**: repos_research/ 내 2개 연구 자동화 도구
> **교차 참조**: 기존 9개 Claw 런타임 분석 (5개 보고서)
> **작성 일자**: 2026-03-09

---

## 1. Executive Summary

기존 `repos/`의 9개 에이전트 런타임 프레임워크와 달리, `repos_research/`의 2개 도구는 **연구 자동화**라는 특수 목적에 집중한다. 이 보고서는 두 도구의 아키텍처를 분석하고, 기존 Claw 패턴과의 교차 매핑을 통해 idea3(AI Research Agent), idea4(Lab AI Agent) 설계에 활용할 패턴을 추출한다.

### 핵심 발견

| 발견 | 설명 |
|------|------|
| **연구 특화 에이전트 패턴** | DeepInnovator는 7개 전문 에이전트의 계층적 파이프라인, Autoresearch는 단일 에이전트 무한 루프. 완전히 다른 설계 철학 |
| **Program.md = 새로운 에이전트 지시 패턴** | Autoresearch의 program.md는 SKILL.md, HAND.toml과 동급의 에이전트 프로그래밍 형식이나, "무한 루프 + 자동 평가 + Git 상태 관리"라는 고유 특성 보유 |
| **Authenticity Discriminator** | DeepInnovator만의 혁신. 생성된 아이디어의 "실제성"을 LLM 판별기로 검증. 기존 9개 런타임에 없는 패턴 |
| **Prepare/Train 분리 = Capability Attenuation** | Autoresearch의 prepare.py(읽기전용) + train.py(수정가능) 분리는 IronClaw의 capability attenuation과 동일한 설계 원리 |
| **GRPO + Delta Reward** | DeepInnovator의 "이전 아이디어 대비 개선도" 보상은 연구 에이전트의 점진적 개선을 강화학습으로 학습시키는 최초 사례 |
| **5분 고정 예산** | Autoresearch의 시간 기반 종료 조건은 비용 제한(ZeroClaw $5/일)의 연구 버전 |

### 포지셔닝

```
                    복잡도 ^
                         |
  DeepInnovator *        |        * OpenClaw (범용 런타임)
  (RL 훈련 + 7 에이전트)   |
                         |
                         |        * IronClaw (보안 런타임)
                         |
                         |
  Autoresearch *         |        * Nanobot (경량 런타임)
  (단일 파일 수정 루프)    |
                         |
          ---------------------------------------->  범용성
          연구 특화              범용 에이전트
```

---

## 2. 비교 매트릭스

### 2.1 기본 정보

| 항목 | DeepInnovator | Autoresearch |
|------|--------------|--------------|
| **저자** | HKUDS (홍콩대) | Karpathy |
| **언어** | Python | Python |
| **규모** | ~105K LOC, 402 파일 | ~1K LOC, 3 파일 |
| **목적** | RL 기반 연구 아이디어 생성 모델 훈련 | 자율 ML 실험 반복 |
| **에이전트 수** | 7개 (YAML 설정) | 1개 (LLM 자체) |
| **도구** | MCP + Sandbox + 검색 | 없음 (코드 수정만) |
| **기억** | JSON 파일 계층적 | results.tsv + Git |
| **보상** | GRPO + Delta Reward + Token Amount | val_bpb (낮을수록 좋음) |
| **루프** | 데이터 준비 파이프라인 (4단계) | 무한 루프 (5분/실험) |

### 2.2 Claw 패턴 대비

| 패턴 | DeepInnovator | Autoresearch | 가장 유사한 Claw |
|------|--------------|--------------|----------------|
| **에이전트 지시** | YAML 프롬프트 | program.md | HAND.toml (OpenFang) |
| **기억 관리** | JSON 파일 계층 | results.tsv + Git | ZeroClaw Soul Snapshot |
| **도구 통합** | MCP + Native | 없음 | Nanobot (MCP) |
| **보안/격리** | VERL 샌드박스 | prepare.py 읽기전용 | IronClaw (Capability) |
| **비용 제한** | 없음 | 5분 고정 예산 | ZeroClaw ($5/일) |
| **검증** | Discriminator | val_bpb 비교 | 없음 (신규 패턴) |
| **병렬 실행** | Step 3 병렬 가능 | 순차 | PicoClaw (goroutine) |

---

## 3. DeepInnovator 에이전트 아키텍처

### 3.1 7개 에이전트 역할

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

### 3.2 YAML 설정 스키마

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

### 3.3 오케스트레이션 루프

#### 데이터 준비 파이프라인 (Step 1-4)

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

#### 런타임 Agent Loop (DeepInnovatorAgentLoop)

```
[PENDING] -> [GENERATING] -> [PROCESSING_TOOLS] -> [INTERACTING] -> [TERMINATED]
                                                       |
                                              Discriminator 호출
                                              authenticity==1 -> 종료 (reward=1.0)
                                              authenticity==0 -> 재생성 (reward=0.0)
```

**에이전트 호출**: `call_agent()` (utils.py:182-257)
- OpenAI 호환 클라이언트로 스트리밍 응답 처리
- JSON 파싱 + json_repair로 손상된 JSON 복구
- 에러 시 빈 dict 반환

### 3.4 Authenticity Discriminator (고유 혁신)

**파일**: `DeepInnovator_interation.py:31-152`

기존 9개 Claw 런타임에 없는 **완전히 새로운 패턴**:

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

### 3.5 계층적 메모리 시스템

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

## 4. DeepInnovator 보상/훈련 시스템

### 4.1 GRPO (Group Relative Policy Optimization)

**파일**: `verl/trainer/ppo/core_algos.py:267-330`

같은 프롬프트에 대한 k개 생성물을 그룹화하여 **상대적 이점**을 계산:

```python
# 각 그룹 g에 대해:
# a_i = (r_i - mean_g) / std_g  (정규화된 이점)

for idx in id2score:
    scores_tensor = torch.stack(id2score[idx])
    id2mean[idx] = torch.mean(scores_tensor)    # mean_g
    id2std[idx] = torch.std(scores_tensor)      # std_g

for i in range(bsz):
    scores[i] = (scores[i] - id2mean[index[i]]) / (id2std[index[i]] + epsilon)
```

**예시**:
```
프롬프트 P1에 대한 3개 샘플:
- 샘플1 점수: 0.8 -> advantage = +1.22 (학습 강화)
- 샘플2 점수: 0.6 -> advantage =  0.00 (중립)
- 샘플3 점수: 0.4 -> advantage = -1.22 (학습 약화)
```

**변형**: GRPO (기본), GRPO_VECTORIZED (최적화), GRPO_PASSK (최고 샘플만), Dr.GRPO (정규화 제거)

### 4.2 다중 메트릭 보상 시스템

**파일**: `reward_function.py:111-477`

| 메트릭 | 가중치 | 파일 | 역할 |
|--------|-------|------|------|
| **Delta Reward** | 5.0 | `metrics/delta_reward.py` | 인접 아이디어 간 개선도 측정 (LLM 판사) |
| **Token Amount** | 0.1 | `metrics/token_amount.py` | 3000-5000자 범위 유지 장려 |
| **Basic Reward** | - | `metrics/basic_reward.py` | turn_scores 합산 (authenticity) |

#### Delta Reward 상세

```python
# 대화에서 모든 아이디어 추출 (시간순)
all_ideas = extract_all_ideas(conversation_history)

# 인접 쌍별 개선도
for i in range(len(all_ideas) - 1):
    # LLM 판사가 두 아이디어를 ground_truth 대비 평가
    prompt = IMPROVE_PROMPT.format(idea1=new, idea2=old, ground_truth=gt)
    pair_score = response["idea1_improve_score"] - response["idea2_improve_score"]
    total_score += pair_score
```

**핵심**: 절대 점수가 아닌 **상대적 개선도**를 보상. 에이전트가 반복할수록 더 나은 아이디어를 생성하도록 학습.

#### Token Amount 보상 형태

```
보상값
  1.0 |     ___________
      |    /           \
  0.5 |   /             \___
      |  /                   \
  0.0 |_/____________________\______
      0  3000  5000  7000 10000  길이(문자)
```

### 4.3 도구 통합 (훈련 시)

| 도구 유형 | 파일 | 특징 |
|----------|------|------|
| **Native Tools** | `verl/tools/base_tool.py` | BaseTool 인터페이스: create->execute->calc_reward->release |
| **MCP Tools** | `verl/tools/mcp_search_tool.py` | MCPSearchTool: MCP 서버 응답 파싱 |
| **Sandbox** | `verl/tools/sandbox_fusion_tools.py` | Ray 기반 분산 실행, TokenBucket 속도 제한 |

**도구 등록**: `verl/tools/utils/tool_registry.py`
```python
def initialize_tools_from_config(tools_config_file):
    for tool_config in tools_config.tools:
        tool_type = ToolType(tool_config.config.type)  # NATIVE or MCP
        if tool_type == ToolType.NATIVE:
            tool = tool_cls(config=..., tool_schema=...)
        elif tool_type == ToolType.MCP:
            mcp_tools = run_coroutine(initialize_mcp_tool(...))
```

### 4.4 훈련 데이터 흐름

```
+---------------------------------------------------+
| 1. 생성 (Rollout): 프롬프트 -> k개 응답 샘플링      |
+--------------------------+------------------------+
                           |
+--------------------------v------------------------+
| 2. 인터랙션: 아이디어 추출 -> Discriminator 검증    |
|    authenticity==1 -> turn_score=1.0 (종료)        |
|    authenticity==0 -> turn_score=0.0 (재생성)      |
+--------------------------+------------------------+
                           |
+--------------------------v------------------------+
| 3. 보상 계산 (asyncio.gather 병렬):                |
|    delta_reward x 5 + token_amount x 0.1           |
|    + basic_reward (turn_scores 합산)               |
|    -> clamp(-1, 1) 정규화                          |
+--------------------------+------------------------+
                           |
+--------------------------v------------------------+
| 4. GRPO: 그룹별 mean,std -> 정규화 이점 -> PPO 손실|
+---------------------------------------------------+
```

---

## 5. Autoresearch 설계 철학

### 5.1 전체 구조

Autoresearch는 극단적 단순성의 자율 ML 실험 프레임워크:

```
+--------------------------------------------------+
|  program.md (115줄)                                |
|  = 에이전트 지시서 (= SKILL.md/HAND.toml 역할)     |
|  "NEVER STOP" + 5분 고정 예산 + keep/discard 루프  |
+--------------------------------------------------+
|  train.py (630줄) -- 에이전트가 수정하는 유일한 파일 |
|  - GPT 모델 아키텍처 (ResFormer + Value Embedding) |
|  - MuonAdamW 옵티마이저                            |
|  - 하이퍼파라미터 + 학습 루프                      |
+--------------------------------------------------+
|  prepare.py (389줄) -- 읽기전용                     |
|  - evaluate_bpb(): BPB 메트릭 (수정 금지)          |
|  - make_dataloader(): best-fit packing             |
|  - 토크나이저: rustbpe, 8K vocab                   |
+--------------------------------------------------+
```

### 5.2 Program.md 패턴 상세

program.md는 **자율 연구 루프의 완전한 프로그래밍**:

```markdown
LOOP FOREVER:
1. 현재 git 상태 확인
2. train.py 수정 (아키텍처, 하이퍼파라미터, 옵티마이저 등)
3. git commit
4. uv run train.py > run.log 2>&1
5. grep "^val_bpb:" run.log -- 결과 확인
6. crash 여부 판단
7. results.tsv에 기록
8. val_bpb 개선? -> 유지 (커밋 유지)
9. val_bpb 미개선? -> git reset (되돌림)
```

**핵심 규칙**:
- **5분 고정 시간 예산**: `TIME_BUDGET = 300` (warmup 10스텝 제외). 12회/시간 가능
- **수정 가능**: train.py만 (모델, 옵티마이저, 하이퍼파라미터, 학습 루프 전부)
- **수정 불가**: prepare.py, 패키지 설치, evaluate_bpb() 함수
- **NEVER STOP**: 사용자가 잠을 자는 동안 ~100회 실험 수행 기대

### 5.3 Keep/Discard 메커니즘

```
results.tsv 형식:
commit    val_bpb   memory_gb  status   description
a1b2c3d   0.997900  44.0       keep     baseline
b2c3d4e   0.993200  44.2       keep     increase LR to 0.04
c3d4e5f   1.005000  44.0       discard  switch to GeLU activation
d4e5f6g   0.000000  0.0        crash    double model width (OOM)
```

| 상태 | 조건 | 행동 |
|------|------|------|
| `keep` | val_bpb 개선 (감소) | 커밋 유지, 브랜치 진행 |
| `discard` | val_bpb 미개선 | git reset (코드 되돌림) |
| `crash` | OOM, 버그, 타임아웃 | 스택 트레이스 조사, 이동 |

**단순성 기준**: 같은 val_bpb면 간단한 코드 우선. 코드 삭제로 개선되면 최우선.

### 5.4 모델 아키텍처 (에이전트가 수정하는 대상)

train.py에 구현된 GPT 모델의 주요 혁신:

| 혁신 | 구현 | 설명 |
|------|------|------|
| **ResFormer** | `resid_lambdas`, `x0_lambdas` (train.py:133-134) | 레이어별 잔차 가중치 (동적 깊이 제어) |
| **Value Embedding** | `ve_gate` (train.py:73-86) | Alternating 레이어에 입력 의존 gate로 v 보강 |
| **relu-squared** | `F.relu(x).square()` (train.py:106) | GELU/ReLU 대신 실험적 활성화 |
| **Soft-capping** | `softcap * tanh(logits / softcap)` (train.py:281) | logits [-15, 15] 범위 제한 |
| **MuonAdamW** | 이중 옵티마이저 (train.py:296-426) | 2D 행렬: Muon (극좌표 정규화), 나머지: AdamW |

**에이전트 수정 가능 범위**:

```
[O] 수정 가능 (하이퍼파라미터):
  DEPTH=8, ASPECT_RATIO=64, HEAD_DIM=128
  EMBEDDING_LR=0.6, MATRIX_LR=0.04
  TOTAL_BATCH_SIZE=2^19, WEIGHT_DECAY=0.2

[O] 수정 가능 (코드 구조):
  활성화 함수, 어텐션 구조, 초기 가중치, LR 스케줄

[X] 수정 불가:
  prepare.py, evaluate_bpb(), 패키지 설치, TIME_BUDGET
```

### 5.5 BPB (Bits Per Byte) 메트릭

```python
# prepare.py:342-364
def evaluate_bpb(model, tokenizer, batch_size):
    # per-token cross-entropy를 합산
    # 타겟 바이트 길이를 합산
    # nats/byte -> bits/byte 변환
    # 특수 토큰(바이트 길이 0)은 제외
    return total_nats / (math.log(2) * total_bytes)
```

- **Vocab-size 독립**: 아키텍처 간 공정한 비교
- **베이스라인**: val_bpb = 0.997900, VRAM = 44.0 GB, MFU = 39.80%

---

## 6. Claw 패턴 매핑

기존 9개 런타임 분석에서 추출된 패턴이 연구 도구에 어떻게 적용/변형되는지 분석한다.

### 6.1 Memory 패턴

#### DeepInnovator JSON 계층 vs Claw Memory Tiers

| 측면 | DeepInnovator | Claw Tier 1 (IronClaw/OpenClaw/ZeroClaw) |
|------|--------------|----------------------------------------|
| **저장** | JSON 파일 (6개: memory, group, connections, serendipity, trending, ideas) | Vector DB (pgvector/sqlite-vec/LanceDB) + MEMORY.md |
| **검색** | 없음 (전체 로드) | 하이브리드 (BM25 + vector + temporal decay) |
| **계층** | 4-layer (분석->그룹->통찰->합성) | 2-layer (MEMORY.md 항상 로드 + DB 동적 주입) |
| **지속성** | 파일 시스템 | DB + Git (ZeroClaw Soul Snapshot) |

**핵심 차이**: DeepInnovator의 4-layer 계층은 **연구 워크플로우에 특화**된 구조. Claw의 2-layer "이중 주입"과는 다른 설계 원리이나, 논문 수가 수천 편으로 늘어나면 전체 로드가 불가능하므로 **Claw Tier 1 하이브리드 검색이 필수**.

#### Autoresearch results.tsv vs ZeroClaw Soul Snapshot

| 측면 | Autoresearch results.tsv | ZeroClaw Soul Snapshot |
|------|------------------------|----------------------|
| **형식** | TSV (탭 구분) | Markdown (Git 추적) |
| **내용** | commit, val_bpb, memory, status, description | 핵심 기억, 정체성, 사용자 선호 |
| **복원** | git reset으로 코드 상태 복원 | DB 손실 시 cold-boot hydration |
| **버전 관리** | Git 브랜치 (keep 커밋만 누적) | Git (MEMORY_SNAPSHOT.md) |

**발견**: 두 시스템 모두 **Git을 상태 관리 시스템으로 활용**. Autoresearch는 "코드 상태"를, ZeroClaw는 "기억 상태"를 Git으로 추적. **연구 에이전트는 두 가지를 결합**해야 한다: 코드 버전 + 연구 맥락 버전.

### 6.2 Tool 패턴

#### VERL Tools vs MCP 표준

| 측면 | DeepInnovator (VERL) | MCP 표준 (5/9 Claw 구현) |
|------|---------------------|--------------------------|
| **정의** | Python BaseTool 클래스 + YAML | JSON Schema + HTTP/stdio |
| **실행** | 인프로세스 (Ray 분산) | 별도 프로세스 (MCP 서버) |
| **격리** | Sandbox Fusion (TokenBucket 속도 제한) | 런타임별 (WASM/Docker/인프로세스) |
| **확장** | Native + MCP 혼합 가능 | MCP 표준화 |

**Autoresearch의 "NO TOOLS" 패턴**:
- 모든 작업을 LLM 코드 생성으로 처리 (도구 0개)
- 장점: 극단적 단순성, 어디서든 실행
- 단점: 에러 처리 불가, 재현성 약함

**idea3/idea4 시사점**: DeepInnovator의 MCP 통합 + Autoresearch의 단순성을 결합. 핵심 도구만 MCP로 (논문 검색, 메모리), 나머지는 LLM 직접 처리.

### 6.3 Context 패턴: program.md vs SKILL.md vs HAND.toml

| 측면 | program.md (Autoresearch) | SKILL.md (OpenFang Skills) | HAND.toml (OpenFang Hands) |
|------|--------------------------|---------------------------|---------------------------|
| **포맷** | Markdown (자연어) | Markdown (자연어) | TOML (구조화) |
| **목적** | 자율 루프 지시 | 스킬 기능 설명 | 자율 능력 선언 |
| **런타임 주입** | [X] (에이전트가 읽음) | [O] (시스템 프롬프트, 최대 2K) | [X] (선언적, 파싱) |
| **루프** | 무한 (NEVER STOP) | 1회 호출 | activate/pause/resume/deactivate |
| **상태 관리** | Git (코드 버전) | 외부 (DB/파일) | hand_state.json |
| **자동 평가** | [O] (val_bpb) | [X] | [X] |
| **에이전트 자율성** | 매우 높음 | 중간 | 높음 |
| **실패 처리** | 명시 (crash 분류, retry) | 암시적 | 예외 처리 |

**program.md의 고유성**:
1. **완전한 무한 루프 정의**: SKILL.md/HAND.toml은 단일 호출 또는 생명주기 관리. program.md는 무한 반복 연구 루프
2. **자동 성공 판정**: val_bpb 기반 keep/discard. 다른 형식은 LLM 또는 사용자 판정
3. **Git = 상태 머신**: 코드 버전이 곧 실험 상태. 다른 형식은 별도 상태 저장

**idea3/idea4를 위한 "Research Instruction Document" 제안**:
```markdown
# research_program.md (program.md + HAND.toml 결합)

## Objective
[연구 목표 + 평가 메트릭 정의]

## Tools (HAND.toml 영감)
[사용 가능한 MCP 도구 목록 + 접근 권한]

## Loop (program.md 영감)
[무한 루프 정의 + keep/discard 기준 + 시간 예산]

## Memory (ZeroClaw 영감)
[RESEARCH_SNAPSHOT.md 자동 생성 + Git 추적]
```

### 6.4 Security 패턴

#### prepare.py 읽기전용 = IronClaw Capability Attenuation

| 측면 | prepare.py 읽기전용 | IronClaw Capability |
|------|-------------------|-------------------|
| **격리 수준** | 파일 시스템 (수정 불가 파일) | Capability 시스템 (ApprovalRequirement) |
| **시간 범위** | 설계 시 고정 | 매 도구 호출마다 |
| **보호 대상** | 평가 메트릭 무결성 | 크레덴셜, 파일 시스템 |
| **메커니즘** | 사회적 계약 (program.md 지시) | 암호학적 강제 (WASM fuel/memory) |

**핵심 통찰**: Autoresearch의 분리는 **사회적 계약**(program.md가 "수정하지 마"라고 지시)이지, 기술적 강제가 아님. IronClaw는 **암호학적 강제**(WASM 샌드박스). 연구 에이전트에서는:
- 평가 메트릭: IronClaw 수준 강제 (수정 불가능)
- 연구 코드: Autoresearch 수준 자유 (자유롭게 수정)

#### 5분 고정 예산 = ZeroClaw 비용 한도

| 측면 | Autoresearch 5분 예산 | ZeroClaw $5/일 |
|------|---------------------|---------------|
| **단위** | 시간 (wall-clock) | 비용 (달러) |
| **목적** | 실험 공정 비교 + 자원 제어 | API 비용 폭발 방지 |
| **구현** | `if total_training_time >= TIME_BUDGET: break` | 일별 누적 비용 추적 |

**idea3/idea4 적용**: 연구 에이전트에도 이중 한도 필요:
1. **시간 한도**: 논문 검색 1회당 최대 N분
2. **비용 한도**: API 호출 일별 $X 제한

---

## 7. idea3/idea4 설계 시사점

### 7.1 두 도구에서 추출한 신규 패턴

#### 패턴 R1: Authenticity Discriminator (DeepInnovator)

연구 에이전트가 생성한 아이디어/요약의 품질을 자동 검증:
- 표면적 기술 깊이 탐지
- 비현실적 통합 탐지
- 실행 가능성 검증

**idea3 적용**: Paper Analysis Agent에 Discriminator 추가. 요약의 "할루시네이션" 탐지.

#### 패턴 R2: Delta Reward (DeepInnovator)

**이전 결과 대비 개선도**를 보상으로 사용:
- 절대 품질이 아닌 **상대적 진보**
- 에이전트가 반복할수록 더 나은 결과

**idea3 적용**: 논문 검색 에이전트가 "이전 검색보다 더 관련성 높은 논문"을 찾도록 강화.

#### 패턴 R3: Fixed-Budget Loop (Autoresearch)

시간 기반 자율 실험:
- 5분/실험 x 12회/시간 x 무한
- Keep/Discard 자동 의사결정
- Git = 상태 머신

**idea4 적용**: Lab Agent의 자율 문헌 조사에 적용. "30분 동안 X 주제 논문 검색" -> 결과 평가 -> keep/discard.

#### 패턴 R4: Hierarchical Agent Pipeline (DeepInnovator)

4-layer 계층 (분석->그룹->통찰->합성):
- 각 레이어가 이전 레이어 출력을 입력으로
- Step 3 병렬 가능 (Connections, Serendipity, Trending 독립)

**idea3 적용**: Paper Search -> Paper Analysis -> Knowledge Integration -> Insight Generation 파이프라인.

#### 패턴 R5: Deep Synthesis Principle (DeepInnovator)

"A+B+C 단순 조합 금지":
- 시너지 원리 이해 필수
- 구체적 기술적 접근 요구
- 실행 가능성 검증

**idea3/idea4 적용**: Literature Review Agent의 프롬프트에 동일 원칙 적용. "논문 A와 B를 그냥 나열하지 말고, 둘의 방법론이 어떻게 상호 보완되는지 설명하라."

### 7.2 기존 Claw 패턴 + 연구 도구 패턴 결합

| 기능 | Claw 패턴 | 연구 도구 패턴 | 결합 |
|------|----------|--------------|------|
| **기억** | ZeroClaw Soul Snapshot + OpenClaw 하이브리드 검색 | DeepInnovator 4-layer 계층 | RESEARCH_SNAPSHOT.md (Git) + 논문 Vector DB + 계층적 메모리 |
| **도구** | MCP 표준 (5/9 수렴) | VERL Native + MCP | 핵심 연구 도구를 MCP로: ArXiv, PubMed, Zotero, Memory |
| **지시** | SKILL.md + HAND.toml | program.md (무한 루프) | research_program.md (루프 + 도구 + 메모리 통합) |
| **보안** | IronClaw Capability | prepare.py 읽기전용 | 평가 메트릭 강제 보호 + 연구 코드 자유 수정 |
| **비용** | ZeroClaw $5/일 | 5분 고정 예산 | 이중 한도 (시간 + 비용) |
| **검증** | 없음 | Discriminator + val_bpb | 연구 품질 자동 검증 레이어 |
| **병렬** | PicoClaw goroutine | Step 3 병렬 | 논문 분석 병렬 + 통찰 생성 병렬 |

### 7.3 연구 에이전트 최적 스택 제안

```
+---------------------------------------------------+
|  [에이전트 지시] research_program.md               |
|  = program.md(루프) + HAND.toml(도구/권한)         |
+---------------------------------------------------+
|  [기억 백엔드] Memory MCP Server                   |
|  ZeroClaw Snapshot + OpenClaw 하이브리드 검색       |
|  + DeepInnovator 4-layer 계층                      |
+---------------------------------------------------+
|  [런타임] NanoClaw (보안+단순) 또는 OpenClaw (생태계)|
|  + MCP 도구: ArXiv, PubMed, Zotero, Notion        |
+---------------------------------------------------+
|  [검증 레이어] Authenticity Discriminator           |
|  + val_bpb류 자동 메트릭                           |
+---------------------------------------------------+
|  [안전장치] 이중 한도 (시간 + 비용)                 |
|  + prepare.py식 평가 메트릭 보호                    |
+---------------------------------------------------+
```

---

## 8. 결론 및 신규 오픈 퀘스천

### 8.1 핵심 결론

1. **DeepInnovator와 Autoresearch는 상호 보완적**: DeepInnovator는 "어떤 아이디어를 추구할 것인가"(방향), Autoresearch는 "아이디어를 어떻게 검증할 것인가"(실행). 연구 에이전트는 양쪽 모두 필요.

2. **기존 9개 Claw 패턴은 직접 재사용 가능**: Memory(ZeroClaw Snapshot + OpenClaw 하이브리드), Tool(MCP 표준), Security(IronClaw Capability), Cost(ZeroClaw 한도) 패턴은 연구 에이전트에 그대로 적용 가능.

3. **2개 연구 도구가 보여주는 새로운 패턴 5가지**:
   - R1: Authenticity Discriminator (품질 자동 검증)
   - R2: Delta Reward (상대적 개선도 보상)
   - R3: Fixed-Budget Loop (시간 기반 자율 실험)
   - R4: Hierarchical Agent Pipeline (4-layer 계층)
   - R5: Deep Synthesis Principle (깊이 있는 합성 원칙)

   **외부 참조 패턴 (Google ADK, always-on-memory-agent, 2026)**:
   - R9: Sleep Consolidation Loop — 인간 수면 중 기억 통합을 명시적으로 모델링한 주기적 에이전트. 30분 타이머로 unconsolidated 기억을 병합하고 cross-cutting insight 추출. "24시간 상주"는 Claude Code ralph/Autoresearch NEVER STOP과 동일 개념이나, **수면 통합 비유 + 메모리 마이크로서비스 분리(HTTP API)**는 신규 패턴. 단, 벡터 없이 LLM이 전체 메모리를 직접 읽는 구조라 Claw Tier 1 대비 스케일링 한계 존재.

4. **"Research Instruction Document" 제안**: program.md(루프) + HAND.toml(도구/권한) + ZeroClaw(기억 스냅샷)을 결합한 새로운 에이전트 지시 형식.

### 8.2 신규 오픈 퀘스천

**Q21. Authenticity Discriminator의 한계는?**
- 형식이 아닌 내용 품질을 평가한다고 하지만, LLM 판사(judge) 자체의 편향은?
- Ground truth 없이도 "실제성"을 판단할 수 있는가?
- 도메인 전문가 평가와의 상관관계는?

**Q22. Delta Reward의 수렴 문제**
- 아이디어가 충분히 좋아진 후에도 "더 나은 아이디어"를 요구하면 과적합(overfit)되는가?
- 탐색(exploration) vs 활용(exploitation) 균형은?
- 연구 에이전트에서 "충분히 좋은" 기준은 무엇인가?

**Q23. program.md vs HAND.toml의 최적 결합**
- 자연어(program.md)와 구조화(HAND.toml) 중 에이전트가 더 잘 따르는 형식은?
- 비개발 연구팀을 위한 최적 형식은?
- 둘을 하이브리드로 결합할 때 정보 중복/충돌은?

**Q24. 연구 자동화의 "실험 설계" 문제**
- Karpathy 발견: "에이전트는 잘 정의된 아이디어 구현은 잘하지만, 창의적 실험을 설계하지 못한다" (Q3 재확인)
- DeepInnovator의 Serendipity Engine이 이 문제를 부분적으로 해결하는가?
- 아니면 "실험 설계 skill"을 별도로 만들어야 하는가?

**Q25. 4-layer 계층적 메모리의 스케일링**
- DeepInnovator의 JSON 기반 메모리는 논문 수십 편 수준
- 수천 편으로 스케일링하면 어떤 문제가 발생하는가?
- Claw Tier 1 하이브리드 검색과의 통합 지점은?

**Q26. 연구 에이전트의 재현성 보장**
- Autoresearch: 고정 seed + 고정 데이터 + 고정 시간 -> 재현 가능
- 연구 에이전트: 외부 API(arxiv, pubmed) 의존 -> 재현 불가능?
- 재현 가능한 연구 파이프라인 설계는?

---

## 부록: 코드 참조 인덱스

### DeepInnovator 핵심 파일

| 파일 | 라인 | 기능 |
|------|------|------|
| `recipe/DeepInnovator/reward_function.py` | 111-477 | DeepInnovatorRewardManager |
| `recipe/DeepInnovator/metrics/delta_reward.py` | 107-255 | 아이디어 개선도 메트릭 |
| `recipe/DeepInnovator/metrics/token_amount.py` | 86-139 | 길이 최적화 메트릭 |
| `recipe/DeepInnovator/metrics/basic_reward.py` | 15-66 | turn_scores 합산 |
| `recipe/DeepInnovator/DeepInnovator_agent_loop.py` | 36-132 | 에이전트 루프 |
| `recipe/DeepInnovator/DeepInnovator_interation.py` | 31-289 | Discriminator + 보상 |
| `verl/trainer/ppo/core_algos.py` | 267-419 | GRPO 알고리즘 (4 변형) |
| `verl/tools/base_tool.py` | 24-94 | BaseTool 인터페이스 |
| `verl/tools/mcp_search_tool.py` | 28-70 | MCP 검색 도구 |
| `verl/tools/sandbox_fusion_tools.py` | 43-120 | Sandbox + TokenBucket |
| `verl/tools/utils/tool_registry.py` | 82-143 | 동적 도구 로딩 |
| `data_prepare/utils.py` | 182-299 | call_agent() + load_config() |
| `data_prepare/step1.py` | 26-102 | Paper -> JSON Lines |
| `data_prepare/step2.py` | 29-150 | Routing & Analysis |
| `data_prepare/step3.py` | 26-150 | Insight Generation |

### DeepInnovator 에이전트 YAML

| 에이전트 | 파일 |
|---------|------|
| Paper Analyzer | `data_preparation/config/agents/paper_analyzer.yaml` |
| Paper Router | `data_preparation/config/agents/paper_router.yaml` |
| Paper Idea Spark | `data_preparation/config/agents/paper_idea_spark.yaml` |
| Serendipity Engine | `data_preparation/config/agents/idea_serendipity_engine.yaml` |
| Paper Group Creator | `data_preparation/config/agents/paper_group_creator.yaml` |
| Paper Group Updater | `data_preparation/config/agents/paper_group_updater.yaml` |
| Paper Connections | `data_preparation/config/agents/idea_paper_connections.yaml` |
| Research Trending | `data_preparation/config/agents/idea_research_trending.yaml` |

### Autoresearch 전체 파일

| 파일 | 라인 수 | 역할 |
|------|--------|------|
| `program.md` | 115 | 에이전트 지시서 (= SKILL.md/HAND.toml) |
| `train.py` | 630 | 수정 가능: GPT + MuonAdamW + 하이퍼파라미터 |
| `prepare.py` | 389 | 읽기전용: evaluate_bpb + 데이터로더 + 토크나이저 |
| `pyproject.toml` | 28 | 고정 의존성 (패키지 추가 금지) |
