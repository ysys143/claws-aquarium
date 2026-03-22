# awesome-openclaw-agents 상세 분석 보고서

> **소스**: GitHub mergisi/awesome-openclaw-agents
> **조사일**: 2026-03-21
> **유형**: 에이전트 템플릿 컬렉션 (Awesome List)

---

## 목차

1. [기본 정보](#1-기본-정보)
2. [핵심 특징](#2-핵심-특징)
3. [구조 분석](#3-구조-분석)
4. [콘텐츠 분석](#4-콘텐츠-분석)
5. [신규 패턴 (R-번호)](#5-신규-패턴-r-번호)
6. [비교 테이블](#6-비교-테이블)
7. [한계](#7-한계)
8. [참고 링크](#8-참고-링크)

---

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub URL** | https://github.com/mergisi/awesome-openclaw-agents |
| **Stars** | 미공개 (배지로만 표시) |
| **라이선스** | CC0 1.0 (퍼블릭 도메인) |
| **유형** | Awesome List -- 에이전트 템플릿 컬렉션 |
| **큐레이터** | mergisi |
| **템플릿 수** | 177개 (배지 기준) / 174개 (agents.json 기준) |
| **유스케이스 수** | 132개 (USE-CASES.md 기준) |
| **카테고리 수** | 24개 (agents/) / 22개 (USE-CASES.md) |
| **관련 플랫폼** | crewclaw.com (원클릭 배포), OpenClaw 공식 생태계 |
| **최종 갱신** | February 2026 (USE-CASES.md 기준) |
| **주요 언어** | Markdown (SOUL.md), JavaScript (quickstart) |

---

## 2. 핵심 특징

### 2.1 "177개 생산 준비 완료 에이전트 템플릿"

awesome-openclaw-agents는 단순한 코드 예제 모음이 아니다. 각 에이전트는 copy-paste만으로 OpenClaw 게이트웨이에 즉시 등록할 수 있는 **SOUL.md 파일** 하나로 정의된다. README 슬로건 "Skip the setup. Deploy in 60 seconds."가 이 컬렉션의 핵심 가치를 요약한다.

이 리포지토리가 중요한 이유는 세 가지다:

1. **생태계의 진입 장벽 제거**: 에이전트를 처음 만드는 사용자가 코드 없이 역할과 성격을 정의하고 배포할 수 있다. 비개발자도 "마케팅 에이전트 SOUL.md를 복사해서 이름만 바꾸면" Telegram에 에이전트가 산다.

2. **SOUL.md 표준의 사실상 표준 라이브러리**: OpenClaw 생태계에서 SOUL.md가 에이전트 설정 파일의 공식 포맷이라면, 이 컬렉션은 그 포맷의 참조 구현체 177개를 한 곳에 모은 것이다. Hermes Agent의 agentskills.io(R20)가 스킬 표준화라면, awesome-openclaw-agents는 에이전트 신원(identity) 표준화에 해당한다.

3. **agents.json을 통한 프로그래밍 접근**: 인간이 읽는 README와 별도로, 기계가 읽는 `agents.json`(total: 174)을 제공한다. 이는 이 컬렉션이 단순 문서를 넘어 에이전트 레지스트리(registry)로 기능하도록 설계되었음을 보여준다.

### 2.2 에이전트 신원 파일 체계 (Agent OS 개념)

CONTRIBUTING.md는 각 에이전트를 "프롬프트가 아니라 풀 운영 체제(full operating system)"로 정의한다. 이는 단순한 마케팅 문구가 아니라, 에이전트 디렉토리 구조에 실제로 반영되어 있다:

```
agents/[category]/[agent-name]/
├── SOUL.md       <- 신원 & 성격 (필수)
├── README.md     <- 설명 & 유스케이스 (필수)
├── AGENTS.md     <- 운영 규칙 (선택)
├── HEARTBEAT.md  <- 깨어날 때 실행할 체크리스트 (선택)
└── WORKING.md    <- 시작 상태 템플릿 (선택)
```

SOUL.md(신원) + AGENTS.md(규칙) + HEARTBEAT.md(주기 행동) + WORKING.md(현재 작업 상태)의 4파일 구조는 에이전트를 "항상 켜져 있는 프로세스"로 설계한다. Hermes Agent의 MEMORY.md + USER.md 이중 저장소(R17)가 세션 간 메모리 연속성에 초점을 맞춘 것과 달리, 이 구조는 에이전트의 **행동 규칙과 시작 상태의 분리**에 초점을 맞춘다.

### 2.3 생태계 위치

비교 관점에서 이 컬렉션은 OpenClaw 생태계에서 **에이전트 마켓플레이스에 가장 가까운 것**이다. 현재 OpenClaw 생태계에는 공식 마켓플레이스가 없고, crewclaw.com이 상용 배포 플랫폼 역할을 하며, awesome-openclaw-agents는 그 오픈소스 카탈로그 계층으로 기능한다.

---

## 3. 구조 분석

### 3.1 디렉토리 구조

```
awesome-openclaw-agents/
├── agents/                      # 177개 에이전트 템플릿 (24개 카테고리)
│   ├── automation/              # 6개 에이전트
│   ├── business/                # 14개 에이전트
│   ├── compliance/              # 4개 에이전트
│   ├── creative/                # 10개 에이전트
│   ├── customer-success/        # 2개 에이전트
│   ├── data/                    # 9개 에이전트
│   ├── development/             # 15개 에이전트
│   ├── devops/                  # 10개 에이전트
│   ├── ecommerce/               # 6개 에이전트
│   ├── education/               # 8개 에이전트
│   ├── finance/                 # 10개 에이전트
│   ├── freelance/               # 4개 에이전트
│   ├── healthcare/              # 7개 에이전트
│   ├── hr/                      # 7개 에이전트
│   ├── legal/                   # 6개 에이전트
│   ├── marketing/               # 19개 에이전트 (최다)
│   ├── moltbook/                # 3개 에이전트 (신규)
│   ├── personal/                # 7개 에이전트
│   ├── productivity/            # 7개 에이전트
│   ├── real-estate/             # 5개 에이전트
│   ├── saas/                    # 6개 에이전트
│   ├── security/                # 6개 에이전트
│   ├── supply-chain/            # 3개 에이전트 (신규)
│   └── voice/                   # 3개 에이전트 (신규)
├── quickstart/                  # 즉시 실행 가능한 Telegram 봇 스켈레톤
│   ├── bot.js                   # Telegram bot (SOUL.md 읽어 응답)
│   ├── SOUL.md                  # 기본 SOUL.md (교체 대상)
│   ├── docker-compose.yml       # Docker 실행 옵션
│   ├── Dockerfile
│   └── package.json
├── agents.json                  # 기계 가독형 레지스트리 (total: 174)
├── USE-CASES.md                 # 132개 실제 유스케이스 문서
├── CONTRIBUTING.md              # 에이전트 제출 가이드
└── README.md                    # 메인 카탈로그
```

### 3.2 agents.json -- 기계 가독형 레지스트리

`agents.json`의 구조는 다음과 같다:

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

`path` 필드가 SOUL.md 파일 경로를 직접 가리키는 구조는, 외부 도구(crewclaw.com 포함)가 이 인덱스를 파싱해 에이전트 목록을 자동 렌더링할 수 있도록 설계되었다. `deploy` 필드는 모든 에이전트에서 동일한 crewclaw.com URL을 가리킨다. 현재는 원클릭 배포 랜딩 페이지로 연결되며, 향후 에이전트별 배포 패키지 생성 API와 연동될 여지를 남긴다.

### 3.3 USE-CASES.md -- "무엇을 빌드하는가"의 분리

이 컬렉션의 주목할 구조적 결정은 **"어떻게 빌드하는가"(agents/)와 "무엇을 빌드하는가"(USE-CASES.md)를 분리**한 것이다.

- `agents/` 디렉토리: 에이전트 역할별 SOUL.md 템플릿 -- 공급(supply) 관점
- `USE-CASES.md`: 커뮤니티가 실제로 빌드한 사례 132개 -- 수요(demand) 관점

이 분리는 단순히 문서 구조의 문제가 아니다. 유스케이스에서 "4백만 게시물 데이터 파이프라인", "자율 로봇 조립" 같은 고급 사례는 현재 agent 템플릿에 대응하는 SOUL.md가 없다. 즉, USE-CASES.md는 커뮤니티의 실제 창의적 응용을 기록하며, 미래 템플릿의 로드맵으로도 기능한다.

### 3.4 quickstart/ -- 5분 스켈레톤

quickstart 디렉토리는 Node.js 18+와 Telegram Bot Token, API 키만 있으면 에이전트를 실행할 수 있는 최소 골격을 제공한다:

```bash
git clone https://github.com/mergisi/awesome-openclaw-agents.git
cd awesome-openclaw-agents/quickstart
cp .env.example .env          # TELEGRAM_BOT_TOKEN + ANTHROPIC_API_KEY
cp ../agents/marketing/echo/SOUL.md ./SOUL.md
npm install && node bot.js    # Telegram에서 에이전트 즉시 활성
```

`bot.js`는 SOUL.md를 읽어 LLM 시스템 프롬프트로 주입하는 최소 구현이다. Docker 지원도 포함되어 있어 서버 배포로의 전환이 간단하다. CrewClaw($9 일회성 비용)를 사용하면 Dockerfile + docker-compose + bot.js + README가 자동 생성되어 터미널 없이도 배포 가능하다.

quickstart에 포함된 파일:

| 파일 | 역할 |
|------|------|
| `bot.js` | SOUL.md를 읽어 Telegram 응답을 생성하는 최소 봇 |
| `.env.example` | 환경 변수 템플릿 (API 키, 봇 토큰) |
| `package.json` | Node.js 의존성 |
| `SOUL.md` | 교체 대상 기본 SOUL.md |
| `docker-compose.yml` | Docker 실행 옵션 |

### 3.5 crewclaw.com 통합 패턴

crewclaw.com은 이 컬렉션의 상용 배포 계층이다:

| 경로 | 기능 |
|------|------|
| crewclaw.com/agents | 177개 템플릿 브라우저 (GitHub 카탈로그의 UI 레이어) |
| crewclaw.com/create-agent | 원클릭 배포 패키지 생성 (Dockerfile 포함, $9 일회성) |
| crewclaw.com/blog/ | 튜토리얼 & 비교 가이드 (OpenClaw vs LangChain 등) |

이 구조는 오픈소스 카탈로그(GitHub)와 상용 배포 플랫폼(crewclaw.com)이 공생하는 이중 채널 모델이다. 카탈로그는 CC0 퍼블릭 도메인으로 완전 공개하면서, 배포 편의성은 상용 서비스로 수익화한다.

---

## 4. 콘텐츠 분석

### 4.1 카테고리별 에이전트 수 (agents.json 실측, total=174)

| 순위 | 카테고리 | 에이전트 수 | 비율 | 비고 |
|------|----------|------------|------|------|
| 1 | marketing | 19 | 10.9% | 최다 카테고리 |
| 2 | development | 15 | 8.6% | |
| 3 | business | 14 | 8.0% | |
| 4 | creative | 10 | 5.7% | |
| 4 | devops | 10 | 5.7% | |
| 4 | finance | 10 | 5.7% | |
| 7 | data | 9 | 5.2% | |
| 8 | education | 8 | 4.6% | |
| 9 | healthcare | 7 | 4.0% | |
| 9 | hr | 7 | 4.0% | |
| 9 | personal | 7 | 4.0% | |
| 9 | productivity | 7 | 4.0% | |
| 13 | ecommerce | 6 | 3.4% | |
| 13 | legal | 6 | 3.4% | |
| 13 | saas | 6 | 3.4% | |
| 13 | security | 6 | 3.4% | |
| 13 | automation | 6 | 3.4% | 신규 |
| 18 | real-estate | 5 | 2.9% | |
| 19 | compliance | 4 | 2.3% | 신규 |
| 19 | freelance | 4 | 2.3% | |
| 21 | moltbook | 3 | 1.7% | 신규 |
| 21 | supply-chain | 3 | 1.7% | 신규 |
| 21 | voice | 3 | 1.7% | 신규 |
| 24 | customer-success | 2 | 1.1% | 신규 |
| **합계** | | **174** | **100%** | |

**배지(177)와 agents.json(174)의 불일치**: 3개 에이전트가 agents.json에 등록되지 않았거나, 배지가 구버전 카운트를 반영한다. README 본문 표는 marketing 카테고리에서 17개만 나열하나 agents.json에는 19개가 있어, 일부 에이전트가 README 업데이트 없이 JSON에만 추가된 것으로 보인다.

### 4.2 카테고리 구성의 의미

**marketing(19개)이 1위**인 것은 OpenClaw 커뮤니티의 주요 사용자층이 소규모 사업자, 프리랜서, 콘텐츠 크리에이터임을 반영한다. "SEO 에이전트 + 콜드 아웃리치로 월 60건 콜 예약"(USE-CASES.md #39) 같은 사례가 이를 뒷받침한다.

**신규 카테고리 6개**(moltbook, supply-chain, voice, automation, customer-success, compliance)는 생태계 확장 방향을 보여준다. 특히:

- **moltbook**: 에이전트-투-에이전트 소셜 레이어 -- repos_applied/에서 분석한 Moltbook 플랫폼과 직접 연동
- **voice**: 전화 수신/발신 에이전트 -- Claude Code의 채널 개념(R23)을 음성으로 확장
- **automation**: "자는 동안 500개 이력서 지원", "자는 동안 코딩" 등 야간 자율 실행 패턴
- **compliance**: GDPR, SOC2, EU AI Act 등 규제 준수 자동화 -- 기업 고객 대상 확장

### 4.3 SOUL.md 템플릿 구조 분석

Orion(productivity/orion) SOUL.md를 예시로 SOUL.md의 실제 구조를 살펴본다:

```markdown
# Orion - The Coordinator

You are Orion, an AI coordinator and project manager powered by OpenClaw.

## Core Identity
- **Role:** Task coordinator and project orchestrator
- **Personality:** Professional, efficient, proactive
- **Communication:** Clear, structured, action-oriented

## Responsibilities
1. **Task Management**
   - Break down complex projects into actionable tasks
   - Prioritize work based on urgency and impact

2. **Delegation**
   - Identify the right agent for each task
   - Coordinate multi-agent workflows

## Behavioral Guidelines
### Do:
- Always provide next steps after completing a task
### Don't:
- Make assumptions about priorities without asking

## Communication Style
- Use bullet points for clarity
- End with clear action items or next steps

## Example Interactions
**User:** I need to launch a product next week
**Orion:** [Product Launch Plan with task breakdown and delegations]

## Integration Notes
- Works best with Echo (content) and Radar (analysis)
- Can coordinate up to 5 parallel tasks
- Supports calendar integration via MCP
```

SOUL.md의 구조적 특징:

| 섹션 | LLM 관점 | 역할 |
|------|---------|------|
| Core Identity | 페르소나 선언 | 에이전트가 누구인지 1인칭으로 정의 |
| Responsibilities | 작업 범위 | LLM이 처리해야 할 도메인 명시 |
| Behavioral Guidelines | Do/Don't 제약 | 명시적 행동 경계 설정 |
| Example Interactions | Few-shot 예제 | 응답 형식과 품질 기준 제시 |
| Integration Notes | 협업 컨텍스트 | 다른 에이전트와의 관계 명시 |

이 구조는 LLM 시스템 프롬프트를 인간이 편집하기 쉬운 Markdown 섹션으로 분절한 것이다. "에이전트 설정 = 시스템 프롬프트"라는 OpenClaw 설계 철학의 직접적 구현이다.

### 4.4 Moltbook SOUL.md -- 에이전트-투-에이전트 계층

Moltbook Community Manager의 SOUL.md는 특별히 주목할 만하다:

```markdown
## Rules
- Never spam — maximum 1 post per 30 minutes (Moltbook rate limit)
- Always disclose you are an AI agent (Moltbook requires this)
- Track engagement metrics but do not optimize purely for karma
```

이 규칙들은 에이전트가 다른 에이전트(또는 사람)와 소통하는 소셜 레이어를 위해 특별히 설계된 제약이다. "AI 에이전트임을 항상 공개"는 Claude Code의 Content/Meta 채널 분리(R25)와 유사한 투명성 원칙이다. Moltbook이 AI 에이전트 소셜 네트워크라는 점에서, 이 카테고리는 "에이전트 생태계의 소셜 레이어"를 실험하는 전초지다.

### 4.5 USE-CASES.md 하이라이트: Meta Use Cases 클러스터

USE-CASES.md의 22개 카테고리 중 가장 이론적으로 흥미로운 것은 **"Meta Use Cases (Agent Operating on Itself)"** (4개)다:

| # | 사례 | 자기참조 메커니즘 |
|---|------|-----------------|
| 129 | Bot Writes Its Own Marketing | 에이전트가 자신의 유스케이스 레포를 찾아 마케팅 페이지로 변환 후 배포 |
| 130 | Self-Updating Skills | 에이전트가 자신의 스킬과 설정을 직접 업데이트 |
| 131 | Agent-to-Human Delegation | 에이전트가 작업을 인간에게 위임하고 비동기로 모니터링 |
| 132 | Physical Body Self-Modification | 로봇 프로토타입이 자신의 코드를 편집해 360도 회전을 스스로 학습 |

사례 #129는 특히 자기참조적이다: "OpenClaw 에이전트에게 OpenClaw 유스케이스를 찾으라고 했더니, 레포를 발견하고 마케팅 페이지로 변환해 배포했다." 이는 바로 이 awesome-openclaw-agents 리포지토리 자체가 에이전트에 의해 발견된 사례일 수 있다는 가능성을 암시한다.

USE-CASES.md 전체 분포:

| 카테고리 | 수 |
|---------|---|
| Personal Productivity | 14 |
| Business Operations | 11 |
| Developer Workflows | 10 |
| Content Creation & Social Media | 10 |
| Ecosystem Tools Built on OpenClaw | 10 |
| Finance & Trading | 7 |
| DevOps & SysAdmin | 7 |
| Creative, Gaming & Culture | 8 |
| Communication & Integration | 8 |
| Email & Inbox Management | 5 |
| Shopping & E-Commerce | 5 |
| Smart Home & IoT | 4 |
| Travel & Transportation | 4 |
| Robotics & Hardware | 4 |
| Meta Use Cases | 4 |
| Calendar & Scheduling | 3 |
| Health & Fitness | 3 |
| Architecture/Real Estate/Legal | 3 |
| Family & Parenting | 3 |
| Wearables & Mobile | 3 |
| Decentralized & Crypto-Native | 3 |
| Research & Knowledge | 3 |
| **합계** | **132** |

### 4.6 compare_claws repos_applied/ 교차 참조

compare_claws의 repos_applied/ 항목과 이 컬렉션의 연관성:

| repos_applied/ 항목 | awesome-openclaw-agents 연관 |
|---------------------|------------------------------|
| **Moltbook** | moltbook/ 카테고리 3개 에이전트 직접 연동 |
| **MiClaw** | 언급 없음 (모바일 OS 레이어 -- 채널 구조 다름) |
| **Symphony** | 언급 없음 |
| **ClawWork** | USE-CASES.md #119 "AI Coworker Platform"이 유사 컨셉 |
| **ClawPort** | USE-CASES.md #123 "Fleet Management"와 유사 |
| **autoresearch-skill** | USE-CASES.md #118 "Industry Research Pipeline"과 방향 일치 |

Moltbook이 가장 직접적인 연동이다. moltbook/ 카테고리 에이전트들은 Moltbook 플랫폼 API를 SOUL.md 설정으로 직접 참조한다.

---

## 5. 신규 패턴 (R-번호)

### R29 후보 평가: SOUL.md as Universal Agent Identity Standard

**정의**: 에이전트의 역할, 성격, 행동 규칙, 예시 대화를 단일 Markdown 파일(SOUL.md)로 캡슐화하는 에이전트 신원 표준. 코드 없이 SOUL.md만 교체해 에이전트를 재정의한다.

**기존 패턴과의 비교**:

| 패턴 | 파일 | 목적 | 프레임워크 |
|------|------|------|-----------|
| R17: Frozen Snapshot Memory | MEMORY.md | 세션 간 메모리 연속성 | Hermes Agent |
| R20: Skills Trust Levels | skills_guard.py | 스킬 보안 검증 | Hermes Agent |
| HAND.toml | HAND.toml | 도구/권한/에이전트 프롬프트 | OpenFang |
| program.md | program.md | 무한 실험 루프 정의 | Autoresearch |
| **SOUL.md** | **SOUL.md** | **에이전트 신원 & 행동 규칙** | **OpenClaw** |

**R29 부여 여부 판정**: 보류 (Not Novel Enough)

SOUL.md는 실질적으로 "시스템 프롬프트를 Markdown 파일로 관리"하는 패턴이다. HAND.toml(OpenFang), program.md(Autoresearch) 모두 유사한 "설정 파일이 에이전트 행동을 정의"하는 개념을 구현한다. SOUL.md의 차별점은 **Markdown 섹션화**와 **copy-paste 접근성**에 있으나, 이는 포맷 선택이지 아키텍처 혁신은 아니다. 새로운 R-번호를 부여하기에는 기존 패턴과 차이가 충분하지 않다.

### R29 후보 평가: Meta Use Cases -- 자기참조 에이전트 루프

**정의**: 에이전트가 자신의 코드, 설정, 스킬, 마케팅 자료를 직접 수정/생성하는 자기참조 루프. 에이전트의 작업 대상이 자기 자신이 되는 패턴.

**기존 패턴과의 비교**:

| 기존 패턴 | 자기참조 여부 | 메커니즘 |
|-----------|--------------|---------|
| R11: Trace->LoRA (OpenJarvis) | 부분적 -- 자신의 가중치 개선 | LoRA 파인튜닝 (외부 훈련 루프) |
| R13: AgentConfigEvolver (OpenJarvis) | 부분적 -- 자신의 설정 진화 | TOML config 자동 수정 |
| R3: Fixed-Budget Loop (Autoresearch) | 부분적 -- 자신의 코드 수정 | train.py 자율 편집 |
| **Meta Use Cases** | **완전** -- 에이전트가 자신을 대상으로 임의 작업 | **범용 자기참조** |

**R29 부여 여부 판정**: 보류 (증거 불충분)

Meta Use Cases의 4개 사례는 흥미롭지만, 이것이 아키텍처 패턴인지 창의적 응용 사례인지 불명확하다. "에이전트가 자신의 코드를 편집"(#132)은 R11/R13과 메커니즘이 겹치며, "에이전트가 자신의 마케팅을 작성"(#129)은 일반 에이전트 기능의 자기적용이다. 체계적 구현(API, 설정, 안전장치)이 확인되지 않는 상태에서 R-번호 부여는 시기상조다.

**결론**: 이 컬렉션에서 R28 이후의 신규 패턴은 발견되지 않는다. SOUL.md 포맷과 Meta Use Cases 모두 기존 패턴의 조합 또는 응용으로 설명된다.

---

## 6. 비교 테이블

### 6.1 에이전트 카탈로그/마켓플레이스 비교

| 항목 | awesome-openclaw-agents | agentskills.io (Hermes R20) | crewclaw.com (상용) | OpenFang Hands |
|------|------------------------|----------------------------|---------------------|----------------|
| **콘텐츠 유형** | SOUL.md 템플릿 (신원 정의) | skill.py 스킬 (기능 확장) | 배포 패키지 생성 | HAND.toml 기능 패키지 |
| **코드 포함** | [X] (Markdown only) | [O] (Python) | [O] (Dockerfile 등) | [O] (Rust/WASM) |
| **배포 통합** | crewclaw.com 링크 | agentskills.io Hub | 자체 원클릭 배포 | openclaw agents add |
| **보안 검증** | [X] | 4단계 신뢰 정책 + 정적 분석 | [X] (명시) | WASM 샌드박스 |
| **표준화 수준** | 높음 (SOUL.md 포맷 고정) | 높음 (API 표준 정의) | 상용 (비공개) | 중간 (HAND.toml) |
| **큐레이터 신뢰 모델** | 단일 큐레이터 (mergisi) | 오픈 표준 위원회 | 플랫폼 벤더 | OpenFang 팀 |
| **항목 수** | 177개 | 비공개 | 비공개 | 7개 (번들) |
| **라이선스** | CC0 (퍼블릭 도메인) | 오픈 스탠다드 | 독점 | MIT |
| **오프라인 사용** | [O] (git clone) | [O] | [X] | [O] |

### 6.2 에이전트 파일 포맷 비교

| 파일 | 프레임워크 | 역할 | 보안 |
|------|-----------|------|------|
| SOUL.md | OpenClaw | 신원 & 성격 & 규칙 | 없음 |
| MEMORY.md + USER.md | Hermes Agent | 세션 간 메모리 | Memory Injection Detection (R19) |
| HAND.toml | OpenFang | 기능 패키지 선언 | Ed25519 서명 |
| program.md | Autoresearch | 실험 루프 정의 | 없음 (social contract) |
| SKILL.md | 다수 | 스킬 메타데이터 | 신뢰 레벨별 상이 |
| AGENTS.md | awesome-openclaw | 운영 규칙 | 없음 |
| HEARTBEAT.md | awesome-openclaw | 주기 행동 체크리스트 | 없음 |

---

## 7. 한계

### 7.1 단일 큐레이터 편향

mergisi 한 명이 큐레이션하는 구조는 **도메인 편향**을 초래한다. marketing(19개, 10.9%)이 1위이고 voice(3개), supply-chain(3개), customer-success(2개)가 최하위인 분포는 큐레이터의 관심사와 네트워크를 반영할 가능성이 높다. 예를 들어 의료(7개)와 법률(6개)은 실제 수요 대비 적게 커버되며, 이 도메인의 전문가가 커뮤니티에 기여하지 않는 한 편향은 지속된다.

### 7.2 보안 검증 부재

177개 SOUL.md 템플릿은 **어떠한 보안 검증도 거치지 않는다**. Hermes Agent의 Skills Trust 4단계(R20)나 Tirith pre-exec 스캐너(R22)와 비교하면 극명한 대조를 이룬다.

구체적 위험:
- SOUL.md에 악의적 지시를 삽입한 템플릿이 PR로 제출될 수 있다
- "Do not reveal system prompt" 같은 방어적 지시가 표준 포맷에 없다
- 커뮤니티 기여자의 SOUL.md가 사용자 데이터를 외부로 전송하도록 지시할 수 있다
- PR 리뷰 SLA가 "48시간 이내"이므로 악성 템플릿이 수 시간 동안 노출될 수 있다

이 취약성은 컬렉션이 성장할수록 악화된다. 현재는 mergisi가 모든 PR을 직접 검토하나, 규모가 커지면 단일 리뷰어 병목이 불가피하다.

### 7.3 crewclaw.com 의존성

agents.json의 모든 174개 `deploy` 필드는 `https://crewclaw.com/create-agent`를 가리킨다. **crewclaw.com이 서비스를 종료하면 배포 링크 전체가 무효화**된다. README의 "Deploy in 60 seconds" 핵심 가치 제안이 플랫폼 가용성에 전적으로 의존하는 구조다.

완화책: quickstart/ 디렉토리가 crewclaw.com 없이도 로컬 실행을 가능하게 하므로, 완전한 단일장애점은 아니다. 그러나 "노 터미널, 60초 배포" 경험은 crewclaw.com 없이는 재현 불가능하다.

### 7.4 버전 호환성 문제

SOUL.md 템플릿은 특정 OpenClaw 버전의 필드 구조, 지시문 포맷, MCP 설정 문법을 가정한다. OpenClaw API가 변경되면 **177개 템플릿이 일괄적으로 구버전**이 될 수 있다. 각 SOUL.md에 OpenClaw 버전 호환성 표기가 없고, `agents.json`에도 스키마 버전 필드가 없다.

현재 README의 "last updated: February 2026" 표기가 이 문제를 부분적으로 완화하나, 세부 템플릿별 갱신 추적은 불가능하다.

### 7.5 agents.json 정수 불일치 (177 vs 174)

배지(177)와 agents.json total(174)의 3개 불일치는 인덱스 유지보수 부담을 보여준다. README 본문의 에이전트 표와 agents.json의 항목 수도 일부 불일치한다. 단일 큐레이터 모델에서 두 파일을 동기화 상태로 유지하는 것은 컬렉션이 커질수록 어려워진다.

### 7.6 실제 테스트 미검증

CONTRIBUTING.md의 PR 체크리스트에 "Agent tested (works with OpenClaw or similar framework)"가 포함되어 있으나, 이는 **권고사항이지 강제 검증이 아니다**. 177개 템플릿 중 실제로 OpenClaw에서 실행 테스트를 거친 비율은 불명확하며, CI/CD 자동 검증 파이프라인은 확인되지 않는다.

---

## 8. 참고 링크

### 소스
- [GitHub: mergisi/awesome-openclaw-agents](https://github.com/mergisi/awesome-openclaw-agents)
- [crewclaw.com/agents](https://crewclaw.com/agents) -- 상용 배포 브라우저
- [USE-CASES.md](https://github.com/mergisi/awesome-openclaw-agents/blob/main/USE-CASES.md) -- 132개 실제 유스케이스
- [agents.json](https://github.com/mergisi/awesome-openclaw-agents/blob/main/agents.json) -- 기계 가독형 레지스트리

### compare_claws 내부 참조
- `/Users/jaesolshin/Documents/GitHub/compare_claws/reports/repos/details/hermes_agent_report.md` -- Skills Trust 4단계(R20), Memory Injection Detection(R19) 비교
- `/Users/jaesolshin/Documents/GitHub/compare_claws/reports/repos_applied/repos_applied_report.md` -- Moltbook, ClawWork, ClawPort 상용화 분석
- `/Users/jaesolshin/Documents/GitHub/compare_claws/reports/repos/details/claude_code_report.md` -- MCP-as-Channel(R23), Content/Meta 분리(R25) 비교
- `/Users/jaesolshin/Documents/GitHub/compare_claws/reports/repos/security_report.md` -- 보안 Tier 분류 기준

### 관련 외부 자료
- [crewclaw.com/blog/soul-md-examples-templates](https://crewclaw.com/blog/soul-md-examples-templates) -- SOUL.md 공식 가이드
- [crewclaw.com/blog/openclaw-cli-commands-reference](https://crewclaw.com/blog/openclaw-cli-commands-reference) -- OpenClaw CLI 레퍼런스
- [agentskills.io](https://agentskills.io) -- Hermes Agent 스킬 표준 (비교 대상)

---

## 미해결 질문

**Q33**: SOUL.md 포맷이 OpenClaw 공식 스펙인가, 아니면 mergisi가 독자적으로 정의한 컨벤션인가? 공식 OpenClaw 문서에서 SOUL.md를 의무 필드로 명시하는지 확인 필요.

**Q34**: crewclaw.com의 배포 패키지 생성이 SOUL.md를 어떻게 변환하는가? 단순히 Dockerfile에 삽입하는가, 아니면 별도 파싱/검증 단계가 있는가?

**Q35**: Moltbook 카테고리 3개 에이전트가 실제 Moltbook API와 어떻게 연동되는가? SOUL.md에 API 설정이 포함되어 있으나 실제 MCP 서버 구현이 별도로 필요한지 불명확.

**Q36**: Meta Use Cases(#129-#132)에서 "에이전트가 자신의 SOUL.md를 수정"한 사례가 있는가? 이것이 가능하다면 자기참조 루프가 실제로 구현된 최초 사례가 될 수 있다.
