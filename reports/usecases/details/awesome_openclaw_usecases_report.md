# awesome-openclaw-usecases 상세 분석 보고서

> **소스**: GitHub hesamsheikh/awesome-openclaw-usecases
> **조사일**: 2026-03-21
> **유형**: 커뮤니티 어썸 리스트 (Awesome List)

---

## 목차

1. [기본 정보](#1-기본-정보)
2. [핵심 특징](#2-핵심-특징)
3. [구조 분析](#3-구조-분석)
4. [콘텐츠 분析](#4-콘텐츠-분석)
5. [신규 패턴 (R-번호)](#5-신규-패턴-r-번호)
6. [비교 테이블](#6-비교-테이블)
7. [한계](#7-한계)
8. [참고 링크](#8-참고-링크)

---

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub URL** | https://github.com/hesamsheikh/awesome-openclaw-usecases |
| **큐레이터** | hesamsheikh (X: @Hesamation) |
| **커뮤니티** | Discord: Open Source AI Builders |
| **유형** | Awesome List — 커뮤니티 기여 활용 사례 컬렉션 |
| **총 항목 수** | 40개 (영문 README 기준 배지), 실제 파일: 38개, KR README: 36개 |
| **사용 사례 파일** | `usecases/` 디렉토리에 개별 `.md` 파일 38개 |
| **언어** | 영어(EN), 중국어(CN), 한국어(KR) 다국어 README |
| **라이선스** | 별도 LICENSE 파일 없음 — 컨트리뷰션 가이드라인만 존재 |
| **대상 플랫폼** | OpenClaw (이전 명칭: ClawdBot, MoltBot) |
| **Awesome 배지** | [![Awesome](https://awesome.re/badge.svg)](https://awesome.re) 공식 등재 |
| **카테고리 수** | 6개 |
| **크립토 정책** | 명시적 제외 — "No crypto-related use cases" |
| **보안 경고** | README 최상단에 경고문 게시 (제3자 스킬 미감사 명시) |

---

## 2. 핵심 특징

### 왜 이 프로젝트가 존재하는가

README 첫 줄이 이 프로젝트의 존재 이유를 정확히 짚는다:

> "Solving the bottleneck of OpenClaw adaptation: Not ~~skills~~, but finding **ways it can improve your life**."

OpenClaw 생태계의 스킬/플러그인은 이미 수백 개 존재한다. 그러나 기술적으로 스킬을 설치할 수 있다고 해서, 실제로 내 삶이 개선되지는 않는다. 이 어썸 리스트가 채우는 공백은 **"이걸로 뭘 할 수 있나?"** 라는 질문에 대한 커뮤니티 집단지성 답변이다.

이는 프레임워크 생태계 성숙도를 측정하는 중요한 지표다. 스킬 마켓플레이스(ClawHub)가 "어떤 부품이 있나"를 보여준다면, awesome-openclaw-usecases는 "그 부품으로 실제 무엇을 만들었나"를 보여준다. 후자가 없으면 전자는 기술 시연에 그친다.

### 생태계 내 위치

```
OpenClaw 생태계 계층도:
┌─────────────────────────────────────────────┐
│  awesome-openclaw-usecases (이 저장소)        │
│  "실제 사용 사례 40개 — 복제 가능한 레시피"      │
├─────────────────────────────────────────────┤
│  ClawHub / 공식 마켓플레이스                   │
│  "설치 가능한 스킬 카탈로그"                     │
├─────────────────────────────────────────────┤
│  awesome-openclaw-agents                     │
│  "에이전트 구성 패턴 모음"                       │
├─────────────────────────────────────────────┤
│  OpenClaw 런타임 (openclaw/openclaw)           │
│  "실행 엔진"                                   │
└─────────────────────────────────────────────┘
```

### 프로젝트가 만들어 낸 네러티브

"스킬이 아닌 활용법"이라는 프레임은 커뮤니티 기여 방식도 바꾼다. CONTRIBUTING.md는 명시적으로 다음을 요구한다:

- "직접 사용하고 검증한 사례만 제출" (최소 1일 이상 운용)
- "AI로 사용 사례를 생성하지 말 것" — 직접 경험만 허용
- 크립토 관련 사례 전면 배제

이는 단순 아이디어 목록이 아닌 **검증된 실사용 레시피 컬렉션**을 지향하는 것이다.

---

## 3. 구조 분析

### 정보 아키텍처

```
awesome-openclaw-usecases/
├── README.md              # 영어 메인 (40개 항목, 6개 카테고리)
├── README_KR.md           # 한국어 번역 (36개 항목 — 4개 누락, 번역 지연)
├── CONTRIBUTING.md        # 기여 가이드라인 (5개 규칙 + 형식 안내)
└── usecases/              # 개별 사용 사례 파일 38개
    ├── autonomous-project-management.md
    ├── n8n-workflow-orchestration.md
    ├── self-healing-home-server.md
    ├── semantic-memory-search.md
    └── ... (34개 더)
```

### 개별 파일 형식 (usecases/*.md)

각 파일은 CONTRIBUTING.md가 제안한 구조를 대체로 따르되, 기여자마다 밀도 차이가 크다. 실제 파일을 분석한 결과:

**표준 섹션 (대부분 파일에 존재)**:
| 섹션 | 역할 |
|------|------|
| `## Pain Point` | 이 사용 사례가 해결하는 구체적 고통 |
| `## What It Does` | 기능 목록 (불릿 포인트) |
| `## Skills You Need` | 필요한 OpenClaw 스킬/외부 도구 |
| `## How to Set It Up` | 단계별 설정 가이드 |
| `## Key Insights` | 실운용에서 얻은 핵심 교훈 |
| `## Related Links` | 참고 링크 (GitHub, 문서, 원본 블로그) |

**고품질 파일에 추가로 존재하는 섹션**:
- `## AGENTS.md Configuration` — 실제 사용할 AGENTS.md 내용 (`self-healing-home-server.md`, `n8n-workflow-orchestration.md`)
- `## Core Pattern` — 코드/YAML 형식으로 핵심 패턴 시각화 (`autonomous-project-management.md`)
- `## Inspired By` — 원출처 블로그/X 게시물 명시 (출처 추적성 확보)

**실제 예시 — `autonomous-project-management.md`의 STATE.yaml 패턴**:
```yaml
# STATE.yaml - Project coordination file
project: website-redesign
updated: 2026-02-10T14:30:00Z

tasks:
  - id: homepage-hero
    status: in_progress
    owner: pm-frontend
    started: 2026-02-10T12:00:00Z
    notes: "Working on responsive layout"

  - id: api-auth
    status: done
    owner: pm-backend
    completed: 2026-02-10T14:00:00Z
    output: "src/api/auth.ts"

  - id: content-migration
    status: blocked
    owner: pm-content
    blocked_by: api-auth
    notes: "Waiting for new endpoint schema"

next_actions:
  - "pm-content: Resume migration now that api-auth is done"
  - "pm-frontend: Review hero with design team"
```

**실제 예시 — `n8n-workflow-orchestration.md`의 아키텍처 다이어그램**:
```
┌──────────────┐     webhook call      ┌─────────────────┐     API call     ┌──────────────┐
│   OpenClaw   │ ───────────────────→  │   n8n Workflow   │ ─────────────→  │  External    │
│   (agent)    │   (no credentials)    │  (locked, with   │  (credentials   │  Service     │
│              │                       │   API keys)      │   stay here)    │  (Slack, etc)│
└──────────────┘                       └─────────────────┘                  └──────────────┘
```

### 다국어 지원 현황

| README | 항목 수 | 카테고리 수 | 상태 |
|--------|---------|------------|------|
| `README.md` (영어) | 40개 | 6개 | 최신 (마스터) |
| `README_KR.md` (한국어) | 36개 | 6개 | 4개 항목 누락 (번역 지연) |
| `README_CN.md` (중국어) | 미확인 | — | 존재 가능성 있음 |

한국어 버전에서 누락된 4개 항목:
- `X/Twitter Automation` (Social Media)
- `Local CRM Framework` (Productivity)
- `arXiv Paper Reader` (Research & Learning)
- `LaTeX Paper Writing` (Research & Learning)

이는 커뮤니티 번역의 구조적 한계다. 영문 원본이 업데이트될 때 번역본이 동기화되지 않는다.

### 기여 프로세스

```
기여자 여정:
1. usecases/ 에 새 .md 파일 생성
2. Pain Point / What It Does / Prompts / Skills Needed / Related Links 작성
3. README.md의 해당 카테고리 테이블에 행 추가
4. PR 오픈
5. 큐레이터(hesamsheikh) 검토 → 머지

품질 기준:
- 직접 사용하고 하루 이상 검증한 사례만
- AI로 생성한 아이디어 금지
- 크립토 관련 내용 금지
- 접근 방식이 다르면 중복 허용
```

---

## 4. 콘텐츠 분析

### 카테고리별 분포

| 카테고리 | 항목 수 | 비율 | 특징 |
|---------|---------|------|------|
| Productivity | 18개 | 45% | 압도적 1위 — 개인/팀 생산성 자동화 |
| Research & Learning | 6개 | 15% | RAG, 논문 리더, 시맨틱 검색 |
| Social Media | 5개 | 13% | 콘텐츠 소비 자동화 |
| Creative & Building | 5개 | 13% | 콘텐츠 생성 파이프라인 |
| Infrastructure & DevOps | 2개 | 5% | 고기술 사용자 대상 |
| Finance & Trading | 1개 | 3% | 비크립토 금융 (예측 시장) |

### 카테고리별 상세 분석

#### 4-1. Social Media (5개)

이 카테고리는 **정보 소비 자동화** 패턴이 지배적이다. 사용자가 콘텐츠를 직접 소비하는 대신 에이전트가 집약·필터링해서 전달한다.

| 사용 사례 | 핵심 패턴 | 주목 이유 |
|-----------|-----------|-----------|
| Daily Reddit Digest | 서브레딧 선호도 기반 다이제스트 | 가장 단순한 진입점 |
| Daily YouTube Digest | 구독 채널 새 영상 요약 | 콘텐츠 크리에이터 필수 도구 |
| X Account Analysis | 내 계정 정성 분석 | 자기 인식 도구 |
| Multi-Source Tech News Digest | **109개+ 소스** 품질 점수 기반 뉴스 집약 | 규모 면에서 돋보임 |
| X/Twitter Automation | 트윗/답글/DM/검색/기브어웨이 자동화 | TweetClaw 플러그인 의존 |

Multi-Source Tech News Digest가 109개+ 소스(RSS, Twitter/X, GitHub, 웹 검색)를 커버하는 점은 주목할 만하다. 단일 에이전트가 멀티소스 데이터 집약 + 품질 스코어링을 수행하는 구조는 DeepInnovator의 4계층 파이프라인(R4)과 개념적으로 유사하다.

#### 4-2. Creative & Building (5개)

이 카테고리는 **콘텐츠 생성 파이프라인** 패턴이다. 단순 아이디어에서 출판 가능한 자산까지 에이전트가 전 과정을 처리한다.

| 사용 사례 | 파이프라인 단계 |
|-----------|---------------|
| Goal-Driven Autonomous Tasks | 목표 입력 → 일일 태스크 생성 → 자율 실행 → 야간 미니앱 빌드 |
| YouTube Content Pipeline | 아이디어 발굴 → 리서치 → 트래킹 |
| Multi-Agent Content Factory | Discord 채널별 리서치/작성/썸네일 에이전트 분리 |
| Autonomous Game Dev Pipeline | 백로그 선택 → 구현 → 등록 → 문서화 → git commit |
| Podcast Production Pipeline | 게스트 리서치 → 에피소드 개요 → 쇼노트 → 소셜 홍보 |

Multi-Agent Content Factory는 Discord를 에이전트 간 통신 채널로 사용한다. 각 에이전트가 전용 Discord 채널을 점유하고 다른 채널의 출력을 입력으로 사용하는 구조다. 이는 NullClaw나 Hermes Agent의 채널 다중화와 다른 "채널=에이전트 격리 공간" 패턴이다.

Autonomous Game Dev Pipeline의 "Bugs First" 정책도 인상적이다. 백로그에서 신규 기능보다 버그 수정을 우선하는 정책을 AGENTS.md에 명시하는 것은 소프트웨어 개발 관행의 에이전트화다.

#### 4-3. Infrastructure & DevOps (2개)

항목 수는 적지만 기술 밀도가 가장 높다. 둘 다 **보안에 대한 명시적 경고**를 포함하며, 실제 운용 경험에서 나온 "하드코딩 금지" 교훈이 문서화되어 있다.

**Self-Healing Home Server** — 실사용 사례 Nathan의 "Reef" 에이전트 분석:

```
Reef 에이전트 스펙:
- 홈 서버에 상시 실행
- SSH: 전체 홈 네트워크 (192.168.1.0/24)
- kubectl: K3s 클러스터 관리
- 1Password: AI 전용 볼트 (읽기 전용)
- 15개 활성 크론 잡, 24개 커스텀 스크립트
- 처리한 ChatGPT 히스토리: 49,079개 원자 사실 추출

크론 스케줄:
- 15분마다: 칸반 보드 진행 중 태스크 체크
- 매시간: 헬스 체크, Gmail 트리아지
- 6시간마다: 지식 베이스 갱신, self-health check
- 매일 4AM: 야간 브레인스토밍
- 매일 8AM: 모닝 브리핑
- 주간: 인프라 보안 감사
```

Day 1에 API 키 노출 사고를 경험한 Nathan의 교훈이 이 사용 사례 전체에 반영되어 있다. TruffleHog pre-push hook + 로컬 Gitea + CI 스캐닝의 3단계 방어가 구체적인 명령어와 함께 제시된다.

**n8n Workflow Orchestration** — 자격증명 격리 패턴:

이 사례는 순수한 보안 아키텍처 패턴이다. OpenClaw가 n8n 웹훅 URL만 알고 실제 API 키는 n8n 볼트에 격리하는 구조다. Simon Høiberg의 원본 패턴을 구체적인 AGENTS.md 설정으로 변환했다.

세 가지 동시 이점: 가시성(n8n 시각적 UI) + 보안(자격증명 격리) + 성능(결정론적 서브태스크는 LLM 토큰 소모 없음). Hermes Agent의 R22 Tirith Pre-Exec Scanner와 목적은 같지만 접근이 반대다 — Tirith는 에이전트 내부에서 스캔, n8n 패턴은 에이전트 밖으로 자격증명을 이동시킨다.

#### 4-4. Productivity (18개 — 전체의 45%)

생산성 카테고리가 40개 중 18개를 차지하는 것은 커뮤니티 우선순위를 직접 반영한다. 세부 패턴으로 분류하면:

**음성/전화 인터페이스 클러스터** (3개):
- Phone-Based Personal Assistant
- Event Guest Confirmation
- Phone Call Notifications

전화 기반 접근이 3개나 존재하는 것은 흥미롭다. Telegram/Discord 중심인 기존 12개 Claw 프레임워크와 달리, 실사용자들은 "전화기"를 에이전트 채널로 원한다. 전화는 스마트폰 알림보다 더 높은 긴급도 신호를 전달하며, 손을 사용할 수 없는 상황(운전 중 등)에서도 접근 가능하다.

**멀티에이전트 조율 클러스터** (4개):
- Autonomous Project Management (STATE.yaml 패턴)
- Multi-Agent Specialized Team (전략/개발/마케팅/사업)
- Multi-Agent Content Factory (Discord 채널 분리)
- Desktop Cowork (통합 UI + 멀티에이전트)

이 중 Autonomous Project Management의 STATE.yaml 패턴이 가장 아키텍처적으로 중요하다. 별도 섹션(§5)에서 분석한다.

**지식 관리 클러스터** (4개):
- Personal CRM
- Second Brain
- Health & Symptom Tracker
- Todoist Task Manager

Second Brain은 "텍스트를 봇에 보내 기억 → Next.js 대시보드에서 검색"이라는 사용자 대면 파이프라인이다. Research & Learning의 Semantic Memory Search와 기술 스택은 다르지만 해결하는 문제는 동일하다 — OpenClaw 마크다운 메모리의 검색 불가 문제.

**멀티채널 통합 클러스터** (3개):
- Multi-Channel Personal Assistant (Telegram + Slack + 이메일 + 캘린더)
- Multi-Channel Customer Service (WhatsApp + Instagram + 이메일 + Google Reviews)
- Desktop Cowork (WebUI + Telegram + Lark + DingTalk)

Desktop Cowork의 Lark/DingTalk 지원은 주목할 만하다. 중국 기업용 메신저를 지원하는 사용 사례가 영문 어썸 리스트에 포함된 것은 OpenClaw의 글로벌 사용자 분포를 반영한다.

#### 4-5. Research & Learning (6개)

이 카테고리는 **외부 지식 소스의 에이전트화** 패턴이다.

| 사용 사례 | 외부 소스 | 핵심 기술 |
|-----------|-----------|-----------|
| AI Earnings Tracker | 기업 실적 발표 | 자동 알림 + 요약 |
| Personal Knowledge Base (RAG) | URL/트윗/기사 | RAG 파이프라인 |
| Market Research & Product Factory | Reddit/X | Last 30 Days 스킬 |
| Pre-Build Idea Validator | GitHub/HN/npm/PyPI/PH | 자동 경쟁 분석 |
| Semantic Memory Search | OpenClaw 메모리 파일 | Milvus + BM25 하이브리드 |
| arXiv Paper Reader | arXiv 논문 | ID 기반 fetch + 섹션 브라우징 |
| LaTeX Paper Writing | 없음 (로컬) | LaTeX 컴파일 + PDF 미리보기 |

Semantic Memory Search는 standalone Python CLI(memsearch, Zilliz 개발)를 OpenClaw 외부에서 연동하는 구조다. SHA-256 콘텐츠 해시로 변경된 파일만 재임베딩하는 스마트 중복 제거가 핵심이다. 이는 OpenJarvis의 R11(Trace→LoRA)처럼 에이전트 데이터를 외부 파이프라인으로 처리하는 패턴의 사용자 수준 구현이다.

Pre-Build Idea Validator는 DeepInnovator의 Authenticity Discriminator(R1)와 방향이 반대다. R1은 "생성된 아이디어가 진짜인가?" 검증이고, Pre-Build Validator는 "이 아이디어의 시장 공간이 열려있나?" 검증이다. 둘 다 아이디어 파이프라인의 게이트키퍼다.

#### 4-6. Finance & Trading (1개)

Polymarket Autopilot 단 1개. 크립토 배제 정책으로 인해 이 카테고리가 자연스럽게 소규모가 된다. Polymarket은 예측 시장(크립토 거래소 아님)이므로 허용된다. 백테스팅 + 전략 분석 + 일일 성과 리포트를 포함한 "페이퍼 트레이딩"으로 실제 자금 위험을 피하는 구조다.

### 사용 채널 분석

40개 사용 사례에서 언급된 채널/인터페이스를 집계하면:

| 채널/인터페이스 | 언급 빈도 | 비고 |
|---------------|----------|------|
| Telegram | 8개+ | 압도적 1위 |
| 이메일 (Gmail) | 6개+ | 비동기 채널 선호 |
| 전화/SMS | 4개 | 고긴급도 알림 |
| Jira/Linear/Todoist | 4개 | 작업 관리 통합 |
| Discord | 3개 | 멀티에이전트 조율 |
| Slack | 3개 | 기업용 |
| WhatsApp | 2개 | 고객 서비스 |
| Calendar (Google/iCal) | 4개 | 가족/팀 조율 |
| Lark/DingTalk | 1개 | 중국 기업용 |
| WebUI | 3개 | 직접 인터페이스 |

Telegram이 압도적 1위인 것은 NullClaw(19채널 지원) 연구 결과와 일치한다. 실사용자들이 Telegram을 1순위 에이전트 채널로 선택하는 이유는 무료 알림, 봇 API 단순성, 채널/그룹 다중화 지원이다.

---

## 5. 신규 패턴 (R-번호)

MEMORY.md의 기존 R1–R28 패턴과 대조하여 신규 여부를 평가한다.

### 후보 1: STATE.yaml 분산 조율 패턴

**`autonomous-project-management.md`에서 발견**

**기존 패턴과의 비교**:
- R13 AgentConfigEvolver (OpenJarvis): TOML config 자동 진화 — 에이전트 설정 파일 진화
- R14 Training=Agent Format (OpenJarvis): 실행 = 훈련 데이터 생성
- R21 Bounded Delegation Tree (Hermes Agent): MAX_DEPTH=2, 중앙 오케스트레이터 기반

STATE.yaml 패턴은 **중앙 오케스트레이터를 제거하고 공유 파일을 통해 에이전트들이 자율 조율**하는 구조다. 이는 기존 패턴과 다르다:

```
기존 멀티에이전트 패턴:
Main Agent → spawns → Sub-Agent 1
           → spawns → Sub-Agent 2
           → spawns → Sub-Agent 3
           (메인이 조율 병목)

STATE.yaml 패턴:
Main Agent (최소 역할)
    ↕ reads/writes
STATE.yaml ←→ Sub-Agent 1
            ←→ Sub-Agent 2
            ←→ Sub-Agent 3
           (파일이 조율 매체)
```

이 패턴의 핵심 인사이트:
1. 파일 기반 조율이 메시지 패싱 오케스트레이터보다 확장성 좋음
2. Git을 STATE.yaml 버전 관리에 사용 → 전체 감사 로그
3. 메인 세션 부하 최소화 ("0-2 tool calls max")
4. 서브에이전트가 `blocked_by` 필드로 의존성 선언

R21(Bounded Delegation Tree)이 "깊이 제한이 있는 트리 구조"라면, STATE.yaml 패턴은 "깊이 무제한의 수평 조율"이다. 방향이 다르다.

**신규 패턴 판정: 신규 (R31 부여)**

---

**R31: Shared-State File Coordination** — 중앙 오케스트레이터 없는 분산 멀티에이전트 조율 패턴.
구현: awesome-openclaw-usecases / autonomous-project-management.md
원리: 에이전트들이 공유 YAML/JSON 파일을 단일 소스로 읽기/쓰기하며 조율. 메인 에이전트는 전략 결정만 담당 (CEO 패턴). 서브에이전트는 `status`, `blocked_by`, `next_actions` 필드로 의존성 선언 및 진행상태 공유. Git 버전 관리로 변경 이력 감사.
시사점: 오케스트레이터 병목 없이 수십 개 에이전트가 병렬 실행 가능. 파일 기반이므로 에이전트 충돌 후 재개 가능(idempotent 재실행). Autoresearch의 results.tsv(R3, Git-as-state-machine)를 멀티에이전트 조율로 확장한 패턴.

---

### 후보 2: n8n 자격증명 격리 패턴

**`n8n-workflow-orchestration.md`에서 발견**

기존 패턴 검토:
- R22 Tirith Pre-Exec Scanner: 명령 실행 전 외부 바이너리 스캔
- R24 Platform-Controlled Allowlist: 플랫폼 벤더가 채널 원격 제어
- Hermes Agent: 1Password CLI로 자격증명 관리

n8n 패턴은 "에이전트가 자격증명을 아예 알지 못하도록 아키텍처적으로 강제"한다. 웹훅 URL만 에이전트 컨텍스트에 노출되고, 실제 API 키는 별도 서비스(n8n)가 보유한다.

이 패턴은 IronClaw의 "Zero-Exposure 자격증명"(에이전트가 `secret-exists()`만 호출)과 개념이 유사하다. 그러나 IronClaw는 WASM 내부 런타임 격리이고, n8n 패턴은 외부 서비스 위임이다. 이미 기존 Tier 1 보안 패턴의 변형으로 볼 수 있으므로 독립 R번호 부여는 불필요하다.

**신규 패턴 판정: 기존 Zero-Exposure 변형 — R번호 미부여**

### 후보 3: HEARTBEAT.md 크론 기반 상시 에이전트 패턴

**`self-healing-home-server.md`에서 발견**

R9(Sleep Consolidation Loop, always-on-memory-agent)와 비교:
- R9는 30분 주기 메모리 통합에 특화
- HEARTBEAT.md 패턴은 15분~주간 멀티스케줄 + 자가 치유 + 이메일 트리아지 + 인프라 관리까지 포괄

그러나 "크론 기반 상시 에이전트"의 개념 자체는 Autoresearch(R3, Fixed-Budget Loop)과 R9에서 이미 다뤄졌다. HEARTBEAT.md는 구현 세부사항이지 새로운 아키텍처 원리는 아니다.

**신규 패턴 판정: R3/R9의 구현 변형 — R번호 미부여**

### 후보 4: 외부 도구를 에이전트 메모리 계층으로 격상

**`semantic-memory-search.md`에서 발견**

memsearch(standalone Python CLI)를 OpenClaw 메모리 파이프라인에 연결해 벡터 검색을 추가하는 구조. OpenJarvis의 Qdrant 벡터 DB 계획(R10 인용 문서)과 유사하지만, 외부 도구를 후설치(post-hoc)로 연결한다는 점에서 다르다.

기존 memory_architecture_report.md에서 다룬 "3-tier maturity" 패턴과 겹친다.

**신규 패턴 판정: 기존 메모리 계층 패턴의 구현 사례 — R번호 미부여**

### 최종 신규 패턴 결론

| 후보 | 판정 | 근거 |
|------|------|------|
| STATE.yaml 분산 조율 | **R31 부여** | R21(Bounded Delegation Tree)과 방향성 상이, 파일 기반 수평 조율로 독립성 인정 |
| n8n 자격증명 격리 | 미부여 | IronClaw Zero-Exposure 변형 |
| HEARTBEAT.md 멀티크론 | 미부여 | R3/R9 구현 변형 |
| 외부 벡터 검색 연동 | 미부여 | 기존 메모리 계층 패턴 구현 사례 |

---

## 6. 비교 테이블

awesome-openclaw-usecases를 유사 리소스와 비교한다.

| 비교 항목 | awesome-openclaw-usecases | awesome-openclaw-agents | ClawHub (공식 마켓) | OpenClaw 공식 docs/showcase |
|-----------|--------------------------|------------------------|--------------------|-----------------------------|
| **항목 수** | 40개 | 미확인 (별도 저장소) | 수백개+ 스킬 | 소수 공식 사례 |
| **카테고리 체계** | 6개 (사용 도메인 기준) | 에이전트 유형 기준 | 스킬 유형 기준 | 미분류 |
| **AGENTS.md 포함** | 고품질 파일에 포함 (약 20%) | 핵심 내용 | 없음 | 없음 |
| **SOUL/설정 포함** | 일부 포함 | 중심 내용 | 스킬 설정 중심 | 없음 |
| **코드 포함** | YAML/bash/JSON 예시 포함 | 에이전트 설정 중심 | 설치 명령 중심 | 없음 |
| **큐레이션 기준** | 직접 검증 필수, AI 생성 금지 | 미확인 | 스킬 기능 기준 | Anthropic 자체 선정 |
| **크립토 정책** | 명시적 배제 | 미확인 | 정책 없음 | 해당 없음 |
| **다국어 지원** | EN/KR/CN | 미확인 | EN 중심 | EN 중심 |
| **보안 경고** | README 최상단 명시 | 미확인 | 없음 | 없음 |
| **출처 추적성** | "Inspired By" 섹션 | 미확인 | 없음 | 공식 출처 |
| **사용 목적** | 실생활 적용 패턴 발견 | 에이전트 아키텍처 학습 | 스킬 설치 | 공식 기능 확인 |
| **신뢰 모델** | 커뮤니티 자율 검증 | 미확인 | 벤더 관리 | Anthropic 보증 |

### 어썸 리스트 생태계 내 포지션

```
OpenClaw 생태계 정보 레이어:

What is OpenClaw?  →  openclaw/openclaw (공식 문서)
What can it do?    →  awesome-openclaw-usecases (이 저장소) ← 공백 공략
How to build?      →  awesome-openclaw-agents (에이전트 패턴)
Which skills?      →  ClawHub (스킬 마켓플레이스)
Who built what?    →  openclaw/openclaw showcase
```

---

## 7. 한계

### 7-1. 단일 큐레이터 편향

모든 PR이 hesamsheikh 단 1인을 통과한다. 이는 일관성을 보장하지만 동시에:
- 큐레이터 시간이 병목이다
- 큐레이터의 관심사/배경이 수집된 사례에 편향을 준다
- 큐레이터 비활성화 시 프로젝트가 정체된다

README_KR.md의 4개 항목 누락이 이미 유지보수 부하를 보여준다.

### 7-2. 품질 게이트의 구조적 약점

CONTRIBUTING.md는 "직접 검증한 사례만"을 요구하지만 이를 기술적으로 강제할 수단이 없다. "최소 하루 이상 운용"의 증거를 요구하지 않는다. AI로 생성한 사례를 배제하는 규칙도 선언에 그친다. 결과적으로 품질은 기여자 양심에 의존한다.

### 7-3. 영어 중심 한계

KR/CN README 번역본이 존재하지만 개별 사용 사례 파일(`usecases/*.md`) 자체는 모두 영어다. 한국어/중국어 사용자가 상세 설정을 따라가려면 영어 원문을 읽어야 한다. 다국어 README는 진입장벽을 낮추지만 실제 구현 단계의 언어 장벽은 유지된다.

### 7-4. 보안 감사 부재

README의 경고문은 솔직하지만 해결책이 없다:

> "Many use cases link to community-built skills, plugins, and external repos that have **not been audited by the maintainer**."

사용자가 개별 스킬의 소스코드를 직접 검토해야 한다. Hermes Agent의 R20(Skills Trust Levels — builtin/trusted/community/agent-created 4단계)이나 R22(Tirith Pre-Exec Scanner)처럼 자동화된 신뢰 분류 체계가 없다. awesome-openclaw-usecases는 링크 큐레이션만 하며, 링크가 가리키는 스킬의 안전성은 보증하지 않는다.

### 7-5. 크립토 배제의 공백

크립토 배제 정책은 스팸/사기 방지 목적으로 타당하지만, 실질적인 DeFi 자동화, 온체인 데이터 분석, 블록체인 개발 도구 등 적법한 기술 사용 사례까지 배제된다. Finance 카테고리가 Polymarket Autopilot 단 1개인 것은 이 정책의 직접 결과다.

### 7-6. 복제 가능성의 불균등

40개 사용 사례의 복제 가능성은 편차가 크다:

| 복제 난이도 | 해당 사례 수 | 예시 |
|------------|------------|------|
| 쉬움 (설정 복붙) | 약 15개 | Inbox Declutter, Daily Reddit Digest |
| 보통 (외부 서비스 연동 필요) | 약 15개 | Personal CRM, Habit Tracker |
| 어려움 (인프라 세팅 필요) | 약 10개 | Self-Healing Server, n8n Stack, Local CRM |

어려운 사례들은 Docker, Kubernetes, 1Password CLI 등 선행 인프라를 요구한다. 이는 진입 장벽이 높은 사용자에게는 실제로 복제 불가능한 사례다.

### 7-7. 상태 관리 부재

사용 사례가 현재도 작동하는지 알 수 없다. OpenClaw API가 변경되거나 외부 서비스 엔드포인트가 바뀌면 기존 사례가 조용히 무효화된다. "Last tested: 2026-01-15" 같은 검증 날짜 필드가 없다. README의 배지는 최근 커밋 날짜를 표시하지만 이는 사례의 현행 유효성과 무관하다.

---

## 8. 참고 링크

### 소스 저장소
- **GitHub**: https://github.com/hesamsheikh/awesome-openclaw-usecases
- **큐레이터 X**: https://x.com/Hesamation
- **커뮤니티 Discord**: https://discord.gg/vtJykN3t

### 로컬 보고서 (compare_claws)
- 이 파일의 인덱스: `/Users/jaesolshin/Documents/GitHub/compare_claws/reports/usecases/usecases_index.md` (생성 예정)
- 관련 보고서: `/Users/jaesolshin/Documents/GitHub/compare_claws/reports/repos_applied/repos_applied_report.md`
- 보안 기준: `/Users/jaesolshin/Documents/GitHub/compare_claws/reports/repos/security_report.md`
- 메모리 아키텍처: `/Users/jaesolshin/Documents/GitHub/compare_claws/reports/repos/memory_architecture_report.md`

### 참조된 사용 사례 블로그
- Nathan's "Everything I've Done with OpenClaw": https://madebynathan.com/2026/02/03/everything-ive-done-with-openclaw-so-far/
- Simon Høiberg's n8n pattern: https://x.com/SimonHoiberg/status/2020843874382487959

### 참조된 외부 도구
- memsearch (Semantic Memory Search): https://github.com/zilliztech/memsearch
- openclaw-n8n-stack (Docker Compose): https://github.com/caprihan/openclaw-n8n-stack
- TruffleHog (Secret Scanner): https://github.com/trufflesecurity/trufflehog
- OpenClaw 공식: https://github.com/openclaw/openclaw

### MEMORY.md 업데이트 필요 항목
- R31: Shared-State File Coordination 패턴 추가
- repos_applied 항목에 awesome-openclaw-usecases 추가
- 열린 질문: awesome 리스트 품질 보장의 자동화 가능성?

---

*보고서 작성: 2026-03-21*
*분석 대상 파일: README.md, README_KR.md, CONTRIBUTING.md, usecases/autonomous-project-management.md, usecases/n8n-workflow-orchestration.md, usecases/self-healing-home-server.md, usecases/semantic-memory-search.md (총 7개 직접 분석)*
