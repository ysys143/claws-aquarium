# usecases/ 종합 분析 보고서

> **계층**: usecases/ — 커뮤니티 콘텐츠 & 실사용 모음 (4번째 분析 계층)
> **항목 수**: 3개 (awesome-openclaw-usecases, awesome-openclaw-agents, OpenClaw 서울 빌더 밋업 2026-03-15)
> **총 데이터 포인트**: 40개 활용 사례 + 174개 에이전트 템플릿 + 132개 유스케이스 + 16개 밋업 세션 = 362개
> **신규 패턴**: R29, R30, R31 (3개)
> **조사일**: 2026-03-21

---

## 목차

1. [개요 (Executive Summary)](#1-개요-executive-summary)
   - 1.1 usecases/ 계층이란
   - 1.2 3개 항목 요약 표
   - 1.3 핵심 발견 3가지
2. [usecases/ 계층 정의](#2-usecases-계층-정의)
   - 2.1 다른 계층과의 차이
   - 2.2 수요 신호(demand signal) 계층으로서의 역할
   - 2.3 分析 방법론
3. [항목별 심층 分析](#3-항목별-심층-분析)
   - 3.1 awesome-openclaw-usecases
   - 3.2 awesome-openclaw-agents
   - 3.3 OpenClaw 서울 빌더 밋업 2026-03-15
4. [교차 分析](#4-교차-분析)
   - 4.1 도메인 수요 분포
   - 4.2 채널 선호도 격차
   - 4.3 배포 패턴 다양성
   - 4.4 보안 인식 격차
   - 4.5 에이전트 신원 파일 표준 경쟁
   - 4.6 멀티 디바이스 패턴 — OpenClaw의 핵심 설계 방향
5. [기존 보고서와의 연결](#5-기존-보고서와의-연결)
   - 5.1 repos/ 프레임워크와의 교차 검증
   - 5.2 repos_applied/ 항목과의 중복/보완 관계
   - 5.3 보안 보고서와의 연결
   - 5.4 메모리 아키텍처 보고서와의 연결
6. [신규 패턴 (R29–R31) 상세](#6-신규-패턴-r29r31-상세)
7. [결론 및 미해결 질문](#7-결론-및-미해결-질문)

---

## 1. 개요 (Executive Summary)

### 1.1 usecases/ 계층이란

compare_claws 프로젝트는 지금까지 **코드**를 분析 대상으로 삼아왔다. `repos/`는 에이전트 런타임 13개, `repos_applied/`는 그 위에 구축된 응용 프로젝트, `repos_research/`는 연구 자동화 도구를 다뤘다. 이 세 계층 모두 "무엇이 만들어졌는가"를 소스코드로 파악한다.

`usecases/` 계층은 다른 질문을 던진다:

> **"실제 개발자들이 OpenClaw로 무엇을 만들고, 어떻게 운영하며, 어떤 문제를 겪는가?"**

이 질문은 코드 저장소로는 답할 수 없다. 커뮤니티 Awesome 리스트, 에이전트 템플릿 컬렉션, 밋업 트랜스크립트라는 1차 소스 데이터를 통해서만 접근 가능하다. usecases/ 계층은 OpenClaw 생태계의 **수요 신호(demand signal)**이자, 코드 분析에서 보이지 않는 실무 패턴의 보고(寶庫)다.

### 1.2 3개 항목 요약 표

| 이름 | 유형 | 항목 수 | 핵심 발견 | 신규 패턴 |
|------|------|---------|----------|----------|
| **awesome-openclaw-usecases** (hesamsheikh) | Awesome List — 활용 사례 | 40개 use-case, 6개 카테고리 | Productivity 45% 압도; 전화/SMS 채널 수요 4건; STATE.yaml 분산 조율 패턴 발견; 크립토 명시 배제 정책; 보안 경고문 README 최상단 게시 | R31 |
| **awesome-openclaw-agents** (mergisi) | Awesome List — 에이전트 템플릿 | 174개 SOUL.md + 132개 use-case, 24개 카테고리 | marketing(19개) 1위; SOUL.md 표준의 사실상 레퍼런스 라이브러리; crewclaw.com 오픈소스-상용 이중 채널; agents.json 기계 가독형 레지스트리; 보안 검증 전무 | 해당 없음 |
| **OpenClaw 서울 빌더 밋업 2026-03-15** (Instruct.KR) | 커뮤니티 밋업 — 1차 소스 | 16개 세션, 950+ RSVP | AWS Lambda+tsc TypeScript 동적 실행(R29); MEMORY.md 포인터맵(R30); OMC/OMX PR 80%+ 에이전트 생성; ClawCon Seoul 확정; 3-tier 로컬 AI 모델 전략 | R29, R30 |

**총합**: 3개 항목 | 40 + 174 + 132 + 16 = 362개 실사용 데이터 포인트 | 신규 패턴 3개 (R29, R30, R31)

### 1.3 핵심 발견 3가지

**발견 1: 프레임워크 설계 목표와 실제 수요 사이의 구조적 간극**

13개 repos/ 프레임워크 중 "개인 사용자 생산성 자동화"를 명시적 설계 목표로 내세운 것은 없다. 그러나 awesome-openclaw-usecases의 40개 사례 중 18개(45%), awesome-openclaw-agents의 174개 템플릿 중 personal+productivity 합산 약 14개, 밋업 16세션 중 3개가 개인 생산성 자동화다. 프레임워크는 보안, 확장성, 다채널 지원을 설계하는 동안, 커뮤니티는 "나의 일상을 자동화"하는 데 집중한다.

**발견 2: 커뮤니티 콘텐츠 계층의 보안 공백**

repos/ 계층에는 Tier 1(WASM+암호화+다층 방어)부터 Tier A+(OS 레벨 샌드박스)까지 세밀한 보안 분류 체계가 있다. 그러나 usecases/ 계층에는 이에 상응하는 보안 검증이 전혀 없다. awesome-openclaw-usecases는 README에 "커뮤니티 스킬은 감사되지 않았다"고 경고만 하고, awesome-openclaw-agents의 177개 SOUL.md 템플릿은 어떠한 자동 검증도 거치지 않는다. Hermes Agent(R20 Skills Trust 4단계, R22 Tirith Pre-Exec Scanner)가 프레임워크 계층에서 해결한 문제가 커뮤니티 콘텐츠 계층으로 전파되지 않고 있다.

**발견 3: 1차 소스 데이터에서만 발굴 가능한 신규 패턴**

R29(TypeScript-as-Tool 동적 실행), R30(포인터맵 메모리 아키텍처), R31(Shared-State File Coordination) — 이 3개 패턴은 소스코드 분析으로는 발견할 수 없었다. 이상현의 AWS Lambda 실험, 허예찬의 OMC/OMX 운영 경험, awesome-openclaw-usecases의 autonomous-project-management.md 사례는 각각 "실무에서 먼저 발견하고 나중에 문서화"된 패턴이다. usecases/ 계층이 패턴 발굴 파이프라인으로서 독자적 가치를 갖는 이유가 여기에 있다.

---

## 2. usecases/ 계층 정의

### 2.1 다른 계층과의 차이

compare_claws의 계층 구조를 확인하면 usecases/의 위치가 명확해진다:

```
┌─────────────────────────────────────────────────────────────────┐
│  usecases/  "실제 개발자들이 만드는 것 / 운영 방식 / 문제점"      │
│  [수요 신호 & 실무 패턴] — 코드가 아닌 콘텐츠를 분析              │
├─────────────────────────────────────────────────────────────────┤
│  repos_applied/  "프레임워크 위에 구축된 응용 프로젝트"            │
│  [ClawWork, ClawPort, Symphony, Moltbook, MiClaw]               │
├─────────────────────────────────────────────────────────────────┤
│  repos_research/  "AI 연구 자동화 도구"                          │
│  [DeepInnovator, Autoresearch]                                   │
├─────────────────────────────────────────────────────────────────┤
│  repos/  "에이전트 런타임 프레임워크"                             │
│  [OpenClaw, NanoClaw, IronClaw, ZeroClaw ... 13개]              │
└─────────────────────────────────────────────────────────────────┘
```

**분析 대상의 본질적 차이**:

| 계층 | 분析 대상 | 답하는 질문 | 데이터 유형 |
|------|---------|-----------|-----------|
| `repos/` | 소스코드 | 엔진은 어떻게 동작하는가? | 정량 (LOC, API, 보안 메커니즘) |
| `repos_applied/` | 소스코드 | 위에 무엇이 쌓였는가? | 정량 + 설계 결정 |
| `repos_research/` | 소스코드 | 연구를 어떻게 자동화하는가? | 정량 + 알고리즘 |
| `usecases/` | 커뮤니티 콘텐츠 | **실제로 무엇을 만들고 어떻게 쓰는가?** | **정성 (실무 경험, 실패담, 우회 전략)** |

### 2.2 수요 신호(demand signal) 계층으로서의 역할

프레임워크 설계자는 "어떤 기능이 필요한가?"를 예측해야 한다. 하지만 커뮤니티가 실제로 원하는 것은 다를 수 있다. usecases/ 계층이 제공하는 수요 신호는 세 종류다:

1. **채택 신호** (What is being built): awesome-openclaw-usecases의 카테고리 분포, awesome-openclaw-agents의 템플릿 분포가 실제 사용 도메인을 집계한다.

2. **마찰 신호** (Where friction exists): 밋업 세션에서 발표자들이 공유한 우회 전략 — "n8n을 쓰면 자격증명을 격리할 수 있다", "MQTT로 방화벽을 통과한다", "tsc가 tool definition을 대체한다" — 은 프레임워크가 해결하지 못한 마찰 지점을 드러낸다.

3. **갭 신호** (What is missing): 전화/SMS 채널 수요 4건이 있으나 13개 프레임워크 중 지원하는 것이 사실상 없다. 이는 생태계의 명확한 공백이다.

awesome-openclaw-usecases의 README 첫 줄이 이 역할을 정확히 설명한다:

> "Solving the bottleneck of OpenClaw adaptation: Not skills, but finding **ways it can improve your life**."

스킬 카탈로그(ClawHub)가 "어떤 부품이 있나"를 보여준다면, usecases/ 계층은 "그 부품으로 실제 무엇을 만들었나"를 보여준다.

### 2.3 分析 방법론 — Awesome 리스트 vs 밋업 콘텐츠

이 계층의 두 유형은 서로 다른 분析 접근을 요구한다:

| 항목 유형 | 대표 항목 | 강점 | 약점 | 분析 초점 |
|---------|---------|------|------|---------|
| **Awesome List** | awesome-openclaw-usecases, awesome-openclaw-agents | 광범위한 커버리지; 구조화된 데이터(agents.json); 복제 가능한 패턴 | 단일 큐레이터 편향; 실제 운용 여부 미검증; 정적 스냅샷 | 카테고리 분포, 패턴 추출, 보안 검증 체계 |
| **밋업 콘텐츠** | OpenClaw 서울 빌더 밋업 0315 | 1차 소스 정성 데이터; 실패담 포함; 최신 실무 패턴 | 발표자 자기 선택 편향; 한국어 전용; 단일 시점 스냅샷 | 신규 패턴 발굴, 운영 전략, 채택 추세 |

두 유형의 분析을 결합하면 서로의 약점을 보완한다. Awesome 리스트의 "40개 사례 중 전화 인터페이스 4건"이라는 정량 데이터와 밋업의 "이상현이 MQTT로 방화벽을 우회했다"는 정성 사례가 결합되어 "Phone/SMS 채널 지원 부재"라는 갭 신호를 확정한다.

---

## 3. 항목별 심층 分析

### 3.1 awesome-openclaw-usecases (hesamsheikh)

**기본 정보**

| 항목 | 내용 |
|------|------|
| GitHub | https://github.com/hesamsheikh/awesome-openclaw-usecases |
| 유형 | Awesome List (Awesome.re 공식 등재) |
| 큐레이터 | hesamsheikh (X: @Hesamation) |
| 총 항목 | 40개 (배지 기준), 실제 파일 38개 |
| 카테고리 | 6개 |
| 언어 지원 | EN / KR / CN 다국어 README |
| 크립토 정책 | "No crypto-related use cases" 명시적 배제 |
| 보안 경고 | README 최상단에 제3자 스킬 미감사 명시 |

**카테고리별 분포**

| 카테고리 | 항목 수 | 비율 | 특징 |
|---------|---------|------|------|
| Productivity | 18개 | 45% | 압도적 1위 — 개인/팀 생산성 자동화 |
| Research & Learning | 6개 | 15% | RAG, 논문 리더, 시맨틱 검색 |
| Social Media | 5개 | 13% | 콘텐츠 소비 자동화 |
| Creative & Building | 5개 | 13% | 콘텐츠 생성 파이프라인 |
| Infrastructure & DevOps | 2개 | 5% | 고기술 사용자 대상 |
| Finance & Trading | 1개 | 3% | 비크립토 금융 (예측 시장) |

Productivity 카테고리가 45%를 차지한다는 사실은 단순한 통계가 아니다. 이 18개 사례는 다시 세부 클러스터로 나뉜다: 음성/전화 인터페이스(3개), 멀티에이전트 조율(4개), 지식 관리(4개), 멀티채널 통합(3개), 기타(4개). 특히 음성/전화 클러스터 — Phone-Based Personal Assistant, Event Guest Confirmation, Phone Call Notifications — 는 Telegram/Discord 중심인 기존 13개 프레임워크와 다른 실사용 수요를 보여준다.

**주요 패턴 — 아키텍처적으로 중요한 사례들**

**autonomous-project-management.md의 STATE.yaml 패턴** (→ R31 부여, §6 상세):
```yaml
# STATE.yaml — 멀티에이전트 분산 조율의 단일 소스
project: website-redesign
tasks:
  - id: api-auth
    status: done
    owner: pm-backend
  - id: content-migration
    status: blocked
    blocked_by: api-auth
next_actions:
  - "pm-content: Resume migration now that api-auth is done"
```
중앙 오케스트레이터 없이 에이전트들이 공유 파일을 통해 자율 조율하는 이 패턴은 R21(Bounded Delegation Tree, Hermes Agent)의 중앙 집중식 접근과 방향이 반대다.

**n8n-workflow-orchestration.md의 자격증명 격리 패턴**:
```
OpenClaw (에이전트) → webhook URL만 알고 있음
    ↓
n8n Workflow (자격증명 보유) → 실제 API 키로 외부 서비스 호출
    ↓
External Service (Slack, Gmail...)
```
세 가지 동시 이점: 가시성(n8n 시각적 UI) + 보안(자격증명 격리) + 성능(결정론적 서브태스크는 LLM 토큰 소모 없음).

**self-healing-home-server.md의 Nathan "Reef" 에이전트**:
- 15개 활성 크론 잡, 24개 커스텀 스크립트 상시 운용
- 처리한 ChatGPT 히스토리: 49,079개 원자 사실 추출
- Day 1 API 키 노출 사고를 경험한 뒤 TruffleHog pre-push hook + 로컬 Gitea + CI 스캐닝 3단계 방어 구축

**채널 선호도 집계**

40개 사례에서 언급된 채널/인터페이스를 집계하면:

| 채널/인터페이스 | 언급 빈도 | 비고 |
|---------------|----------|------|
| Telegram | 8개+ | 압도적 1위 — 사실상 표준 |
| 이메일 (Gmail) | 6개+ | 비동기 채널 선호 |
| Calendar (Google/iCal) | 4개 | 가족/팀 조율 |
| 전화/SMS | 4개 | 고긴급도 알림 — **프레임워크 지원 미흡** |
| Jira/Linear/Todoist | 4개 | 작업 관리 통합 |
| Discord | 3개 | 멀티에이전트 조율 공간 |
| Slack | 3개 | 기업용 |
| WebUI | 3개 | 직접 인터페이스 |
| WhatsApp | 2개 | 고객 서비스 |
| Lark/DingTalk | 1개 | 중국 기업용 — 글로벌 분포 증거 |

전화/SMS 4건은 현재 13개 프레임워크 어디에서도 1등급 채널로 지원되지 않는다. NullClaw가 19채널을 지원하지만 Voice over IP 기반이다.

**기여 규칙과 품질 게이트**

CONTRIBUTING.md의 요건:
- 직접 사용하고 검증한 사례만 (최소 1일 이상 운용)
- "AI로 사용 사례를 생성하지 말 것" — 직접 경험만 허용
- 크립토 관련 사례 전면 배제

그러나 이 기준을 기술적으로 강제할 수단은 없다. 품질은 기여자 양심과 단일 큐레이터(hesamsheikh) 검토에 전적으로 의존한다. 한국어 README(README_KR.md)의 4개 항목 누락(X/Twitter Automation, Local CRM Framework, arXiv Paper Reader, LaTeX Paper Writing)은 단일 큐레이터 모델의 유지보수 한계를 이미 드러내고 있다.

---

### 3.2 awesome-openclaw-agents (mergisi)

**기본 정보**

| 항목 | 내용 |
|------|------|
| GitHub | https://github.com/mergisi/awesome-openclaw-agents |
| 라이선스 | CC0 1.0 (퍼블릭 도메인) |
| 유형 | Awesome List — 에이전트 템플릿 컬렉션 |
| 템플릿 수 | 177개 (배지), 174개 (agents.json 실측) |
| 유스케이스 수 | 132개 (USE-CASES.md) |
| 카테고리 수 | 24개 (agents/) |
| 관련 플랫폼 | crewclaw.com (원클릭 배포) |

**카테고리별 분포 (agents.json 실측, total=174)**

| 순위 | 카테고리 | 에이전트 수 | 비율 |
|------|----------|------------|------|
| 1 | marketing | 19 | 10.9% |
| 2 | development | 15 | 8.6% |
| 3 | business | 14 | 8.0% |
| 4 | creative | 10 | 5.7% |
| 4 | devops | 10 | 5.7% |
| 4 | finance | 10 | 5.7% |
| 7 | data | 9 | 5.2% |
| 8 | education | 8 | 4.6% |
| 9 | healthcare | 7 | 4.0% |
| 9 | hr | 7 | 4.0% |
| 9 | personal | 7 | 4.0% |
| 9 | productivity | 7 | 4.0% |
| 13–22 | 나머지 12개 | 41 | 23.6% |

marketing(19개)이 1위인 것은 OpenClaw 커뮤니티의 주요 사용자층이 소규모 사업자, 프리랜서, 콘텐츠 크리에이터임을 반영한다. "SEO 에이전트 + 콜드 아웃리치로 월 60건 콜 예약"(USE-CASES.md #39) 같은 사례가 이를 뒷받침한다.

**신규 카테고리 6개의 의미**

| 카테고리 | 항목 수 | 시사점 |
|---------|---------|-------|
| moltbook | 3 | 에이전트-투-에이전트 소셜 레이어 — repos_applied/의 Moltbook과 직접 연동 |
| voice | 3 | 전화 수신/발신 에이전트 — awesome-openclaw-usecases 전화/SMS 수요와 교차 |
| automation | 6 | "자는 동안 500개 이력서 지원" 등 야간 자율 실행 패턴 확산 |
| compliance | 4 | GDPR, SOC2, EU AI Act 규제 준수 자동화 — 기업 고객 진출 신호 |
| supply-chain | 3 | B2B 운영 자동화로 확장 |
| customer-success | 2 | SaaS 기업 고객 성공팀 대상 |

**SOUL.md 표준 — 177개의 참조 구현체**

SOUL.md는 에이전트의 역할, 성격, 행동 규칙, 예시 대화를 단일 Markdown 파일로 캡슐화한다. awesome-openclaw-agents는 이 포맷의 참조 구현체 174개를 한 곳에 모은 사실상의 표준 라이브러리다.

Orion (productivity/orion) SOUL.md의 구조적 특징:

| 섹션 | LLM 관점 | 역할 |
|------|---------|------|
| Core Identity | 페르소나 선언 | 에이전트가 누구인지 1인칭으로 정의 |
| Responsibilities | 작업 범위 | LLM이 처리해야 할 도메인 명시 |
| Behavioral Guidelines | Do/Don't 제약 | 명시적 행동 경계 설정 |
| Example Interactions | Few-shot 예제 | 응답 형식과 품질 기준 제시 |
| Integration Notes | 협업 컨텍스트 | 다른 에이전트와의 관계 명시 |

**agents.json — 기계 가독형 레지스트리**

```json
{
  "total": 174,
  "agents": [
    {
      "id": "competitor-pricing",
      "category": "business",
      "name": "Competitor Pricing",
      "role": "Competitive Pricing Intelligence Agent",
      "path": "agents/business/competitor-pricing/SOUL.md",
      "deploy": "https://crewclaw.com/create-agent"
    }
  ]
}
```

`path` 필드가 SOUL.md 경로를 직접 가리키는 구조는 외부 도구(crewclaw.com 포함)가 이 인덱스를 파싱해 자동 렌더링할 수 있도록 설계되었다. `deploy` 필드는 174개 모두 동일한 crewclaw.com URL을 가리킨다. crewclaw.com이 서비스를 종료하면 배포 링크 전체가 무효화되는 단일 의존성 위험이 존재한다.

**USE-CASES.md — "무엇을 빌드하는가"의 분리**

agents/ 디렉토리(에이전트 역할별 SOUL.md 템플릿, 공급 관점)와 USE-CASES.md(커뮤니티 실제 구축 사례 132개, 수요 관점)를 분리한 것은 이 컬렉션의 주목할 구조적 결정이다.

USE-CASES.md에서 가장 이론적으로 흥미로운 클러스터는 **"Meta Use Cases — Agent Operating on Itself"**(4개)다:

| # | 사례 | 자기참조 메커니즘 |
|---|------|-----------------|
| 129 | Bot Writes Its Own Marketing | 에이전트가 자신의 유스케이스 레포를 찾아 마케팅 페이지로 변환 후 배포 |
| 130 | Self-Updating Skills | 에이전트가 자신의 스킬과 설정을 직접 업데이트 |
| 131 | Agent-to-Human Delegation | 에이전트가 작업을 인간에게 위임하고 비동기로 모니터링 |
| 132 | Physical Body Self-Modification | 로봇 프로토타입이 자신의 코드를 편집해 360도 회전을 스스로 학습 |

R11(Trace→LoRA), R13(AgentConfigEvolver)과 부분적으로 겹치지만, 이 사례들은 아키텍처 패턴보다는 창의적 응용 사례에 가깝다. 체계적 구현이 확인되지 않아 독립 R번호 부여는 보류했다.

**crewclaw.com 통합 — 오픈소스-상용 이중 채널 모델**

| 경로 | 기능 |
|------|------|
| crewclaw.com/agents | 177개 템플릿 브라우저 (GitHub 카탈로그의 UI 레이어) |
| crewclaw.com/create-agent | 원클릭 배포 패키지 ($9 일회성, Dockerfile 포함) |
| crewclaw.com/blog/ | 튜토리얼 & 비교 가이드 |

CC0 퍼블릭 도메인으로 카탈로그를 완전 공개하면서, 배포 편의성은 상용 서비스로 수익화하는 구조다. repos_applied/의 Symphony(Elixir/OTP 자동화)나 ClawPort(대시보드 프록시)와 비교하면, crewclaw.com은 "배포 마찰 제거"라는 단일 가치 명제에 집중하는 얇은 상용 계층이다.

---

### 3.3 OpenClaw 서울 빌더 밋업 2026-03-15

**기본 정보**

| 항목 | 내용 |
|------|------|
| 행사명 | OpenClaw 서울 빌더 밋업 2026-03-15 |
| 주최 | Instruct.KR |
| 수용 인원 | 50-60명 |
| RSVP | 950명 이상 (수용 인원 대비 **16-19배 초과**) |
| 세션 수 | 16개 (오프닝 포함) |
| 글로벌 발표자 | Lionel Sim (싱가포르), Zoe Chen (Unibase), Logan Kang (Base Korea) |
| 주요 의의 | 첫 번째 OpenClaw 서울 공식 행사 + **ClawCon Seoul 공식 확정** |

950+ RSVP / 50-60명 수용이라는 비율(16-19배 초과 신청)은 단순한 인기 지표가 아니다. 한국 AI 에이전트 개발자 커뮤니티의 밀도를 정량적으로 보여주는 데이터다.

**16개 세션 클러스터링**

| 클러스터 | 세션 | 핵심 내용 |
|---------|------|---------|
| 인프라/배포 | 03, 04, 07, 12, 15 | 다중디바이스 원격, 서버리스 Lambda, M3 Ultra 로컬 LLM, NanoClaw 경량, 중고PC 세컨드브레인 |
| 온체인/에이전트 경제 | 02, 06, 14 | X402 프로토콜, Virtuals ACP 에이전트 흥정, Base USDC 정산, ERC-8004 |
| 운영 패턴 | 05, 08, 09, 10, 16 | 스킬 개발 4단계, 툴콜링 최적화, FLOCK 워크플로우, MEMORY.md 포인터맵, Ultraworker |
| 연구 자동화 | 11 | FSM 기반 ResearchClaw, 반도체 RTL 자동 생성, 로봇 하네스 |
| 글로벌 커뮤니티 | 01, 13 | 서울 첫 공식 행사, 싱가포르 커뮤니티 현황, ClawCon Seoul 확정 |

**주요 세션 심층 분석**

**Session 2 — Logan Kang: Composable Agent Economy**

3개 레이어 스택으로 에이전트 간 자율 거래를 구현:
- **OpenClaw**: 에이전트 생성·실행 런타임
- **Virtuals ACP**: 서비스 디스커버리 + 에스크로 + 흥정 프로토콜
- **Base**: 이더리움 L2, USDC 스테이블코인 정산

X402 프로토콜: HTTP 402 상태코드에서 영감받아 에이전트가 유료 API 엔드포인트 호출 시 블록체인 지갑으로 결제 서명을 생성하여 재전송. 에이전트-투-에이전트 가격 흥정을 프로토콜 수준에서 내장. 소액 결제 단위: USDC 기준 $10⁻⁶(약 0.001원)까지 지원. 이는 기존 13개 Claw 프레임워크 어디에도 없는 결제 자율성이다.

**Session 4 — 이상현: Serverless Agent** (→ R29 발굴)

AWS Lambda + TypeScript + MQTT 조합의 서버리스 에이전트:
```
LLM이 TypeScript 코드 생성
    ↓
tsc 타입 체크 (안전성 게이트)
    ↓
통과 시 Lambda에서 실행
(MQTT가 방화벽 뒤 로컬 기기 브리징)
```
별도 tool definition 파일이 필요 없다. 타입 정의 파일(`.d.ts`)이 tool schema를 대체한다. 비용: 테스트 기준 약 $1-2.

**Session 7 — 진주성: M3 Ultra SNS 크롤링**

3계층 로컬 AI 모델 전략:
- Layer 1: 로컬 소형 모델 → 빠른 판단, 라우팅
- Layer 2: 로컬 중간 모델 → 일반 작업 처리
- Layer 3: 클라우드 SOTA → 복잡한 추론만 위임

"전체 쿼리의 80%+를 로컬에서 처리"하는 목표는 OpenJarvis의 R10(Intelligence Per Watt) — 로컬 모델이 88.7% 단일 턴 쿼리 처리 가능이라는 연구 데이터 — 과 정확히 일치한다. 프레임워크 연구와 실무 구현이 수렴하는 지점이다.

**Session 10 — 허예찬: OMX/OMC/Claw 운영생태계** (→ R30 발굴)

대규모 에이전트 팀 운영의 핵심 패턴 두 가지:

1. **MEMORY.md 포인터맵**: MEMORY.md는 실제 내용이 아닌 파일 경로(포인터)만 저장. "MEMORY.md는 지도다. 실제 지형이 아니다." 에이전트는 bash/grep/ls로 필요한 정보를 직접 탐색. 벡터 DB 없이 10만 줄 이상 메모리 관리.

2. **agents.md 교리 엔진**: agents.md가 에이전트의 행동 원칙, 판단 기준, 협업 규약을 집약하는 "교리"로 작동. **OMC/OMX PR의 80%+ 이상이 에이전트 자신에 의해 생성**되는 자율 기여 생태계의 근거.

**Session 11 — 한수관: ResearchClaw**

결정론적 FSM(유한 상태 기계) 기반 에이전트 하네스:
```
플래닝 에이전트 (사람과 상호작용, 쓰기 권한: plan/ 디렉토리)
    ↓ 스펙 확정
실험 에이전트 (코드 작성 → 반드시 실행 강제, 쓰기 권한: experiments/)
    ↓ 실험 결과
평가 에이전트 (시각화, 쓰기 권한: results/ 디렉토리)
```
"각 에이전트에게 특정 디렉토리 내에서만 작업하도록 권한을 제한하는 것이 안정성의 핵심." 반도체 RTL 코드 자동 생성 검증 완료. 실제 FPGA + 로봇 환경 전환 시 레이턴시와 툴콜 해석 오류가 현재 과제.

**Session 16 — 정세민 (Sionic AI): Ultraworker**

Claude Code 기반 MCP 확장 시스템. Slack을 Human-in-the-Loop 인터페이스로 활용:
- 4단계 워크플로: 컨텍스트 탐색 → 슬랙 승인 → 테크 스펙 → 구현 완료
- 하이브리드 메모리: BM25 + Qdrant(벡터DB) + 그래프DB 3계층
- **5W1H 기반 룰베이스 온톨로지**로 맥락 랭킹 구성
- 실운용: 오래된 슬랙 채널 기록 스캔 → Rust 서비스 장애 원인 3가지 자동 파악

**메타 아티팩트: summary-prompt.md**

이 아카이브의 독특한 특징은 세션 요약 생성에 사용한 LLM 프롬프트 자체가 소스로 보존된다는 점이다. 이는 OpenClaw 생태계가 자체 지식 관리에도 에이전트를 활용하는 메타 패턴의 증거다. 동일 프롬프트를 다른 이벤트 트랜스크립트에 적용하면 동일 형식의 요약을 재현할 수 있다.

---

## 4. 교차 分析

이 섹션이 usecases/ 종합 보고서의 핵심이다. 3개 항목의 362개 데이터 포인트를 교차하면, 어느 단일 항목에서도 보이지 않던 패턴이 드러난다.

### 4.1 도메인 수요 분포 — 무엇을 만드는가

세 항목의 도메인 분포를 합산하면 OpenClaw 생태계의 실제 수요 지형이 드러난다:

| 도메인 | usecases (hesamsheikh) | agents (mergisi) | 밋업 (16세션) | 합산 순위 |
|--------|----------------------|-----------------|--------------|---------|
| **생산성/개인 자동화** | 18개 (45%) | personal 7 + productivity 7 = 14개 | 세션 05, 10, 15 | **1위 압도** |
| 개발/DevOps | 2개 | development 15 + devops 10 = 25개 | 세션 03, 04, 12 | 2위 |
| 비즈니스/마케팅 | 1개 | business 14 + marketing 19 = 33개 | — | 2위 (agents 기준) |
| 콘텐츠/크리에이티브 | 5개 | creative 10개 | 세션 07 | 3위 |
| 온체인/Web3 | 0개 (명시 배제) | moltbook 3개 | 세션 02, 06, 14 | 데이터 소스마다 상이 |
| 연구/지식 관리 | 6개 | — | 세션 11, 15 | 밋업에서 유독 강조 |
| 인프라/하드웨어 | 2개 | devops 10개 | 세션 03, 04, 07, 12 | 밋업 실무자층 특징 |
| 헬스케어/법률 | — | healthcare 7 + legal 6 = 13개 | — | agents에만 존재 |

**핵심 통찰**: 생산성/개인 자동화가 세 소스 모두에서 최상위다. 그런데 13개 repos/ 프레임워크 중 "개인 사용자 생산성 자동화"를 명시적 설계 목표로 내세운 것은 없다. 프레임워크들은 보안 격리, 채널 다중화, 확장성을 설계하는 동안, 커뮤니티는 일관되게 "나의 일상 자동화"를 원한다.

이 간극을 수치로 표현하면: repos/ 계층의 설계 우선순위 TOP 3가 보안(Tier 분류), 채널 다양성(NullClaw 19채널), 메모리 아키텍처라면, usecases/ 계층의 실수용 우선순위 TOP 3는 생산성(45%), 개발 자동화, 마케팅이다. 두 우선순위는 교집합이 있지만 완전히 다른 축에서 형성된다.

**awesome-openclaw-usecases가 명시적으로 배제한 크립토가 밋업에서는 세션 3개**를 차지한 것도 주목할 만하다. 동일한 커뮤니티 내에서 규범(Awesome List의 크립토 배제 정책)과 실무(밋업의 X402, ACP, ERC-8004 세션)가 분리되어 있다.

### 4.2 채널 선호도 격차 — 프레임워크 설계 vs 실사용

채널 선호도를 세 소스에서 교차 집계하면 뚜렷한 격차가 드러난다:

| 채널 | repos/ 프레임워크 지원 현황 | usecases/ 실수용 빈도 | 격차 판정 |
|------|--------------------------|---------------------|---------|
| **Telegram** | 13개 중 10+ 지원 | 8개+ use-case, 밋업 다수 | **정렬** — 사실상 표준으로 일치 |
| **Discord** | 10+ 지원 | 3개 use-case, 커뮤니티 조율 | 정렬 |
| **전화/SMS** | **NullClaw 1개 (Voice over IP)** | **4개 use-case** | **공백** — 수요 대비 지원 미흡 |
| **WhatsApp** | 3-4개 지원 | 2개 use-case (고객서비스) | 부분 정렬 |
| **Slack** | 다수 지원 | 3개 use-case, 밋업 세션 16 | 정렬 |
| **이메일** | 다수 지원 | 6개+ use-case | 정렬 |
| **Web UI** | 상대적으로 적음 | crewclaw.com으로 수요 해소 | 상용 플랫폼이 공백 메움 |
| **Signal/Matrix** | NullClaw만 지원 | 직접 언급 없음 | NullClaw의 선제적 지원 |

**전화/SMS 채널 공백**이 가장 큰 발견이다. awesome-openclaw-usecases에서 Phone-Based Personal Assistant, Event Guest Confirmation, Phone Call Notifications, Phone Call Notifications — 4개의 독립적 사례가 전화 기반 접근을 요구한다. 이는 개별 선호가 아니라 반복적 수요 패턴이다. 전화는 Telegram 알림보다 높은 긴급도를 전달하며, 손을 사용할 수 없는 상황(운전 중)에서도 접근 가능하다. 그러나 13개 프레임워크 중 이를 1등급 채널로 처리하는 것이 없다.

NullClaw의 19채널 전략은 실수용 다양성과 가장 정렬되어 있다. Telegram + Discord에 집중한 대부분의 프레임워크는 실제 수요 대비 채널 다양성이 과소하다.

### 4.3 배포 패턴 다양성 — 로컬/서버리스/분산

세 항목에서 확인된 배포 패턴을 유형화하면:

| 배포 유형 | 소스 | 대표 사례 | 특징 |
|---------|------|---------|------|
| **로컬 24/7 상주** | 밋업 세션 07, 15 | M3 Ultra 로컬 LLM, 중고PC 세컨드브레인 | 데이터 프라이버시 + 비용 절감 + OpenJarvis R10(Intelligence Per Watt) 구현 |
| **서버리스 클라우드** | 밋업 세션 04 | AWS Lambda + MQTT 브리지 | 무한 확장, 방화벽 우회, tsc 타입 검사 안전장치 |
| **홈랩 분산** | 밋업 세션 09 | FLOCK + Arbiter 워크플로우 | 자체 인프라 위 분산 에이전트 |
| **No-terminal 원클릭** | agents (crewclaw.com) | SOUL.md → $9 Dockerfile 생성 | 비개발자 진입장벽 제거 |
| **n8n 위임** | usecases | n8n 자격증명 격리 스택 | 보안 + 가시성 + LLM 비용 절감 동시 달성 |
| **다중디바이스 게이트웨이** | 밋업 세션 03 | OCI 무료 인스턴스 게이트웨이 | 여러 기기를 단일 에이전트로 통합 |
| **엔터프라이즈 분산** | 밋업 세션 16 | RTX 6000 6대, 30대 서버 | Human-in-the-Loop + BM25+벡터+그래프 3계층 |

**핵심 발견**: repos/ 계층의 공식 문서는 대체로 "로컬 실행"을 기본 시나리오로 제시한다. 그러나 usecases/ 계층에서 실무자들은 클라우드, 서버리스, 분산으로 활발하게 이동 중이다. 배포 패턴의 다양성이 프레임워크 문서의 가정을 이미 초과했다.

밋업 세션 05(정우석)의 비개발자 사례도 주목할 만하다. M백 PC, 맥북, 시놀로지 서버 3개 인스턴스를 동시 운용하는 비개발자가 스킬 개발 4단계 프로세스를 스스로 정립한 것은 OpenClaw의 접근성이 실무에서 검증됐다는 신호다. n8n + MCP 조합을 시도하다 "Claude Code 스킬 방식이 더 효율적"이라는 결론을 내린 것은 프레임워크 간 실무 비교 데이터로서 가치가 있다.

### 4.4 보안 인식 격차 — 커뮤니티 계층의 취약한 고리

repos/ 계층에는 Tier 1부터 Tier A+까지 세밀한 보안 분류 체계가 있다. 커뮤니티 콘텐츠 계층(usecases/)의 보안 상태는 이와 대조적이다:

| 항목 | 보안 현황 | 위험 수준 |
|------|---------|---------|
| **awesome-openclaw-usecases** | README 최상단 경고문만 있음 ("community-built skills not audited by maintainer") | 경고는 있으나 해결책 없음 |
| **awesome-openclaw-agents** | 177개 SOUL.md 중 **자동 검증 전혀 없음** | 악성 SOUL.md 삽입 가능. PR 리뷰 SLA 48시간 동안 노출 위험 |
| **밋업 세션 (전반)** | 보안에 대한 명시적 논의 거의 없음 (예외: 세션 04 tsc 안전장치) | 실무자들이 보안을 인지하나 체계화되지 않음 |

repos/ 계층의 Hermes Agent(R20 Skills Trust 4단계, R22 Tirith Pre-Exec Scanner)가 해결한 문제가 커뮤니티 콘텐츠 계층으로 전파되지 않고 있다. 구체적 위험 시나리오:

```
1. 공격자가 악성 지시가 담긴 SOUL.md를 PR로 제출
2. "Marketing Agent"처럼 보이지만 사용자 대화 내용을 외부 서버로 전송
3. 48시간 PR 리뷰 대기 중 다른 사용자가 복제·사용
4. 발견되더라도 이미 사용한 사람들에 대한 영향 추적 불가
```

awesome-openclaw-usecases의 경고문은 솔직하지만 행동 가능한 해결책을 제시하지 않는다. Hermes Agent의 R20 패턴(4단계 신뢰 정책: builtin/trusted/community/agent-created)이나 agents.json에 보안 스캔 통과 여부 필드를 추가하는 방향이 실질적 개선이 될 수 있다.

### 4.5 SOUL.md vs SKILL.md vs HAND.toml — 에이전트 신원 파일 표준 경쟁

세 항목의 분析에서 가장 중요한 교차 발견은 **에이전트 신원 파일 포맷의 표준 경쟁**이다. 동일한 기능("에이전트가 무엇을 어떻게 해야 하는지 정의")을 서로 다른 포맷으로 구현하는 세 접근이 공존한다:

| 파일 | 프레임워크/프로젝트 | 역할 | 포맷 | 보안 |
|------|------------------|------|------|------|
| **SOUL.md** | OpenClaw (awesome-openclaw-agents) | 신원, 성격, 행동 규칙, 예시 대화 | Markdown (섹션별 구조) | 없음 |
| **SKILL.md** | Hermes Agent, NanoClaw, 다수 | 스킬 메타데이터, 기능 확장 정의 | Markdown | agentskills.io 4단계 신뢰 (R20) |
| **HAND.toml** | OpenFang | 도구 선언, 권한, 에이전트 프롬프트, 생명주기 | TOML | Ed25519 서명, WASM 샌드박스 |
| **program.md** | Autoresearch | 무한 실험 루프 정의 | Markdown | Social contract만 (준수 강제 없음) |
| **AGENTS.md** | awesome-openclaw-usecases | 운영 규칙 (SOUL.md 보완) | Markdown | 없음 |
| **HEARTBEAT.md** | awesome-openclaw-agents | 주기 행동 체크리스트 | Markdown | 없음 |

```
현재 표준 경쟁 구도:

OpenClaw 생태계  →  SOUL.md + AGENTS.md + HEARTBEAT.md
(awesome-openclaw-agents 177개 참조 구현체가 사실상 표준화)

Hermes Agent     →  MEMORY.md + USER.md + SKILL.md
(agentskills.io 오픈 스탠다드 — R20)

OpenFang         →  HAND.toml
(Ed25519 서명 + WASM 샌드박스 — 보안 최강)

공통 없음: 세 진영이 각자의 포맷을 개발 중
```

**왜 이 경쟁이 중요한가**: 에이전트 신원 파일이 표준화되면 에이전트 이식성(portability)이 가능해진다. OpenClaw SOUL.md 에이전트를 Hermes Agent에서 실행하려면 현재 전면 재작성이 필요하다. 이 경쟁의 승자가 Claw 생태계의 "에이전트 컨테이너 포맷"이 된다. Docker가 컨테이너 이미지 포맷을 표준화한 것처럼.

현재는 awesome-openclaw-agents가 174개 참조 구현체로 사실상의 표준 라이브러리를 구축했다는 점에서 SOUL.md가 규모 면에서 앞선다. 그러나 보안과 이식성에서는 HAND.toml(Ed25519 서명)과 SKILL.md(agentskills.io 오픈 스탠다드)가 더 성숙하다.

### 4.6 멀티 디바이스 패턴 — OpenClaw의 핵심 설계 방향

최재훈의 발언("openclaw는 멀티 디바이스 에이전틱 자동화 프레임워크")은 개인 의견이 아닌 OpenClaw 공식 설계 문서로 뒷받침된다.

**공식 아키텍처 (Gateway-Node, R32)**:

```
Internet / Tailnet
        ↓
OCI Remote Gateway (에이전트 런타임, Tailscale-only edge)
        ↓ WebSocket
  ┌─────┬─────┬─────────────┐
  │     │     │             │
macOS Linux Windows/WSL  기타
Node  Node   Node
```

- Gateway: 단일 장기 프로세스. 채널(Telegram 등) + WebSocket 제어 플레인.
- Node: 각 기기의 OS별 Capabilities를 Gateway에 Advertisement.
- Tailscale: 퍼블릭 IP 불필요, VCN에서 UDP 41641만 개방.

**실사용 증거 3중 교차 확인**:

| 소스 | 내용 |
|------|------|
| OpenClaw 공식 문서 | `docs/gateway/network-model.md`, `raspberry-pi.md`, `oracle.md` |
| 밋업 세션 03 (최재훈) | OCI + Tailscale + 3기기 라이브 데모 |
| 페북 게시글 (최재훈) | 5기기 확장 구성 (macOS×2 + Linux + Windows/WSL×2) |
| awesome-openclaw-agents | `agents/devops/raspberry-pi/SOUL.md` Pi 전용 에이전트 템플릿 |
| awesome-openclaw-usecases | AionUi 원격 구조, Self-Healing Home Server |

**프레임워크 비교 관점**: 13개 Claw 프레임워크 중 멀티 디바이스를 1등급 설계로 문서화한 것은 OpenClaw뿐이다. NullClaw(19채널)가 채널 다양성, OpenJarvis(R10 Intelligence Per Watt)가 에너지 효율을 강조하는 것과 달리, OpenClaw는 **물리적 기기 플릿 관리**를 핵심으로 삼는다.

---

## 5. 기존 보고서와의 연결

### 5.1 repos/ 프레임워크와의 교차 검증

**NullClaw** (`reports/repos/details/nullclaw_report.md`): NullClaw의 19채널 전략이 usecases/ 계층의 실수용 다양성과 가장 정렬되어 있다. 전화/SMS 4건, Lark/DingTalk 1건, Signal/Matrix 언급 등 비주류 채널 수요가 13개 프레임워크 중 NullClaw에서만 일부 지원된다. NullClaw의 설계 결정 — "채널을 처음부터 19개로 시작" — 이 usecases/ 데이터로 사후 검증된다.

**OpenJarvis** (`reports/repos/details/openjarvis_report.md`): 밋업 세션 07(진주성)의 3계층 로컬 AI 모델 전략("로컬 소형 → 로컬 중간 → 클라우드 SOTA")이 R10(Intelligence Per Watt) — "로컬 모델이 88.7% 단일 턴 쿼리 처리 가능" — 과 정확히 대응한다. 프레임워크 연구(repos/)와 실무 구현(usecases/)이 독립적으로 같은 결론에 도달한 것은 패턴의 강건성을 의미한다.

**Hermes Agent** (`reports/repos/details/hermes_agent_report.md`): Hermes Agent의 R17(Frozen Snapshot Memory)은 세션 시작 시 MEMORY.md를 1회 캡처해 불변으로 유지한다. 허예찬(세션 10)의 R30(포인터맵 메모리 아키텍처)은 MEMORY.md가 포인터 지도 역할만 하고 에이전트가 bash/grep으로 탐색한다. 같은 "MEMORY.md" 파일이지만 역할이 정반대다:

| 비교 항목 | R17 Frozen Snapshot (Hermes) | R30 포인터맵 (OMC/OMX) |
|----------|------------------------------|----------------------|
| MEMORY.md 내용 | 실제 메모리 내용 (불변) | 파일 경로 포인터만 (지도) |
| 메모리 접근 방식 | 시스템 프롬프트 인컨텍스트 | bash/grep/ls 탐색 |
| 확장성 | 3,575 chars 상한 | 이론적 무제한 |
| Prefix cache 보존 | 핵심 목적 | 해당 없음 |
| 벡터 DB | 없음 | 없음 |

**Claude Code** (`reports/repos/details/claude_code_report.md`): 밋업 세션 05(정우석)에서 발표자가 "복잡한 명령은 'Claude Code를 진행하면 좋겠다'고 지시하면 자동으로 세팅"된다고 언급했다. OpenClaw와 Claude Code가 실무에서 보완적으로 사용된다는 1차 소스 데이터다. OpenClaw가 기획/오케스트레이션/검수를, Claude Code가 복잡한 구현을 담당하는 역할 분담이 자연스럽게 형성되고 있다.

**ResearchClaw (밋업 세션 11)**과 `reports/repos_research/research_tools_report.md`: 한수관의 FSM 기반 에이전트 하네스(플래닝/실험/평가 분리)는 DeepInnovator의 4계층 파이프라인(R4: 분析→그룹→인사이트→합성)과 구조적으로 유사하다. 두 접근 모두 "단일 에이전트에 모든 것을 맡기면 컨텍스트가 뒤섞인다"는 같은 문제를 명시적 단계 분리로 해결한다.

### 5.2 repos_applied/ 항목과의 중복/보완 관계

`reports/repos_applied/repos_applied_report.md`에서 분析한 5개 응용 프로젝트와 usecases/ 항목의 관계:

| repos_applied/ 항목 | usecases/ 교차점 |
|--------------------|----------------|
| **Moltbook** | awesome-openclaw-agents의 moltbook/ 카테고리 3개 에이전트가 Moltbook API를 SOUL.md에서 직접 참조. USE-CASES.md #38 "AI-to-AI Social Network" |
| **ClawWork** | USE-CASES.md #119 "AI Coworker Platform"과 컨셉 유사. 단, ClawWork는 경제 지표 측정에 특화 |
| **ClawPort** | 밋업 세션 16(정세민)의 Ultraworker가 Slack을 관제 인터페이스로 활용 — ClawPort의 대시보드 관제 철학과 방향 일치 |
| **Symphony** | USE-CASES.md #120 "Automated PR Triage"와 기능 겹침. Symphony가 PR 랜딩까지 자동화 |
| **MiClaw** | usecases/ 어디에도 모바일 OS 네이티브 에이전트 수요 없음 — MiClaw는 중국 시장 특화, 글로벌 커뮤니티 수요와 분리 |

**crewclaw.com의 repos_applied/ 자격 검토**: usecases_index.md에서 지적한 바와 같이, crewclaw.com은 repos_applied/ 등록 자격이 있다. Moltbook(소셜 플랫폼)과 유사하게, crewclaw.com은 SOUL.md 카탈로그(오픈소스)와 배포 패키지 생성 서비스(상용)의 이중 계층을 운영한다. 현재 repos_applied/에는 소스코드 접근 가능한 항목만 등록되어 있으나, crewclaw.com의 영향력($9 배포 패키지, 174개 에이전트 deploy 링크 전담)은 이미 분析 가치를 지닌다.

### 5.3 보안 보고서와의 연결 (Tier 분류 관점)

`reports/repos/security_report.md`의 Tier 분류 체계를 usecases/ 계층에 적용하면:

| 계층/항목 | 보안 Tier | 근거 |
|---------|---------|------|
| awesome-openclaw-usecases | **Tier 4 수준** | 보안 경고문만 있음, 기술적 검증 없음. n8n 패턴은 Zero-Exposure 유사(IronClaw 참조) |
| awesome-openclaw-agents (SOUL.md 생태계) | **Tier 4 이하** | 177개 템플릿 중 자동 보안 검증 전무. Hermes Agent R20/R22 상당의 보호 없음 |
| 밋업 세션 실무자 스택 | **Tier 2~3** | 이상현(tsc 타입 검사 게이트), 한수관(디렉토리 권한 제한) 등 일부 보안 의식 있음 |

이 계층 격차가 시사하는 바: Tier 1~A+ 프레임워크로 실행하더라도, 그 위에 로드되는 SOUL.md가 Tier 4 이하의 보안이라면 전체 보안은 가장 약한 고리로 제한된다. "에이전트 보안 = 프레임워크 보안"이 아니라 "에이전트 보안 = 프레임워크 × 스킬/템플릿 보안"이다.

```
실제 보안 등식:
전체 보안 = min(프레임워크 Tier, 스킬/SOUL.md Tier)

IronClaw (Tier 1) + 악성 SOUL.md (Tier 4이하) = 실제 Tier 4이하
```

### 5.4 메모리 아키텍처 보고서와의 연결 (R30 포인터맵)

`reports/repos/memory_architecture_report.md`는 13개 프레임워크의 메모리 아키텍처를 3단계 성숙도로 분류했다: Layer 0(SQLite/인컨텍스트 단순 저장), Layer 1(벡터 DB 하이브리드 검색), Layer 2(그래프 DB 관계형). 허예찬의 R30(포인터맵 메모리 아키텍처)은 이 분류체계 밖에 있다.

기존 분류와 비교:

```
기존 메모리 아키텍처 성숙도:

Layer 0: TinyClaw, PicoClaw — 인컨텍스트 직접 로드 (한계: 컨텍스트 윈도우)
Layer 1: OpenClaw, NanoClaw — DB 검색 후 삽입 (한계: 검색 품질)
Layer 2: IronClaw, OpenJarvis — 벡터+키워드 하이브리드 (한계: 인프라 비용)

R30 포인터맵: 별도 범주 — "에이전트의 도구(bash/grep)가 검색 엔진"
              벡터 DB 없이 10만 줄 이상 관리. 인프라 비용 없음.
              한계: bash/grep 도구 실행 능력에 의존, Windows 이식성 문제.
```

R30이 memory_architecture_report.md에 추가적 섹션으로 기록되어야 할 이유가 여기에 있다. 기존 분류체계가 벡터 DB 사용 여부를 축으로 삼는다면, R30은 "에이전트의 도구 사용 능력 자체를 메모리 검색 엔진으로 전용"하는 새로운 축을 제시한다.

---

## 6. 신규 패턴 (R29–R31) 상세

세 usecases/ 항목 분析에서 R1–R28에 없는 3개 패턴을 발굴했다.

---

### R29: TypeScript-as-Tool 동적 실행 패턴

**발굴 소스**: 이상현 (OpenClaw 서울 빌더 밋업, Session 04)
**구현**: AWS Lambda + tsc + MQTT

**원리**:

기존 에이전트 프레임워크에서 tool 사용은 두 단계로 구성된다: (1) 개발자가 tool definition(스키마)을 사전에 정의하고, (2) 에이전트가 그 정의에 따라 tool을 호출한다. 이 패턴은 단계 (1)을 제거한다.

```
기존 방식:
개발자 → tool definition 작성 → 에이전트가 호출

R29 패턴:
타입 정의 파일(.d.ts) → LLM이 TypeScript 코드 생성 → tsc 타입 검사 → 실행
(tool definition 없음)
```

구체적 구현:
1. API의 타입 정의 파일(`.d.ts`)만 에이전트 컨텍스트에 제공
2. LLM이 TypeScript로 도구 호출 코드를 직접 생성
3. tsc가 타입 오류를 정적 검증 (실행 전 안전성 게이트)
4. 검사 통과 시 Lambda에서 실행
5. MQTT 브리지가 방화벽 뒤 로컬 기기와 통신

**기존 R1~R28과의 차별성**:

| 비교 대상 | 차이점 |
|---------|--------|
| R20 Skills Trust (Hermes) | 사전 정의 스킬의 신뢰 등급 검증. R29는 스킬 정의 자체를 LLM이 생성 |
| R22 Tirith Pre-Exec Scanner (Hermes) | 바이너리 무결성(SHA-256). R29는 코드 타입 안전성 검증 — 다른 레이어 |
| R15 정적 컴파일 (NullClaw) | 사전 컴파일된 바이너리. R29는 에이전트 생성 코드의 실시간 타입 검증 |

**시사점**:
- tool schema 관리 비용 제거: 타입 정의 파일(.d.ts)만 있으면 어떤 TypeScript API도 즉시 도구화 가능
- 서버리스 환경에서 stateless하게 동작: 무한 확장성
- tsc 타입 검사 통과 = 실행 가능 보장 (런타임 타입 오류 사전 차단)
- 한계: LLM이 타입 안전하지만 의미론적으로 잘못된 코드를 생성할 수 있음 (타입 시스템은 로직 오류를 잡지 못함)

---

### R30: 포인터맵 메모리 아키텍처

**발굴 소스**: 허예찬 (OpenClaw 서울 빌더 밋업, Session 10, OMC/OMX 운영자)
**구현**: MEMORY.md(포인터맵) + 분산 콘텐츠 파일 + bash/grep/ls 탐색

**원리**:

> "MEMORY.md는 지도다. 실제 지형이 아니다."

```
기존 메모리 패턴:
MEMORY.md: [실제 내용 전체]
에이전트: 시스템 프롬프트에 MEMORY.md 전체 로드 → 인컨텍스트 검색

R30 포인터맵 패턴:
MEMORY.md: [파일 경로 포인터만] → "agent_decisions.md에 설계 결정 있음"
에이전트: bash/grep/ls로 필요한 파일만 선택적 탐색 → 결과만 컨텍스트에 로드
```

**기존 R17(Frozen Snapshot), R18(Char-Limited Memory)과의 비교**:

| 비교 항목 | R17 Frozen Snapshot | R18 Char-Limited | R30 포인터맵 |
|----------|---------------------|-----------------|------------|
| MEMORY.md 내용 | 실제 내용 (불변 스냅샷) | 실제 내용 (문자 예산 제한) | 파일 경로만 |
| 검색 방법 | 인컨텍스트 (LLM이 읽음) | 인컨텍스트 | bash/grep/ls (도구 실행) |
| 확장성 | ~3,575 chars | ~3,575 chars | 이론적 무제한 |
| Prefix cache | 보존 (핵심 목적) | 보존 | 해당 없음 |
| 벡터 DB | 없음 | 없음 | 없음 |
| 인프라 비용 | 없음 | 없음 | 없음 |

**시사점**:
- 벡터 DB 인프라 없이 100K+ 줄 메모리 관리 가능
- 에이전트의 도구 사용 능력(bash/grep)이 검색 능력 자체가 됨 — 에이전트 역량 향상이 메모리 검색 품질 향상으로 직결
- 메모리 구조가 파일시스템 구조와 동형(isomorphic) — 별도 추상화 레이어 불필요
- OMC/OMX 80%+ PR 에이전트 자동 생성에서 실증된 대규모 운영 패턴
- 한계: bash/grep 의존으로 Windows(PowerShell) 이식성 문제 존재. 에이전트의 도구 실행 능력에 메모리 검색 품질이 종속됨

---

### R31: Shared-State File Coordination

**발굴 소스**: awesome-openclaw-usecases / autonomous-project-management.md (STATE.yaml 패턴)
**구현**: STATE.yaml 공유 파일 + 각 에이전트의 직접 읽기/쓰기 + Git 버전 관리

**원리**:

```
기존 멀티에이전트 패턴 (R21 Bounded Delegation Tree 등):
메인 에이전트(오케스트레이터)
    → 서브에이전트 1 (스폰, 결과 수집)
    → 서브에이전트 2 (스폰, 결과 수집)
    → 서브에이전트 3 (스폰, 결과 수집)
(메인이 조율 병목, 깊이 제한 있음)

R31 Shared-State File Coordination:
메인 에이전트 (전략 결정만, "CEO 패턴")
    ↕ reads/writes
STATE.yaml ←→ pm-frontend (자율 실행)
            ←→ pm-backend (자율 실행)
            ←→ pm-content (blocked_by 감지 → 대기)
(파일이 조율 매체, 깊이 무제한, 병렬 실행)
```

**기존 R21(Bounded Delegation Tree)과의 차이**:

R21은 MAX_DEPTH=2, MAX_CONCURRENT=3의 **트리 구조**. 메인이 조율. R31은 깊이 제한 없는 **수평 자율 조율**. 파일이 조율 매체.

STATE.yaml 필드 의미:
```yaml
status: blocked        # 다른 에이전트가 감지하는 신호
blocked_by: api-auth   # 의존성 선언 (중앙 오케스트레이터 불필요)
next_actions:          # 자신이 끝난 후 다른 에이전트에게 남기는 메모
  - "pm-content: 이제 재개 가능"
```

**시사점**:
- 오케스트레이터 병목 없이 수십 개 에이전트가 병렬 실행 가능
- 파일 기반이므로 에이전트 충돌 후 재개 가능 (idempotent 재실행)
- Git 버전 관리로 전체 조율 이력 감사 가능
- Autoresearch의 results.tsv(R3, Git-as-state-machine)를 멀티에이전트 조율로 확장한 패턴
- 한계: 공유 파일에 대한 동시 쓰기 충돌 위험. 에이전트가 STATUS.yaml을 동시에 수정하면 레이스 컨디션 발생 가능

---

**신규 패턴 3개 비교 요약**

| R번호 | 패턴명 | 발굴 계층 | 해결 문제 | 핵심 기제 | 가장 유사한 기존 패턴 |
|-------|--------|---------|---------|---------|-----------------|
| R29 | TypeScript-as-Tool 동적 실행 | 밋업 (실무자 발표) | Tool definition 작성 부담 | tsc 타입 검사를 안전장치로 전용 | R20 (Skills Trust — 다른 레이어) |
| R30 | 포인터맵 메모리 아키텍처 | 밋업 (실무자 발표) | 대용량 메모리 관리 비용 | 파일시스템 도구를 검색 엔진으로 사용 | R17 (Frozen Snapshot — 반대 방향) |
| R31 | Shared-State File Coordination | Awesome List (사례 분析) | 오케스트레이터 병목 | 공유 파일이 조율 매체 | R21 (Bounded Delegation Tree — 반대 방향) |

세 패턴 모두 "기존 도구를 새로운 목적으로 전용"한다는 공통점이 있다. tsc를 안전장치로, bash/grep을 검색 엔진으로, YAML 파일을 조율 채널로. 이는 OpenClaw 생태계의 실무자들이 새 인프라를 구축하는 대신 기존 도구를 창의적으로 재활용하는 경향을 반영한다.

---

## 7. 결론 및 미해결 질문

### 핵심 결론 5가지

**결론 1: usecases/ 계층은 compare_claws 프로젝트에서 대체 불가능한 역할을 한다**

repos/ 분析이 "무엇이 만들어졌는가"를 답하고 repos_applied/ 분析이 "그 위에 무엇이 쌓였는가"를 답한다면, usecases/ 분析은 "실제로 어떻게 쓰이는가"를 답한다. R29, R30, R31 세 패턴은 소스코드 분析으로는 발굴할 수 없었다. 이는 커뮤니티 콘텐츠 분析이 독립적 방법론으로서 가치를 가짐을 입증한다.

**결론 2: 프레임워크 설계와 실수용 사이의 간극은 구조적이다**

13개 프레임워크 어느 것도 "개인 생산성 자동화"를 1등급 사용 사례로 설계하지 않았다. 그러나 usecases/ 계층의 모든 데이터 소스에서 개인 생산성이 1위다. 전화/SMS 채널 수요 4건이 있으나 프레임워크 지원은 사실상 없다. 이 간극은 개별 항목의 한계가 아니라 생태계 전체의 구조적 불일치다. 다음 세대 프레임워크 설계 시 반드시 참고해야 할 수요 신호다.

**결론 3: 커뮤니티 콘텐츠 계층의 보안은 가장 약한 고리다**

"전체 보안 = min(프레임워크 Tier, 스킬/SOUL.md Tier)"라는 등식이 성립한다. Tier 1 프레임워크 위에서 Tier 4 이하의 SOUL.md가 실행되면 실제 보안은 Tier 4 이하다. awesome-openclaw-agents의 177개 SOUL.md는 현재 어떠한 자동 보안 검증도 받지 않는다. Hermes Agent의 R20(4단계 신뢰 정책)과 R22(Tirith Pre-Exec Scanner)를 커뮤니티 콘텐츠 계층에 적용하는 것이 가장 시급한 생태계 개선 과제다.

**결론 4: 에이전트 신원 파일 표준 경쟁은 생태계의 이식성을 결정한다**

SOUL.md(OpenClaw, 174개 참조 구현체), SKILL.md(Hermes Agent, agentskills.io 표준), HAND.toml(OpenFang, Ed25519 서명)의 3파전이 진행 중이다. 현재는 규모에서 SOUL.md가 앞선다. 그러나 보안과 이식성에서 SKILL.md와 HAND.toml이 성숙하다. 표준이 수렴하지 않으면 에이전트 이식성은 불가능하며, Claw 생태계는 플랫폼별 에이전트 사일로가 된다.

**결론 5: 1차 소스 데이터(밋업, 커뮤니티)는 선행 지표다**

밋업에서 실무자들이 공유하는 우회 전략과 패턴은 나중에 프레임워크로 공식화된다. R29(TypeScript-as-Tool)와 R30(포인터맵 메모리)은 현재 개인 실험 수준이지만, 이 패턴들이 다음 OpenClaw 버전에 통합될 가능성이 있다. usecases/ 분析을 지속하면 프레임워크 설계의 미래 방향을 선행 감지할 수 있다.

---

### 미해결 질문 (Q39–Q43)

**Q39**: Telegram 편중이 심화되면 OpenClaw 생태계가 Telegram 플랫폼 정책 변경(봇 API 제한, 가격 인상 등)에 취약해지는가? Signal/Matrix 대안 수요가 충분한가? NullClaw가 이미 Signal/Matrix를 지원하지만 마이그레이션 비용이 커뮤니티 이동을 막는가?

**Q40**: crewclaw.com이 OpenClaw 공식 플랫폼인가 독립 서드파티인가? agents.json의 174개 deploy 링크를 전담하는 규모를 감안할 때, repos_applied/ 등록 자격을 갖춘 핵심 인프라인가? crewclaw.com 종료 시 awesome-openclaw-agents 생태계에 미치는 영향은?

**Q41**: 커뮤니티 큐레이션(hesamsheikh의 awesome-openclaw-usecases, mergisi의 awesome-openclaw-agents)과 공식 ClawHub 마켓플레이스의 품질·보안 기준 차이가 장기적으로 생태계 분열을 초래하는가? 아니면 공식-커뮤니티 이중 구조가 건강한 다양성을 창출하는가?

**Q42**: R29 TypeScript-as-Tool 패턴이 타입 정의 파일(.d.ts) 유지보수 부담을 도구 호출 스키마 정의 부담으로 단순히 치환하는가, 아니면 실질적으로 더 단순한가? tsc가 타입 안전성은 보장하지만 의미론적 오류(논리적으로 잘못된 코드)는 잡지 못하는 한계를 실무에서 어떻게 보완하는가?

**Q43**: R30 포인터맵 메모리 패턴이 bash/grep에 의존하므로 Windows 환경(PowerShell)에서 이식성 문제가 발생하는가? Windows 환경의 OpenClaw 사용자는 어떤 대안을 사용하는가? WSL이 해결책이 되는가, 아니면 PowerShell 버전의 R30 구현이 필요한가?

---

## 참고 문서

### usecases/ 상세 보고서
- `reports/usecases/details/awesome_openclaw_usecases_report.md` — hesamsheikh 어썸 리스트 상세 분析 (560줄)
- `reports/usecases/details/awesome_openclaw_agents_report.md` — mergisi 에이전트 템플릿 상세 분析 (498줄)
- `reports/usecases/details/openclaw_seoul_meetup_0315_report.md` — 서울 밋업 상세 분析 (401줄)
- `reports/usecases/usecases_index.md` — 계층 인덱스 및 교차 분析 요약

### 연관 보고서 (compare_claws)
- `reports/repos/security_report.md` — Tier 1~A+ 보안 분류 기준
- `reports/repos/memory_architecture_report.md` — 3단계 메모리 성숙도 분류 (R30 배경)
- `reports/repos/details/hermes_agent_report.md` — R17~R22, Skills Trust, Memory Injection Detection
- `reports/repos/details/nullclaw_report.md` — 19채널 지원, R15~R16
- `reports/repos/details/openjarvis_report.md` — R10~R14, Intelligence Per Watt
- `reports/repos/details/claude_code_report.md` — R23~R26, MCP-as-Channel
- `reports/repos_applied/repos_applied_report.md` — ClawWork, ClawPort, Symphony, Moltbook, MiClaw
- `reports/deployment/` — 서버리스·배포 전략 비교 (밋업 세션 04 배경)
- `reports/meetup/meetup_patterns_report.md` — 밋업 운영 패턴 10개 집약
- `reports/meetup/agent_payment_protocol_report.md` — X402, ACP, ERC-8004 결제 프로토콜

### 외부 소스
- https://github.com/hesamsheikh/awesome-openclaw-usecases
- https://github.com/mergisi/awesome-openclaw-agents
- https://crewclaw.com/agents
- https://agentskills.io (Hermes Agent 스킬 표준 — 비교 대상)

---

### MEMORY.md 업데이트 필요 항목

이 보고서로 인해 MEMORY.md에 반영해야 할 변경사항:

1. **R29~R31 신규 패턴 추가** — "New Patterns" 섹션에 3개 패턴 등록
2. **Q39~Q43 열린 질문 추가** — 현재 Q38까지 등록, Q39부터 추가
3. **usecases/ 계층 등록** — `repos_applied/`와 함께 4번째 분析 계층으로 명시
4. **보안 Tier 보완 노트** — "커뮤니티 콘텐츠 계층(SOUL.md 생태계)은 Tier 4 이하" 추가

---

*보고서 작성: 2026-03-21*
*분析 대상: 3개 항목 (awesome-openclaw-usecases 40개, awesome-openclaw-agents 174개+132개, OpenClaw 서울 밋업 16세션)*
*신규 패턴: R29 (TypeScript-as-Tool), R30 (포인터맵 메모리), R31 (Shared-State File Coordination)*
