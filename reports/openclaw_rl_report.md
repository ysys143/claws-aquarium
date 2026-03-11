# OpenClaw-RL 종합 분석: 대화에서 학습으로

> **분석 대상**: `repos_research/openclaw-rl/` -- Gen-Verse/OpenClaw-RL (1,526 stars)
> **교차 참조**: 기존 9개 Claw 런타임 + DeepInnovator/Autoresearch (5+1개 보고서)
> **작성 일자**: 2026-03-11

---

## 1. Executive Summary

OpenClaw-RL은 기존 9개 Claw 런타임 및 2개 연구 도구와 **근본적으로 다른 설계 목표**를 가진다:

| 분류 | 설계 목표 | 핵심 메커니즘 |
|------|----------|------------|
| 9개 Claw 런타임 | "에이전트를 어떻게 구성할 것인가" | 도구 통합, 메모리 아키텍처, 보안 |
| DeepInnovator | "좋은 연구 아이디어를 어떻게 찾을 것인가" | RL 기반 아이디어 생성, Discriminator |
| Autoresearch | "ML 실험을 어떻게 자동화할 것인가" | 고정 예산 루프, Git 상태 관리 |
| **OpenClaw-RL** | **"대화를 통해 에이전트가 어떻게 스스로 개선할 것인가"** | **Conversation-to-Gradient, 비동기 4-컴포넌트** |

### 핵심 발견 (3대 신규 패턴)

| 패턴 | 설명 | 기존 패턴과의 차별점 |
|------|------|-------------------|
| **R6: Conversation-to-Gradient** | 다음 사용자 메시지를 자동으로 RL 학습 신호로 변환 | 명시적 레이블 불필요, 매 turn 신호 |
| **R7: Async 4-Component Architecture** | 서빙/수집/PRM/훈련이 독립 비동기 루프 | 기존 9개 Claw 전원 동기식 또는 순차 |
| **R8: OPD with Hindsight Hints** | 피드백에서 힌트 추출 -> 토큰 레벨 teacher 신호 | 스칼라 보상 대비 훨씬 풍부한 방향성 |

### 구성

```
repos_research/openclaw-rl/
├── openclaw-rl/       # Binary RL (GRPO) -- Track 1
├── openclaw-opd/      # On-Policy Distillation -- Track 1
├── openclaw-combine/  # Binary RL + OPD 결합 -- Track 1
├── openclaw-test/     # 평가 프레임워크
├── terminal-rl/       # 터미널 에이전트 RL -- Track 2
├── gui-rl/            # GUI 에이전트 RL -- Track 2
├── swe-rl/            # SWEBench RL -- Track 2
├── toolcall-rl/       # 도구 호출 RL -- Track 2
├── openclaw/          # OpenClaw 런타임 (Skills + ACP + Swabble)
└── slime/             # Base RL 프레임워크 (THUDM)
```

---

## 2. 기본 정보 및 비교 매트릭스

### 2.1 기본 정보

| 항목 | 내용 |
|------|------|
| **저자/조직** | Gen-Verse (THUDM 계열) |
| **언어** | TypeScript 31MB, Python 12.5MB, Shell, Swift, Kotlin 등 |
| **규모** | 1,526 stars, Fork 147 (2026-03-11 기준) |
| **목적** | 대화 피드백으로 개인화 AI 에이전트를 온라인 RL 훈련 |
| **기반 프레임워크** | Slime (THUDM), SGLang, Ray |
| **하드웨어 요구** | 8x GPU (CUDA 12.9, Python 3.12) |
| **출시** | 2026-02-26 (v1), 2026-03-11 (Tech Report + Track 2) |

### 2.2 기존 연구 도구 대비

| 항목 | OpenClaw-RL | DeepInnovator | Autoresearch |
|------|------------|--------------|--------------|
| **학습 목표** | 대화형 에이전트 개인화 | 연구 아이디어 생성 | ML 모델 성능 개선 |
| **RL 알고리즘** | GRPO (Binary) + OPD | GRPO + Delta Reward | 없음 (Git 상태 기반) |
| **보상 신호** | 다음 사용자 메시지 (PRM) | Authenticity Discriminator | val_bpb 메트릭 |
| **에이전트 수** | 4 컴포넌트 (비동기) | 7개 (계층 파이프라인) | 1개 (단일 루프) |
| **도구** | SGLang API | MCP + Sandbox | 없음 |
| **기억** | 세션 상태 (in-memory) | JSON 4-layer | results.tsv + Git |
| **루프 종료** | 가중치 업데이트 기반 | 파이프라인 완료 | 5분 고정 예산 |

### 2.3 Claw 런타임 대비 패턴 매핑

| 패턴 | OpenClaw-RL | 가장 유사한 Claw | 차이 |
|------|------------|----------------|------|
| **GRPO 구현** | 스칼라 브로드캐스트 (정규화 없음) | DeepInnovator (그룹 정규화) | 두 구현 모두 "GRPO"지만 다름 |
| **비동기 병렬** | 4-컴포넌트 독립 루프 | PicoClaw (goroutine) | PicoClaw는 도구 실행만 병렬; OpenClaw-RL은 전체 훈련 파이프라인 |
| **검증 메커니즘** | PRM majority vote | DeepInnovator Discriminator | PRM은 대화 흐름 평가; Discriminator는 아이디어 진위 평가 |
| **에이전트 지시** | OpenAI-compatible API (헤더 기반) | HAND.toml, SKILL.md, program.md | 구조화된 지시가 아니라 live traffic |
| **비용 제한** | 없음 | ZeroClaw ($5/day), Autoresearch (5분) | 비용 모델 없음 (자체 인프라 전제) |

---

## 3. Binary RL (GRPO) 아키텍처

### 3.1 GRPO 구현 세부사항

**파일**: `openclaw-rl/openclaw_api_server.py`, `slime/slime/utils/ppo_utils.py:201-208`

```python
def get_grpo_returns(rewards: torch.Tensor, kl: list[torch.Tensor]):
    returns = []
    for i in range(len(rewards)):
        returns.append(torch.ones_like(kl[i]) * rewards[i])
    return returns
```

- **Advantage**: 스칼라 보상 `r`을 모든 응답 토큰에 균일 브로드캐스트 (`A_t = r`)
- **정규화**: 없음 (`--disable-rewards-normalization`)
- **클리핑**: 비대칭 PPO -> `[1-e, 1+e_high] = [0.8, 1.28]` (Dr.GRPO 변형)
- **KL penalty**: `low_var_kl` 방식, 계수 0.0 (실질적으로 비활성)

**DeepInnovator GRPO와의 차이**:
| 측면 | OpenClaw-RL GRPO | DeepInnovator GRPO |
|------|-----------------|-------------------|
| Advantage 계산 | `r` 직접 브로드캐스트 | `(score - group_mean) / group_std` |
| 보상 정규화 | 없음 | 그룹 내 상대적 정규화 |
| 클리핑 | 비대칭 (0.8, 1.28) | 표준 대칭 |
| 신호 출처 | 다음 대화 상태 | Discriminator LLM judge |

### 3.2 PRM (Process Reward Model) Judge

**점수 체계**: `{+1, -1, 0}` (good / bad / neutral)

```
판정 기준:
  +1 (good):    환경 피드백이 성공/진행 -> "task progressed"
  -1 (bad):     수정 요청, 재시도 요청, 도구 오류
   0 (neutral): 모호한 피드백, 무관한 후속 메시지
```

**다중 투표** (majority voting, 기본 m=3):
```python
results = await asyncio.gather(
    *[self._query_prm_once(judge_prompt, i) for i in range(self._prm_m)]
)
final = _majority_vote([r[0] for r in results])
```

### 3.3 Conversation-to-Gradient 파이프라인 (R6 패턴)

```
[Step 1] API 요청 수신 (POST /v1/chat/completions)
  Header: X-Session-Id, X-Turn-Type ("main" or "side"), X-Session-Done

[Step 2] turn_type == "main" -> SGLang 생성 + per-token logprobs 수집
         turn_type == "side" -> 생성만, 학습 데이터 없음

[Step 3] 현재 응답 + logprobs를 _pending_records에 버퍼링

[Step 4] 다음 API 요청 도착 -> next_state = 다음 사용자/환경 메시지

[Step 5] _flush_pending_record() -> PRM 비동기 평가 시작
  "좋아!"            -> +1
  "틀렸어, 다시 해"  -> -1
  "날씨가 어때?"     -> 0

[Step 6] 샘플 조립 + at-least-one guarantee 체크
  if session_effective[session_id] == 0 and score == 0:
      exclude = False  # 세션당 최소 1개 강제 포함

[Step 7] output_queue.put(sample) -> Trainer 소비
```

**At-least-one guarantee** (`openclaw_api_server.py:615-622`):
- 각 세션에서 최소 1개의 유효 샘플(loss_mask=1) 보장
- score=0(neutral)이라도 해당 세션의 첫 샘플이면 포함
- 마지막 턴 (`has_next_state=False`)은 기본 제외

---

## 4. 비동기 4-컴포넌트 아키텍처 (R7 패턴)

### 4.1 컴포넌트 구성

| 컴포넌트 | 파일 | GPU | 역할 |
|---------|------|-----|------|
| **Agent Serving** | SGLang router | ACTOR_GPUS=4 | 정책 모델 배포, logprob 수집 |
| **Rollout Collection** | `openclaw_api_server.py` (FastAPI) | ROLLOUT_GPUS=2 | API 요청 처리, 샘플 버퍼링 |
| **PRM Judging** | 별도 PRM SGLang 서버 | PRM_GPUS=2 | 비동기 reward 평가 (m=3 병렬) |
| **Policy Training** | `train_async.py` (Ray actor) | ACTOR_GPUS | GRPO/OPD loss, 가중치 업데이트 |

### 4.2 컴포넌트 간 통신

```python
# API Server -> Trainer
await asyncio.to_thread(self.output_queue.put, (group_index, [sample]))
# 채널: queue.Queue(maxsize=100000)

# Graceful Weight Update
def pause_submission(self):
    self._submission_enabled.clear()  # 새 샘플 제출 중단

def resume_submission(self):
    self._submission_enabled.set()    # 업데이트 완료 후 재개
```

### 4.3 기존 Claw와 비교

| 측면 | OpenClaw-RL (R7) | PicoClaw (goroutine) | DeepInnovator (Step 3 병렬) |
|------|-----------------|--------------------|-----------------------------|
| 병렬화 범위 | 전체 훈련 파이프라인 | 도구 실행만 | 인사이트 생성 단계만 |
| 결합도 | Queue (느슨) | WaitGroup (동기 수집) | 순차 파이프라인 |
| 실시간성 | 사용자 응답 지연 없음 | 응답 후 처리 | 배치 처리 |
| 가중치 업데이트 | Graceful pause/resume | 없음 | 없음 |

---

## 5. On-Policy Distillation with Hindsight Hints (R8 패턴)

### 5.1 힌트 추출 메커니즘

**파일**: `openclaw-opd/openclaw_opd_api_server.py:71-119`

```python
# Step 1: Judge LLM이 next_state에서 힌트 생성 (m개 독립 투표)
_build_hint_judge_messages(response_text, next_state_text, next_state_role)

# Step 2: 결과 파싱
def _parse_judge_result(text: str) -> tuple[int | None, str]:
    # \boxed{1} = 힌트 있음 ([HINT_START]...[HINT_END] 추출)
    # \boxed{-1} = 힌트 없음

# Step 3: 최고 힌트 선택 (m개 중 가장 긴 non-trivial, 최소 10자)
def _select_best_hint(votes: list[dict]) -> dict | None:
    good = [v for v in votes if v["score"] == 1 and len(v["hint"]) > 10]
    return max(good, key=lambda v: len(v["hint"]))
```

### 5.2 토큰 레벨 Advantage 계산

```
A_t = log pi_teacher(a_t | s + hint) - log pi_student(a_t | s)

구현:
1. prompt_with_hint = original_prompt + "\nHint: {hint}"
2. teacher_logprobs = teacher_model(prompt_with_hint)[response_tokens]
3. student_logprobs = 롤아웃 시점의 old_logprobs
4. teacher_advantages = teacher_logprobs - old_logprobs  (per-token)
```

**핵심 최적화**: response-suffix만 log-prob 계산 -> peak memory 절감

### 5.3 Combined Method

```python
# openclaw-combine/combine_loss.py:27-141
w_opd = float(os.getenv("OPENCLAW_COMBINE_W_OPD", "1.0"))
w_rl  = float(os.getenv("OPENCLAW_COMBINE_W_RL",  "1.0"))
combined_advantages = w_opd * teacher_advantages + w_rl * grpo_advantages
```

**자동 분리 원리**:
- OPD 샘플 -> GRPO advantage = 0 (reward=0), teacher advantage만 유효
- RL 샘플 -> teacher advantage = 0 (hint 없음), GRPO advantage만 유효

| 차원 | Binary RL | OPD | Combined |
|------|-----------|-----|----------|
| 신호 유형 | Evaluative (good/bad) | Directional (per-token) | 혼합 |
| Advantage | Sequence-level scalar | Token-level directional | 양쪽 |
| Coverage | 모든 scored turn | 힌트 있는 turn만 | 모든 turn |
| 피드백 유형 | 암묵적 (좋아요/싫어요) | 명시적 텍스트 수정 | 양쪽 |

---

## 6. Track 2: 범용 에이전트 RL 인프라

### 6.1 시나리오별 비교

| 측면 | Terminal-RL | GUI-RL | SWE-RL | Toolcall-RL |
|------|------------|--------|--------|-------------|
| **환경** | bash shell | Desktop (스크린샷) | Docker (SWEBench) | Python sandbox |
| **Observation** | stdout/stderr | screenshot_b64 | diff + 파일 | 실행 결과 |
| **Action** | 명령어 | Mouse/Keyboard | 파일 수정 | 코드 생성 |
| **보상** | PRM 판정 | PRM 판정 | Test 통과 (binary) | 정확도 |
| **병렬화** | 적음 | 환경 pool (프리웜) | 환경 pool (Docker) | sandbox 격리 |
| **데이터** | 범용 대화 | GUI 태스크 | SWEBench v1/Gym | DAPO-Math-17K |
| **모델** | Qwen3-8B | Qwen3-VL-8B | -- | Qwen3-4B |

### 6.2 공통 인프라

```
모든 Track 2 시나리오:
  정책 모델 (SGLang) -> Rollout Worker -> Sample Queue -> SLIME Trainer
                ↑
         환경 Pool Server (Flask, lease-based)
         - 병렬 초기화 (ThreadPoolExecutor)
         - idle_ttl_seconds 자동 정리
         - max_envs 메모리 관리
```

### 6.3 환경 풀링 패턴

```python
class EnvPool:
    def __init__(self, max_envs, idle_ttl_seconds, env_kwargs,
                 prewarm_envs=0, prewarm_concurrency=1):
        # prewarm_concurrency: 동시 초기화 수
        # idle_ttl_seconds: 미사용 환경 자동 회수
        # lease_id: 에피소드별 원자성 보장
```

---

## 7. OpenClaw 런타임 및 Slime 프레임워크

### 7.1 Skills 시스템 vs OpenFang HAND.toml

| 측면 | OpenClaw Skills | OpenFang HAND.toml |
|------|----------------|-------------------|
| **선언 방식** | 디렉토리 + 스크립트 | TOML 파일 |
| **실행 방식** | Python 스크립트 직접 실행 | Hands lifecycle (activate/pause/resume) |
| **능력 제한** | 없음 (스크립트 권한 상속) | 18-type Capability 선언 |
| **동적 로딩** | [O] | [O] |
| **예시** | model-usage, skill-creator, openai-image-gen | 7개 번들 Hands |

### 7.2 ACP (Agent Client Protocol) + Swabble

**ACP** (`openclaw/docs.acp.md`):
- IDE <-> OpenClaw Gateway 간 stdio 기반 통신
- 세션 지원 (재연결, 리셋), Stream 기반 응답
- OpenClaw-RL 통합: IDE -> ACP -> Gateway -> OpenClaw-RL -> SLIME

**Swabble** (`openclaw/Swabble/docs/spec.md`):
- macOS 전용 음성 입력 daemon ("clawd" 웨이크워드)
- Speech.framework 감지 -> Hook executor -> OpenClaw 텍스트 전송
- OpenClaw-RL과 결합: 음성 대화 -> 자동 RL 훈련 (완전 자율)

**MCP와의 차이**:
| 측면 | ACP | MCP (de facto 표준) |
|------|-----|-------------------|
| **방향** | IDE <-> Agent 양방향 | Host -> Tool 단방향 |
| **프로토콜** | stdio (JSON Lines) | stdio/SSE/HTTP |
| **세션** | 명시적 세션 (재연결) | 요청별 |
| **채택** | OpenClaw 전용 | 5/9 Claw 수렴 |

### 7.3 Slime 프레임워크 구조

```
slime/
├── trainer/
│   ├── ppo/
│   │   ├── core_algos.py    # GRPO 변형 4가지 (GRPO, VECTORIZED, PASSK, Dr.GRPO)
│   │   └── ppo_trainer.py   # 훈련 루프
│   └── dpo/                 # Distillation trainer
├── rollout/
│   └── sglang_rollout.py    # SGLang 기반 롤아웃
└── utils/
    └── ppo_utils.py         # get_grpo_returns, GAE

GPU 분산: FSDP (Fully Sharded Data Parallel)
분산 동기화: all-reduce (actor GPU 간)
Megatron-LM: 체크포인트 호환, 토크나이저 통합
```

---

## 8. 신규 패턴 요약 (R6-R8)

### R6: Conversation-to-Gradient

**정의**: 다중 턴 대화에서 **다음 메시지**를 자동으로 RL 학습 신호로 변환. PRM이 "(현재 응답, 다음 상태)" 쌍을 평가하여 implicit reward 생성.

**핵심**:
- 명시적 레이블 불필요 (사용자의 자연스러운 후속 메시지가 신호)
- 매 turn에서 신호 생성 가능 (DeepInnovator 대비 3-5배 높은 밀도)
- PRM majority voting으로 노이즈 감소

| 측면 | R6 (OpenClaw-RL) | R1 (Discriminator) | Autoresearch val_bpb |
|------|-----------------|-------------------|---------------------|
| 신호 입력 | 대화 다음 상태 | 생성된 아이디어 내용 | 고정 성능 메트릭 |
| 자동화 | [O] 완전 자동 | [O] 완전 자동 | [O] 완전 자동 |
| 신호 밀도 | 높음 (매 turn) | 중간 (매 아이디어) | 낮음 (매 실험) |

### R7: Asynchronous 4-Component Architecture

**정의**: 에이전트의 4개 핵심 컴포넌트가 완전히 독립적인 비동기 루프로 실행. Queue + Event flag로 느슨한 결합.

**핵심**:
- 사용자 응답 지연 없음 (Agent Serving이 Training을 기다리지 않음)
- GPU 활용도 극대화 (모든 컴포넌트 항상 바쁜 상태)
- Graceful weight update (데이터 손상 없이 가중치 교체)

### R8: On-Policy Distillation with Hindsight Hints

**정의**: 다음 상태 피드백에서 LLM이 "힌트"를 추출 -> 힌트를 포함한 teacher 신호로 토큰 레벨 학습.

**핵심**:
- "맞다/틀렸다" 신호를 "왜, 어떻게" 신호로 변환
- 토큰별 방향성 advantage (Binary RL 스칼라 대비 고밀도)
- SDFT/SDPO Top-K distillation 확장 지원

---

## 9. idea3/idea4 설계 시사점

### 9.1 최적 Stack 제안

```
[에이전트 지시] research_program.md
  - 무한 루프 정의 (program.md 패턴, Autoresearch 영감)
  - 도구 선언 (MCP 표준)
  - 메모리 구조 (ZeroClaw 영감)

[런타임] OpenClaw (research edition)
  + Conversation-to-Gradient (R6)
  + 4-Component Async (R7)

[학습] Binary RL + OPD Combined (R8)
  - Binary: 암묵적 피드백 (좋아요/싫어요)
  - OPD: 명시적 텍스트 수정 피드백

[기억] 4-Layer Hierarchical (R4, DeepInnovator)
  - Layer 0: 논문 분석
  - Layer 1: 그룹화
  - Layer 2: 통찰 (connections, serendipity, trending)
  - Layer 3: 합성 (아이디어)

[안전장치] 이중 한도
  - 시간 한도: 30분/실험 (R3, Fixed-Budget)
  - 비용 한도: $5/day (ZeroClaw 패턴)
```

### 9.2 패턴 적용 우선순위

| 우선순위 | 패턴 | idea3 | idea4 |
|---------|------|-------|-------|
| 높음 | **R6 Conversation-to-Gradient** | 논문 추천 자동 개선 | 실험 결과 자동 학습 |
| 높음 | **R4 Hierarchical Pipeline** | 4-layer 메모리 | 실험 히스토리 분류 |
| 중간 | **R8 OPD** | 명시적 피드백 활용 | 실험 방향 힌트 |
| 중간 | **R3 Fixed-Budget Loop** | 30분/세션 | 30분/실험 |
| 낮음 | **R7 Async 4-Component** | 고성능 배포 시 | 고성능 배포 시 |

---

## 10. 미해결 질문 (Q27-Q30)

**Q27**: PRM 정확도 -- LLM judge 자체의 편향 가능성. majority voting m=3으로 충분한가? domain-specific PRM이 필요한가?

**Q28**: Async staleness -- 가중치 업데이트 중 API가 이전 모델 사용 -> off-policy 샘플 발생. 허용 가능한 staleness 한계는?

**Q29**: Hint 품질 -- trivial hint 필터(>10자)가 충분한가? 모순적 힌트 처리 방법은?

**Q30**: 재현성 -- 외부 API 의존 (SGLang, PRM 모델) 환경에서 실험 재현 가능한가?

---

## 11. 포지셔닝

```
                    복잡도 ^
                         |
  OpenClaw-RL *          |        * OpenClaw (범용 런타임)
  (RL 훈련 인프라,         |
   4-컴포넌트 비동기)      |
                         |        * IronClaw (보안 런타임)
                         |
  DeepInnovator *        |
  (7에이전트 파이프라인)   |
                         |
  Autoresearch *         |        * Nanobot (경량 런타임)
  (단일 파일 수정 루프)    |
                         +----------------------------------------> 범용성
                        학습/훈련 특화           범용 에이전트 런타임
```

**핵심 결론**: OpenClaw-RL은 기존 9개 Claw의 **다음 레이어**다. 기존 Claw가 "에이전트를 어떻게 구성하는가"를 정의했다면, OpenClaw-RL은 "그 에이전트가 어떻게 사용 중에 스스로 개선하는가"를 정의한다.

---

## 부록: 핵심 파일 인덱스

| 파일 | 라인 | 기능 |
|------|------|------|
| `openclaw-rl/openclaw_api_server.py` | 730L | Conversation-to-Gradient 전체 파이프라인 |
| `openclaw-rl/openclaw_rollout.py` | 153L | AsyncRolloutWorker, Queue 드레인 |
| `openclaw-rl/run_qwen3_4b_openclaw_rl.sh` | -- | Ray 오케스트레이션, GPU 할당 |
| `openclaw-opd/openclaw_opd_api_server.py` | 857L | OPD 힌트 추출, Teacher log-prob |
| `openclaw-combine/combine_loss.py` | 141L | Binary RL + OPD 결합 loss |
| `slime/slime/utils/ppo_utils.py` | 201-208 | `get_grpo_returns` (스칼라 브로드캐스트) |
| `openclaw/docs.acp.md` | -- | ACP 스펙 |
| `openclaw/Swabble/docs/spec.md` | -- | 음성 입력 daemon 프로토콜 |
