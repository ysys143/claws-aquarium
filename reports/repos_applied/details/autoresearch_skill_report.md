# autoresearch-skill 상세 분석 보고서

> **소스**: GitHub [olelehmann100kMRR/autoresearch-skill](https://github.com/olelehmann100kMRR/autoresearch-skill)
> **조사일**: 2026-03-20
> **연관 분석**: Hermes Agent (12th framework), Karpathy autoresearch

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub** | olelehmann100kMRR/autoresearch-skill |
| **Stars** | 294 |
| **Forks** | 39 |
| **생성일** | 2026-03-18 (조사 시점 기준 이틀 전) |
| **파일** | SKILL.md + eval-guide.md (2개) |
| **언어** | 없음 (순수 마크다운) |
| **라이선스** | 없음 |

---

## 핵심 개념

> "Most skills work about 70% of the time. The other 30% you get garbage. The fix isn't to rewrite the skill from scratch. It's to let an agent run it dozens of times, score every output, and tighten the prompt until that 30% disappears."

autoresearch-skill이 출발하는 문제의식은 단순하다. 스킬(SKILL.md)은 처음 만들 때 잘 작동하는 것처럼 보이지만, 다양한 입력에 반복 실행하면 30% 정도는 기대에 못 미치는 출력이 나온다. 대부분의 사람은 이 시점에서 스킬을 처음부터 다시 쓰거나 막연히 수정한다. autoresearch-skill은 이 접근이 틀렸다고 주장한다.

핵심 아이디어는 Andrej Karpathy의 autoresearch에서 직접 빌려왔다. Karpathy는 ML 연구에서 에이전트가 train.py를 자율적으로 수정하고, 실험하고, 성능이 개선되면 커밋, 나빠지면 reset하는 루프를 설계했다. autoresearch-skill은 이 구조를 그대로 가져와 최적화 대상을 `train.py`에서 `SKILL.md`로, 평가 지표를 `val_bpb`에서 `바이너리 eval 통과율`로 바꿨다. 도메인만 다를 뿐 실험 과학의 논리는 동일하다.

이 도구 자체는 코드가 한 줄도 없다. 두 개의 마크다운 파일 — 실험 루프를 기술하는 `SKILL.md`와 좋은 eval 작성법을 설명하는 `eval-guide.md` — 만으로 이루어져 있다. Claude Code가 이 지침을 읽고 모든 실행을 직접 담당한다.

---

## 아키텍처

### 파일 구조

```
autoresearch-skill/
├── SKILL.md       # 실험 루프 지침
└── eval-guide.md  # 바이너리 eval 작성 가이드
```

autoresearch-skill의 아키텍처라고 할 만한 것은 사실상 이 두 파일뿐이다. 코드가 없는 대신 Claude Code 에이전트 자체가 런타임 역할을 한다. 에이전트는 SKILL.md의 지침을 읽고 실험을 계획하며, 스킬을 실행하고 채점하고, 결과에 따라 keep/discard를 결정하는 모든 과정을 자율적으로 처리한다.

### 실험 실행 시 생성되는 산출물

실험을 돌리면 target 스킬의 디렉토리 안에 `autoresearch-[skill-name]/` 폴더가 생성된다. 여기에 네 가지 파일이 쌓인다.

```
autoresearch-[skill-name]/
├── dashboard.html     # 브라우저에서 실시간으로 볼 수 있는 실험 현황판
├── results.json       # dashboard.html이 읽는 데이터 파일
├── results.tsv        # 실험별 점수, 상태, 설명을 기록한 로그
├── changelog.md       # 각 변이의 의도, 결과, 실패 패턴을 기록한 연구 노트
└── SKILL.md.baseline  # 실험 시작 전 원본 스킬 백업
```

이 중 가장 가치 있는 것은 `changelog.md`다. 어떤 변이가 왜 효과가 있었는지, 왜 없었는지를 실험별로 기록하기 때문에, 미래의 에이전트나 더 스마트한 모델이 이 로그를 읽고 최적화를 이어받을 수 있다. 결과물이 개선된 SKILL.md 하나로 끝나는 것이 아니라, 그 스킬의 실험 역사 전체가 남는다.

`dashboard.html`은 Chart.js를 이용한 점수 진행 그래프와 실험별 keep/discard 현황을 10초마다 자동으로 갱신한다. 실험이 돌아가는 동안 브라우저에서 실시간으로 볼 수 있도록 설계되어 있다.

---

## 실험 루프 — 6단계

autoresearch-skill의 핵심은 6단계로 구성된 실험 프로토콜이다.

**1단계: context 수집.** 실험을 시작하기 전에 에이전트는 반드시 사용자로부터 여섯 가지를 확인한다 — 대상 스킬, 테스트 입력 3-5개, eval 기준 3-6개, 실험당 실행 횟수(기본 5회), 실험 주기(기본 2분), 실험 횟수 상한(기본 없음). 이 확인 없이는 실험을 시작하지 않는다.

**2단계: eval suite 빌드.** 사용자가 제시한 평가 기준을 바이너리 질문으로 변환한다. "출력이 좋은가?"가 아니라 "출력에 번호가 전혀 없는가?"처럼 예/아니오로만 답할 수 있어야 한다. eval이 모호하면 스킬이 eval을 게임하는 방향으로 최적화된다.

**3단계: 대시보드 생성.** 실험 전에 dashboard.html을 먼저 만들고 브라우저를 연다. 이는 실험 중에 사용자가 현황을 볼 수 있도록 하기 위한 것이지만, 동시에 실험이 시작됐다는 신호이기도 하다.

**4단계: baseline 측정.** 아무것도 바꾸기 전에 원본 스킬을 N회 실행해 기준 점수를 잡는다. 이 점수가 90% 이상이면 사용자에게 최적화가 필요한지 물어본다. baseline 없이 변이를 시작하면 개선 여부를 알 수 없다.

**5단계: 자율 실험 루프 (NEVER STOP).** 이 단계부터는 사용자 개입 없이 에이전트가 스스로 돈다. 실패 패턴 분석 → 단일 가설 설정 → SKILL.md 한 가지만 변경 → N회 실행 → 채점 → 점수 개선 시 keep, 아니면 SKILL.md 이전 버전으로 revert → 반복. "NEVER STOP"은 Karpathy autoresearch에서 그대로 가져온 표현이다. 사용자가 자리를 비워도 계속 돌라는 뜻이다.

**6단계: 결과 전달.** 사용자가 돌아오거나 루프가 멈추면 baseline 점수, 최종 점수, 시도한 변이 수, keep/discard 비율, 가장 효과 있었던 변화 top 3, 여전히 실패하는 패턴을 정리해 전달한다.

### 멈춤 조건

루프는 세 가지 조건 중 하나가 되어야 멈춘다 — 사용자 수동 중단, 설정한 실험 횟수 도달, 또는 95% 이상 통과율이 3회 연속 이어지는 경우. 마지막 조건은 "수익 감소" 판단이다. 그 이상을 쥐어짜는 것은 오버피팅 위험이 커진다.

---

## 핵심 원칙

### 바이너리 eval — "scales compound variability"

autoresearch-skill이 가장 강하게 주장하는 원칙은 eval이 반드시 이진수여야 한다는 것이다. 이유는 통계적이다. 4개의 eval이 각각 1-7 점수를 쓰면 총점의 분산이 너무 커져서 실험 간 신호를 신뢰할 수 없다. 스킬이 실제로 나빠졌는지, 아니면 그냥 운이 나빴는지 구분이 안 된다. eval-guide.md는 이를 막기 위해 나쁜 eval과 좋은 eval의 예시를 카테고리별로(텍스트, 시각, 코드, 문서) 상세하게 제공한다.

좋은 eval의 기준은 세 가지다. 두 에이전트가 같은 출력을 보고 같은 답을 낼 수 있어야 하고, 스킬이 실제 개선 없이 eval만 통과하는 방향으로 게임할 수 없어야 하며, 사용자가 실제로 신경 쓰는 것을 측정해야 한다.

### 단일 변이 원칙

한 실험에서 변이는 딱 하나다. 5가지를 동시에 바꾸면 무엇이 효과를 냈는지 모르고, 다음 실험에 살릴 인사이트가 없다. 변이의 종류는 구체적 지침 추가, 모호한 표현 구체화, 잘못된 출력에 대한 anti-pattern 추가, 중요한 지침을 더 앞으로 이동, 예시 추가, 오히려 방해가 되는 지침 제거 등이다. 전체를 다시 쓰는 것은 나쁜 변이다.

### changelog — 지식의 축적

개선된 SKILL.md보다 changelog.md가 더 가치 있다는 주장이 흥미롭다. SKILL.md는 결과물이지만 changelog는 그 결과에 이르는 과정 — 무엇을 시도했고 왜 안 됐는지 — 을 담는다. 이 기록이 있으면 미래에 더 좋은 모델이 등장했을 때 처음부터 실험하지 않고 이어받을 수 있다. autoresearch-skill은 스킬 최적화를 일회성 작업이 아닌 누적 가능한 연구로 취급한다.

---

## Karpathy autoresearch와의 비교

두 도구는 도메인만 다를 뿐 구조가 동일하다.

| 항목 | Karpathy autoresearch | autoresearch-skill |
|------|----------------------|-------------------|
| **최적화 대상** | train.py (ML 학습 코드) | SKILL.md (프롬프트 지침) |
| **평가 지표** | val_bpb (bits per byte) | 바이너리 eval 통과율 |
| **시간 예산** | 5분 고정 | 실험 횟수 설정 |
| **keep/discard** | val_bpb 개선 → keep | 점수 개선 → keep |
| **실험 로그** | results.tsv | results.tsv |
| **자율성** | NEVER STOP | NEVER STOP |
| **되돌리기** | git reset | SKILL.md 이전 버전으로 revert |

Karpathy가 ML 코드를 최적화할 때 쓴 철학 — 자율 실험, 단일 변수 변경, 객관적 지표, 영구 로그 — 이 프롬프트 엔지니어링에도 그대로 적용된다는 것이 이 도구의 핵심 주장이다.

---

## Hermes Agent와의 관계

### 같은 아이디어, 다른 구조화 수준

Hermes Agent의 `skill_manager_tool`은 에이전트가 SKILL.md를 create/edit/patch/delete하는 기능을 내장하고 있고, README에서도 "Skills self-improve during use"를 명시한다. 즉 두 도구는 같은 아이디어 — SKILL.md를 에이전트가 자율적으로 개선한다 — 를 구현하고 있다.

차이는 구조화 수준에 있다. Hermes는 에이전트가 스킬을 사용하다가 문제를 느끼면 암묵적으로 수정한다. 언제, 어떤 기준으로, 얼마나 변경했는지는 에이전트의 판단에 맡긴다. 반면 autoresearch-skill은 이 과정을 명시적으로 과학화한다 — eval 기준을 먼저 정의하고, 하나씩만 바꾸고, 모든 시도를 기록한다.

| 차원 | Hermes Agent | autoresearch-skill |
|------|-------------|-------------------|
| **트리거** | 자동 (에이전트 판단) | 명시적 (사용자 호출) |
| **평가 방법** | 암묵적 | 바이너리 eval (정량) |
| **변이 전략** | 암묵적 | 단일 변이 원칙 |
| **실험 로그** | 없음 | results.tsv + changelog.md |
| **시각화** | 없음 | live HTML dashboard |
| **재현성** | 낮음 | 높음 |

### Hermes에 설치해서 쓰는 플러그인

결합 방식도 단순하다. autoresearch-skill은 SKILL.md 포맷으로 되어 있으므로 Hermes의 `~/.hermes/skills/`에 복사하면 바로 쓸 수 있다. Hermes가 런타임을 담당하고, autoresearch-skill이 실험 프로토콜을 제공하는 플러그인 관계다. Hermes의 `skill_manager_tool`이 스킬 파일 수정을 실행하고, `session_search_tool`이 FTS5로 과거 실험 기록을 검색하는 식으로 자연스럽게 협력한다. agentskills.io 표준도 공유하므로 호환성 문제가 없다.

---

## 포지션

autoresearch-skill은 프레임워크가 아니라 Claude Code 스킬 생태계를 위한 **메타 도구**다. 스킬을 만드는 것이 아니라 이미 있는 스킬을 과학적으로 개선하는 데 쓴다. repos_applied에 편입한 이유는 SKILL.md 포맷이 Claw 생태계의 핵심 인터페이스이고 (ClawWork가 선례를 보였다), Hermes Agent의 스킬 시스템과 직접 결합 가능하기 때문이다.

---

## 한계

autoresearch-skill의 가장 큰 약점은 eval 품질에 전적으로 의존한다는 점이다. 나쁜 eval을 쓰면 스킬은 eval만 통과하는 방향으로 수렴하고, 실제 출력 품질은 오히려 나빠질 수 있다. eval-guide.md가 이 문제를 다루기 위해 존재하지만, 좋은 eval을 정의하는 것 자체가 어려운 작업이다.

오버피팅도 경계해야 한다. 3-5개의 test input으로만 실험하면 그 입력에는 최적화되지만 다른 입력에서는 오히려 퇴보할 수 있다. 실험 입력의 다양성이 결과의 일반화 가능성을 결정한다.

코드가 없는 구조도 양날의 검이다. 에이전트가 지침을 어떻게 해석하느냐에 따라 실험 품질이 달라진다. 같은 SKILL.md도 Claude 버전이나 설정에 따라 다르게 동작할 수 있고, 재현성이 완전히 보장되지 않는다.

라이선스가 없고 생성된 지 이틀밖에 안 된 프로젝트라는 점도 신뢰도 측면에서 감안해야 한다.

---

## 열린 질문

- **Q20**: 좋은 eval을 자동으로 생성할 수 있는가? eval 설계 자체를 autoresearch-skill로 최적화하면 어떻게 되는가?
- **Q21**: 오버피팅을 막기 위한 holdout test set을 두는 방식이 효과적인가?
- **Q22**: autoresearch-skill 자체를 autoresearch-skill로 최적화하면 어떤 일이 벌어지는가? (self-referential loop)

---

## 참고 링크

- [GitHub — olelehmann100kMRR/autoresearch-skill](https://github.com/olelehmann100kMRR/autoresearch-skill)
- [Karpathy autoresearch](https://github.com/karpathy/autoresearch) — 원본 방법론
- [Hermes Agent 상세 보고서](../../repos/details/hermes_agent_report.md)
