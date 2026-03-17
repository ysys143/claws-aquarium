# Autoresearch 상세 분석 보고서

> **소스**: `reports/repos_research/research_tools_report.md` §5에서 추출
> **분석 대상**: Karpathy/Autoresearch — 자율 ML 실험 루프
> **작성 일자**: 2026-03-09

---

## 목차

1. [기본 정보](#기본-정보)
2. [전체 구조](#1-전체-구조)
3. [Program.md 패턴](#2-programmd-패턴-상세)
4. [Keep/Discard 메커니즘](#3-keepdiscard-메커니즘)
5. [모델 아키텍처](#4-모델-아키텍처-에이전트가-수정하는-대상)
6. [BPB 메트릭](#5-bpb-bits-per-byte-메트릭)
7. [Claw 패턴 매핑](#claw-패턴-매핑)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **저자** | Karpathy |
| **언어** | Python |
| **규모** | ~1K LOC, 3 파일 |
| **목적** | 자율 ML 실험 반복 |
| **에이전트 수** | 1개 (LLM 자체) |
| **도구** | 없음 (코드 수정만) |
| **기억** | results.tsv + Git |
| **루프** | 무한 루프 (5분/실험) |

---

## 1. 전체 구조

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

---

## 2. Program.md 패턴 상세

program.md는 **자율 연구 루프의 완전한 프로그래밍** (패턴 R3):

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

### program.md vs SKILL.md vs HAND.toml

| 측면 | program.md | SKILL.md | HAND.toml |
|------|-----------|----------|-----------|
| **포맷** | Markdown (자연어) | Markdown (자연어) | TOML (구조화) |
| **루프** | 무한 (NEVER STOP) | 1회 호출 | activate/pause/resume/deactivate |
| **상태 관리** | Git (코드 버전) | 외부 (DB/파일) | hand_state.json |
| **자동 평가** | [O] (val_bpb) | [X] | [X] |
| **고유성** | 완전한 무한 루프 + 자동 성공 판정 + Git = 상태 머신 | - | - |

---

## 3. Keep/Discard 메커니즘

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

---

## 4. 모델 아키텍처 (에이전트가 수정하는 대상)

train.py에 구현된 GPT 모델의 주요 혁신:

| 혁신 | 구현 | 설명 |
|------|------|------|
| **ResFormer** | `resid_lambdas`, `x0_lambdas` (train.py:133-134) | 레이어별 잔차 가중치 (동적 깊이 제어) |
| **Value Embedding** | `ve_gate` (train.py:73-86) | Alternating 레이어에 입력 의존 gate로 v 보강 |
| **relu-squared** | `F.relu(x).square()` (train.py:106) | GELU/ReLU 대신 실험적 활성화 |
| **Soft-capping** | `softcap * tanh(logits / softcap)` (train.py:281) | logits [-15, 15] 범위 제한 |
| **MuonAdamW** | 이중 옵티마이저 (train.py:296-426) | 2D 행렬: Muon (극좌표 정규화), 나머지: AdamW |

**에이전트 수정 범위**:
```
[O] 수정 가능: 활성화 함수, 어텐션 구조, 초기 가중치, LR 스케줄, 하이퍼파라미터
[X] 수정 불가: prepare.py, evaluate_bpb(), 패키지 설치, TIME_BUDGET
```

---

## 5. BPB (Bits Per Byte) 메트릭

```python
# prepare.py:342-364
def evaluate_bpb(model, tokenizer, batch_size):
    # per-token cross-entropy를 합산
    # 타겟 바이트 길이를 합산
    # nats/byte -> bits/byte 변환
    return total_nats / (math.log(2) * total_bytes)
```

- **Vocab-size 독립**: 아키텍처 간 공정한 비교
- **베이스라인**: val_bpb = 0.997900, VRAM = 44.0 GB, MFU = 39.80%

---

## Claw 패턴 매핑

### prepare.py 읽기전용 = IronClaw Capability Attenuation

| 측면 | prepare.py 읽기전용 | IronClaw Capability |
|------|-------------------|-------------------|
| **격리 수준** | 파일 시스템 (수정 불가 파일) | Capability 시스템 (ApprovalRequirement) |
| **메커니즘** | 사회적 계약 (program.md 지시) | 암호학적 강제 (WASM fuel/memory) |

### 5분 고정 예산 = ZeroClaw 비용 한도

| 측면 | Autoresearch 5분 예산 | ZeroClaw $5/일 |
|------|---------------------|---------------|
| **단위** | 시간 (wall-clock) | 비용 (달러) |
| **목적** | 실험 공정 비교 + 자원 제어 | API 비용 폭발 방지 |

### Autoresearch results.tsv vs ZeroClaw Soul Snapshot

두 시스템 모두 **Git을 상태 관리 시스템으로 활용**. Autoresearch는 "코드 상태"를, ZeroClaw는 "기억 상태"를 Git으로 추적.
