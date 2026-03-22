# OpenClaw 서울 빌더 밋업 2026-03-15 상세 분석 보고서

> **소스**: `usecases/openclaw_seoul_meetup_0315/` (로컬)
> **조사일**: 2026-03-21 (R32 추가: 2026-03-21)
> **유형**: 커뮤니티 밋업 — 실무자 세션 모음 (1차 소스 정성 데이터)

---

## 목차

1. [기본 정보](#1-기본-정보)
2. [핵심 특징](#2-핵심-특징)
3. [구조 분석](#3-구조-분석)
4. [콘텐츠 분析](#4-콘텐츠-분석)
5. [신규 패턴 (R-번호)](#5-신규-패턴-r-번호)
6. [비교 테이블](#6-비교-테이블)
7. [한계](#7-한계)
8. [참고 링크](#8-참고-링크)

---

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **행사명** | OpenClaw 서울 빌더 밋업 2026-03-15 |
| **일시** | 2026년 3월 15일 |
| **주최** | Instruct.KR |
| **수용 인원** | 50-60명 |
| **RSVP** | 950명 이상 (수용 인원 대비 약 16-19배) |
| **의의** | 첫 번째 OpenClaw 서울 공식 행사 + ClawCon Seoul 공식 확정 발표 |
| **소스 경로** | `usecases/openclaw_seoul_meetup_0315/` |
| **세션 수** | 16개 (오프닝 포함) |
| **주요 언어** | 한국어 (일부 영어 발표 포함) |
| **파일 구성** | README.md, 전체 트랜스크립트 .txt, summary-prompt.md, sessions/ (16개 .txt), summary/ (16개 .md), openclaw-meetup.png |
| **글로벌 발표자** | Lionel Sim (싱가포르), Zoe Chen (Unibase, 영어권), Logan Kang (Base Korea) |

---

## 2. 핵심 특징

### 왜 이 이벤트가 분석 가치를 갖는가

이 밋업은 단순한 기술 발표 행사가 아니다. **OpenClaw 생태계가 실제로 어떻게 사용되고 있는지에 대한 1차 소스 정성 데이터**를 제공하는 희귀한 자료다. 학술 논문이나 공식 문서와 달리, 실무 빌더들이 실패 경험과 우회 전략을 포함해 솔직하게 공유하는 내용이 담겨 있다.

**950+ RSVP / 50-60명 수용**이라는 비율(16-19배 초과 신청)은 단순한 숫자가 아니다. 한국 AI 에이전트 개발자 커뮤니티의 규모와 관심 밀도를 정량적으로 보여주는 지표다. 이 비율은 2025년 초 OpenClaw 생태계 성장이 한국에서 가속화되고 있음을 시사한다.

**16개 세션의 도메인 다양성**도 주목할 만하다. 인프라(서버리스, 경량 배포), 온체인 결제(Base, Virtuals ACP), 연구 자동화(반도체, 로보틱스), 하드웨어(M3 Ultra, 중고PC), 콘텐츠 제작, 기업 운영까지 하나의 에이전트 런타임 플랫폼이 이 모든 도메인에서 실제 운용 사례를 확보했다는 사실은 생태계 성숙도를 반영한다.

**ClawCon Seoul 공식 확정** 발표는 한국이 아시아 AI 에이전트 허브로서 역할을 수행하기 시작했다는 상징적 신호다. Lionel Sim(싱가포르)의 참여도 동남아시아 커뮤니티와의 연결을 보여준다.

---

## 3. 구조 분析

### 디렉토리 구조

```
usecases/openclaw_seoul_meetup_0315/
├── README.md                           # 행사 개요 + 16개 세션 테이블
├── OpenClaw_Seoul_Builder_Meetup_2026._03.txt  # 전체 행사 트랜스크립트 (원본)
├── summary-prompt.md                   # LLM 요약 생성 프롬프트 (메타 아티팩트)
├── openclaw-meetup.png                 # 행사 사진
├── sessions/                           # 16개 원본 .txt 트랜스크립트
│   ├── 01_운영진_오프닝.txt
│   ├── 02_Logan_Kang_Composable_Agent_Economy.txt
│   ├── 03_최재훈_Remote_OpenClaw.txt
│   ├── 04_이상현_Serverless_Agent.txt
│   ├── 05_정우석_OpenClaw_스킬_효율적으로_붙이기.txt
│   ├── 06_김서준_AI_에이전트_온체인_경제.txt
│   ├── 07_진주성_M3_Ultra_SNS_크롤링.txt
│   ├── 08_민용기_에이전트_엔진룸_툴콜링.txt
│   ├── 09_윤주운_Arbiter_FLOCK_워크플로우.txt
│   ├── 10_허예찬_OMX_OMC_Claw운영생태계.txt
│   ├── 11_한수관_ResearchClaw.txt
│   ├── 12_김동규_nanoclaw_사용기.txt
│   ├── 13_Lionel_Sim_OpenClaw_Singapore.txt
│   ├── 14_Zoe_Chen_Unibase_Extending_OpenClaw.txt
│   ├── 15_김우현_중고PC_세컨드브레인.txt
│   └── 16_정세민_Ultraworker_AI_Orchestration.txt
└── summary/                            # 16개 LLM 생성 구조화 요약 .md
    └── (세션별 .md 파일)
```

### 메타 아티팩트: summary-prompt.md

이 파일은 이 행사 아카이브의 독특한 특징이다. 세션 요약을 생성하는 데 사용된 LLM 프롬프트 자체가 소스로 보존되어 있다. 프롬프트의 핵심 지시 내용은 다음과 같다:

```
요약 형식:

개요
- 영상의 주요 내용을 2-3문장으로 요약

주요 주제/섹션
- 각 주요 섹션 제목 [MM:SS] 또는 [HH:MM:SS] 형식으로 시작 시간 표시
  - 하위 항목들은 들여쓰기로 구조화

타임코드 추출 규칙:
1. 전사 텍스트에 [00:00], [1:23:45] 등의 타임스탬프가 있으면 해당 섹션의 시작 시간으로 사용
...

핵심 포인트 및 세부 내용
결론 및 권장사항
```

이 프롬프트의 존재는 두 가지를 의미한다. 첫째, 이 아카이브는 **원본 트랜스크립트(sessions/) + LLM 큐레이션 레이어(summary/)의 이중 구조**로 설계되어 있다. 둘째, 큐레이션 방법론이 투명하게 공개되어 있어 재현 가능하다. 다른 이벤트의 트랜스크립트에 동일 프롬프트를 적용하면 동일한 형식의 요약을 생성할 수 있다. 이는 OpenClaw 생태계가 자체 지식 관리에도 에이전트를 활용하는 메타 패턴의 증거다.

---

## 4. 콘텐츠 분析

### 세션 전체 목록

| # | 발표자 | 주제 |
|---|--------|------|
| 1 | Instruct.KR 운영진 | 오프닝 |
| 2 | Logan Kang | Composable Agent Economy |
| 3 | 최재훈 | Remote OpenClaw 다중디바이스 |
| 4 | 이상현 | Serverless Agent |
| 5 | 정우석 | OpenClaw 스킬 효율적으로 붙이기 |
| 6 | 김서준 | AI 에이전트와 온체인 경제 |
| 7 | 진주성 | M3 Ultra SNS 크롤링 |
| 8 | 민용기 | 에이전트 엔진룸 툴콜링 |
| 9 | 윤주운 | Arbiter FLOCK 워크플로우 |
| 10 | 허예찬 | OMX/OMC/Claw 운영생태계 |
| 11 | 한수관 | ResearchClaw |
| 12 | 김동규 | nanoclaw 사용기 |
| 13 | Lionel Sim | OpenClaw Singapore |
| 14 | Zoe Chen | Unibase: Extending OpenClaw |
| 15 | 김우현 | 중고PC 세컨드브레인 |
| 16 | 정세민 | Ultraworker AI Orchestration |

### 주제별 클러스터링

| 클러스터 | 세션 번호 | 핵심 내용 | 관련 기존 보고서 |
|---------|----------|-----------|---------------|
| 인프라/배포 | 03, 04, 07, 12, 15 | 다중디바이스, 서버리스, 로컬LLM, 경량, 중고PC | `reports/deployment/` |
| 온체인/에이전트 경제 | 02, 06, 14 | X402, ACP, ERC-8004, Virtuals ACP, Base | `reports/meetup/agent_payment_protocol_report.md` |
| 운영 패턴 | 05, 08, 09, 10, 16 | 스킬, 툴콜링, 워크플로우, 팀운영, 오케스트레이션 | `reports/meetup/meetup_patterns_report.md` |
| 연구 자동화 | 11 | ResearchClaw, FSM 기반 루프, 하드웨어 연동 | `reports/repos_research/` |
| 글로벌 커뮤니티 | 01, 13, 15 | 서울, 싱가포르, 세컨드브레인, ClawCon 확정 | — |

---

### 세션별 심층 분석

#### Session 2 — Logan Kang: Composable Agent Economy

Logan Kang(BASE Korea 앰배서더)은 에이전트 간 자율 거래를 위한 3개 레이어 스택을 소개했다.

- **OpenClaw**: 에이전트 생성·실행 런타임
- **Virtuals ACP**: 에이전트 마켓플레이스 프로토콜 (서비스 디스커버리 + 에스크로 + 흥정)
- **Base**: 이더리움 L2, USDC 스테이블코인 정산 레이어

핵심 문제 의식은 기존 결제 방식의 구조적 한계다: "API 키는 단순 인증만 가능, 구독/크레딧은 작업 단위 에이전트 분배 어려움, PG 결제(카드/카카오페이)는 인간이 직접 승인해야 하므로 에이전트가 대기해야 함."

**X402 프로토콜**은 HTTP 402 상태코드에서 영감을 받아, 에이전트가 유료 API 엔드포인트 호출 시 블록체인 지갑으로 결제 서명(수표)을 생성하여 재전송하는 방식이다. 전체 흐름에 인간 개입이 없다.

**에이전트 흥정(Negotiation)**: Virtuals ACP는 에이전트 간 가격 협상 기능을 프로토콜 수준에서 내장한다. "0.1 → 0.09로 해줘" 같은 협상이 에이전트 간 자동으로 이루어진다. 이는 기존 11개 Claw 프레임워크 어디에서도 볼 수 없는 패턴이다.

**소액 결제 단위**: USDC 기준 $10⁻⁶(약 0.001원)까지 지원. "사소한 기능도 수익화 가능"하다는 전망과 함께 **에이전트 애즈(Agent Ads)** — 유저 컨텍스트를 보유한 에이전트가 쇼핑을 대행하며 구매 전환율을 극대화하는 모델 — 을 미래 방향으로 제시했다.

---

#### Session 4 — 이상현: Serverless Agent

발표자는 AWS Lambda + TypeScript + MQTT를 조합한 서버리스 에이전트를 구현했다.

핵심 아이디어는 **"LLM이 직접 TypeScript 코드를 생성하고, tsc 타입 체크로 안전성 검증 후 실행"**하는 구조다. 이 접근에서 별도의 tool definition이 필요 없다. 타입 정의 파일(`.d.ts`)이 tool schema를 대체한다.

구체적 구조:
- **클라우드**: AWS Lambda (상태 없음, 무제한 확장)
- **로컬 접근**: MQTT 브리지 (포트 포워딩 없이 방화벽 뒤 데스크톱 접근)
- **비용**: 테스트 기준 약 $1-2

"LLM이 TypeScript를 작성 → tsc가 타입 오류 검증 → 통과 시 실행"이라는 파이프라인은 도구 정의 없이도 타입 시스템을 안전장치로 활용한다는 점에서 R1~R28 패턴들과 구분되는 독창적 접근이다 (R29 후보, 5절에서 상세 논의).

---

#### Session 5 — 정우석: OpenClaw 스킬 효율적으로 붙이기

비개발자 발표자가 세 개의 OpenClaw 인스턴스(M백 PC, 맥북, 시놀로지 서버)를 운영하며 구축한 **스킬 개발 4단계 프로세스**를 공유했다:

1. **요청**: URL과 함께 "어떤 방식으로 실행돼? 어떤 우려가 있어?" 질문
2. **실행**: 계획 수립 먼저 → 반복 피드백 후 실행 ("계획 없이 실행하면 방향이 엇나가는 경우 발생")
3. **고정**: "지금 실행한 걸 스킬로 고정하자"고 지시 → 스킬 자동 저장
4. **트리거**: 트리거 워드 및 문서화 요청 → 스킬 완성

가장 흥미로운 발견은 **OpenClaw와 Claude Code의 역할 분담**이다:
- **OpenClaw**: 기획, 오케스트레이션, 검수
- **Claude Code**: 복잡한 작업, 제작·실행, 고도화된 스킬 개발
- "복잡한 명령은 '네가 Claude Code를 진행하면 좋겠다'고 지시하면 자동으로 세팅"

n8n + MCP 조합을 시도했으나 "Claude Code 스킬 방식이 더 효율적"이라는 결론을 내렸다. 이는 동일한 자동화 문제에 대해 실무자가 여러 도구를 비교·실험한 결과로, 실용적 가치가 높다.

---

#### Session 10 — 허예찬: OMX/OMC/Claw 운영생태계

OMC(oh-my-claudecode) 및 OMX 프레임워크를 운영하는 발표자가 **대규모 에이전트 팀 운영**의 핵심 패턴을 공유했다.

**MEMORY.md 포인터맵 패턴**이 이 세션의 가장 중요한 발견이다. 기존 13개 Claw 프레임워크 대부분은 MEMORY.md에 실제 내용을 저장한다. 하지만 이 접근은 "에이전트가 실제 내용이 저장된 파일들의 경로(포인터)만 기록하는 지도(map)로 활용"한다. 에이전트는 bash/grep/ls 도구로 메모리를 직접 탐색한다.

> "MEMORY.md는 지도다. 실제 지형이 아니다."

이 패턴의 장점:
- 벡터 DB 없이 10만 줄 이상 메모리 관리 가능
- 에이전트가 필요한 부분만 선택적으로 읽음 (전체 로드 불필요)
- 파일 시스템 도구(bash/grep)만으로 의미 기반 탐색 가능

**agents.md 교리 엔진**: agents.md는 단순한 에이전트 지시 파일이 아니라 "교리 엔진"으로 작동한다. 에이전트의 행동 원칙, 판단 기준, 협업 규약이 집약된 문서로, "80% 이상의 OMC/OMX PR이 에이전트 자신에 의해 생성"되는 자율 기여 생태계를 가능하게 한다.

이 패턴은 R17(Frozen Snapshot), R18(Char-Limited Memory)과 명확히 구분된다 (R30 후보, 5절에서 상세 논의).

---

#### Session 11 — 한수관: ResearchClaw

AER Labs의 한수관은 반도체·로보틱스 연구 자동화를 위한 **결정론적 FSM(유한 상태 기계) 기반 에이전트 하네스**를 소개했다.

핵심 문제는 단일 에이전트에 모든 것을 맡겼을 때 "컨텍스트가 뒤섞이는 문제"와 "평가 코드와 실험 코드의 혼재"였다. 해결책은 **플래닝 / 실험 / 평가 단계를 명확히 분리**하고 각 단계 에이전트의 권한(디렉토리, 역할)을 명시적으로 제한하는 FSM 구조다.

```
플래닝 에이전트 (사람과 상호작용)
    ↓ 스펙 확정
실험 에이전트 (코드 작성 → 반드시 실행 강제)
    ↓ 실험 결과
평가 에이전트 (시각화, 별도 디렉토리)
```

"각 에이전트에게 특정 디렉토리 내에서만 작업하도록 권한을 제한하는 것이 안정성의 핵심"이라는 발언은 IronClaw의 능력 감쇠(capability attenuation) 패턴과 유사하지만, FSM 상태 전환 제어와 결합된 형태로 구현했다는 점에서 차별화된다.

하드웨어 연동 현황: 시뮬레이션 환경에서는 안정적으로 동작. 실제 FPGA + 로봇 환경 전환 시 레이턴시와 툴콜 해석 오류가 현재 과제. 반도체 RTL 코드 자동 생성도 검증 완료.

---

#### Session 16 — 정세민 (Sionic AI): Ultraworker AI Orchestration

정세민은 Claude Code 기반 "Ultraworker" 시스템을 소개했다. 본질은 "MCP를 붙인 Claude Code"로, 슬랙을 Human-in-the-Loop 인터페이스로 활용한다.

**4단계 워크플로**:
1. 컨텍스트 탐색 (explore context 스킬)
2. 작업 리스트 작성 → 슬랙 승인 요청
3. 테크 스펙 작성 (복잡한 업무에만)
4. 구현 완료 후 보고

총 3회 승인 프로세스를 슬랙 블록킷 UI(좋아요 버튼)로 구현. "에이전트가 어떻게 동작하는지 실시간으로 관제·중재하는 것이 가장 중요"하다는 Human-in-the-Loop 철학을 명확히 밝혔다.

**하이브리드 메모리 아키텍처**: 에이전트의 장기 기억 한계를 BM25 + 벡터DB(Qdrant) + 그래프DB 3계층 구조로 보완한다. 특이한 점은 **육하원칙(5W1H) 기반 룰베이스 온톨로지**로 맥락 랭킹을 구성한다는 것이다. Q&A에서 발표자는 "파일 시스템은 키워드 매칭 한계, 벡터DB는 의미적 유사도에 강하나 키워드가 완전히 다를 경우 한계, 그래프DB는 관계형 맥락 연결"이라며 각 레이어의 역할을 명확히 구분했다.

**실운용 규모**: 개인 데스크톱부터 RTX 6000 6대, 30대 서버에 분산 운용. 오래된 슬랙 채널 기록을 스캔하여 Rust 서비스 장애 원인(캐시 이슈, 마이그레이션 이슈 등 3가지)을 자동으로 파악한 실사용 사례를 공개했다.

---

## 5. 신규 패턴 (R-번호)

### R29 후보 — TypeScript-as-Tool 실행 패턴 (이상현, Session 4)

**패턴 설명**: LLM이 TypeScript 코드를 직접 생성하고, tsc(TypeScript 컴파일러)의 타입 검사를 안전성 검증 게이트로 활용하여 실행. 별도 tool definition 파일이 필요 없으며, 타입 정의 파일(`.d.ts`)이 tool schema를 대체한다.

**기존 R1~R28과의 비교**:

| R번호 | 패턴 | 비교 |
|-------|------|------|
| R20 (Skills Trust Levels) | Hermes Agent — 스킬 신뢰 등급 정적 분석 | 실행 전 정적 검증이라는 공통점 있으나, TypeScript 타입 시스템 활용은 전혀 다른 접근 |
| R22 (Tirith Pre-Exec Scanner) | Hermes Agent — SHA-256 + cosign 바이너리 스캔 | 바이너리 무결성 검증 vs. 코드 타입 안전성 검증 — 다른 레이어 |
| R15 (정적 컴파일) | NullClaw — Zig 정적 바이너리 | 컴파일 타임 검증이라는 공통점 있으나, 에이전트 생성 코드를 실시간 검증하는 개념은 부재 |

**신규성 판단**: R1~R28 중 "LLM이 생성한 코드를 타입 컴파일러로 실시간 검증 후 tool로 실행"하는 패턴은 없다. 기존 패턴들은 사전 정의된 tool schema를 에이전트가 활용하는 구조이나, 이 패턴은 타입 정의가 schema를 대체하여 tool 정의 자체를 런타임에 생성한다.

**→ R29 부여 결정**

```
R29: TypeScript-as-Tool 동적 실행 패턴 (이상현, OpenClaw 서울 밋업)
구현: AWS Lambda + tsc + MQTT
원리: LLM이 TypeScript 코드를 작성하면 tsc가 타입 오류를 정적 검증한 뒤 실행.
      타입 정의 파일(.d.ts)이 tool schema를 대체하므로 tool definition 작성 불필요.
      타입 시스템을 에이전트 안전장치로 전용(轉用)하는 패턴.
시사점: tool schema 관리 비용 제거. 타입 정의만 있으면 어떤 API도 즉시 도구화 가능.
        서버리스 환경에서 stateless하게 동작하므로 무한 확장성.
        tsc 타입 검사 통과 = 실행 가능 보장(런타임 타입 오류 사전 차단).
관련 기술: AWS Lambda (실행 환경), MQTT (로컬 브리지), TypeScript (검증 레이어)
```

---

### R30 후보 — 포인터맵 메모리 아키텍처 (허예찬, Session 10)

**패턴 설명**: MEMORY.md는 실제 내용이 아닌 파일 경로 포인터(지도)만 저장. 에이전트가 bash/grep/ls 등 파일시스템 도구를 사용해 메모리를 직접 탐색. 벡터 DB나 별도 검색 인덱스 없이 10만 줄 이상의 기억을 관리.

**기존 R17~R18과의 상세 비교**:

| 비교 항목 | R17 Frozen Snapshot (Hermes) | R18 Char-Limited Memory (Hermes) | R30 후보 (허예찬) |
|----------|------------------------------|-----------------------------------|-----------------|
| 저장 방식 | MEMORY.md + USER.md에 실제 내용 | 문자 수 예산(2,200+1,375 chars) 제한 | MEMORY.md는 포인터만, 실제 내용은 별도 파일 |
| 로드 시점 | 세션 시작 시 1회 캡처 (불변) | 항상 전체 로드 | 필요한 부분만 선택적 로드 |
| 검색 방법 | 시스템 프롬프트 인컨텍스트 | 시스템 프롬프트 인컨텍스트 | bash/grep/ls 도구 활용 |
| 확장성 | ~3,575 chars까지 (모델 비종속) | 고정 예산 내 | 이론적으로 무제한 |
| 벡터 DB | 없음 | 없음 | 없음 |
| 핵심 목적 | prefix cache 보존 | 모델 교체 시 예산 일관성 | 대용량 메모리 탐색 비용 절감 |

**기존 session_context_report.md 및 memory_architecture_report.md 패턴과의 비교**:

기존 보고서에서 확인된 메모리 패턴들은 크게 "인컨텍스트 직접 로드"(TinyClaw, PicoClaw), "DB 검색 후 삽입"(OpenClaw, NanoClaw), "벡터+키워드 하이브리드"(IronClaw)로 분류된다. 포인터맵 패턴은 이 세 범주 모두에 해당하지 않는다. "에이전트가 파일시스템을 직접 탐색하는 도구적 메모리 접근"은 별개의 범주다.

**신규성 판단**: R1~R28, 그리고 기존 session_context_report.md / memory_architecture_report.md에서 "메모리 파일이 포인터 역할만 하고 에이전트가 bash/grep으로 탐색"하는 패턴은 확인되지 않는다.

**→ R30 부여 결정**

```
R30: 포인터맵 메모리 아키텍처 (허예찬, OMC/OMX)
구현: MEMORY.md(포인터맵) + 분산 콘텐츠 파일 + bash/grep/ls 탐색
원리: 중앙 메모리 파일은 실제 내용 대신 "어떤 파일에 어떤 정보가 있는지"만 기록.
      에이전트는 bash/grep/ls 도구로 필요한 파일을 직접 탐색하므로
      컨텍스트 윈도우에는 탐색 결과만 로드됨. 벡터 DB, 임베딩 불필요.
시사점: 벡터 DB 인프라 없이 100K+ 줄 메모리 관리 가능.
        에이전트의 도구 사용 능력(bash/grep)이 검색 능력 자체가 됨.
        메모리 구조가 파일시스템 구조와 동형(isomorphic) — 별도 추상화 레이어 불필요.
        대규모 팀 에이전트 운영(OMC/OMX 80%+ PR 자동 생성)에서 실증.
관련 사례: OMC/OMX 운영생태계, compare_claws 프로젝트의 MEMORY.md 설계 원칙과 유사
```

---

### R32 — Gateway-Node 멀티 디바이스 아키텍처 (최재훈, Session 03 + OpenClaw 공식 설계)

**배경**: OpenClaw의 공식 네트워크 모델(`docs/gateway/network-model.md`)은 Gateway-Node 분리 구조를 1등급 설계로 정의한다. 최재훈은 밋업에서 이를 실제 5기기 네트워크로 시연했다.

**핵심 구조**:
- Gateway(OCI 클라우드) ← WebSocket → Node(각 엔드 디바이스)
- 에이전트는 Gateway에만 존재 — Node는 순수 기능 서버(function server)
- 각 Node가 자신의 OS별 Capabilities를 Gateway에 Advertisement
- Tailscale VPN이 퍼블릭 IP 노출 없이 안전한 메시 구성

**최재훈의 구성** (밋업 → 페북 게시글 기준):

| 기기 | OS | Capabilities |
|------|-----|-------------|
| Mac Studio | macOS | browser + camera + canvas + location + screen |
| m2macmini | macOS | browser + system |
| Raspberry Pi 5 | Linux | browser + system |
| GPU server | Windows/WSL | browser + system |
| Game Machine | Windows/WSL | browser + system |

**공식 소스**: `repos/openclaw/docs/gateway/network-model.md`, `docs/platforms/oracle.md`, `docs/platforms/raspberry-pi.md`, `src/agents/pi-embedded-runner/`

**R21·R27과의 구분**:
- R21 (Hermes Bounded Delegation Tree): 에이전트→에이전트 위임 트리. 멀티 에이전트.
- R27 (MiClaw OS-Native): 에이전트가 OS 레이어에 통합.
- **R32**: 단일 에이전트 + N개 디바이스. 에이전트는 1개, Node는 기능 서버. 수평 확장.

→ **R32 부여**

---

## 6. 비교 테이블

### 커뮤니티 아카이브 유형별 비교

| 비교 항목 | OpenClaw 서울 밋업 0315 | autoresearch-skill (응용 계층) | 일반 기술 컨퍼런스 영상 |
|----------|------------------------|-------------------------------|----------------------|
| **큐레이션 방식** | LLM(summary-prompt.md) + 구조화 .md | GitHub README + 코드 주석 | 자막 + 슬라이드 PDF |
| **원본 보존** | sessions/ 16개 원본 .txt 보존 | 소스코드 자체가 원본 | 영상 링크(외부 의존) |
| **방법론 투명성** | summary-prompt.md 공개 (재현 가능) | 코드 오픈소스 | 없음 |
| **코드 포함** | 세션 내 코드 스니펫 다수 | 전체 소스코드 | 슬라이드 일부 |
| **언어** | 한국어 (영어 세션 2개) | 영어 | 영어/다국어 |
| **지리적 범위** | 한국 + 싱가포르 | 글로벌 (GitHub) | 글로벌 |
| **학술적 가치** | 실무 1차 데이터 (정성) | 기술 구현 분석 | 발표자 선별 편향 |
| **복제 가능성** | summary-prompt.md로 타 이벤트 적용 가능 | 포크 후 바로 실행 가능 | 낮음 |
| **에이전트 자동화** | 아카이브 생성 자체에 에이전트 활용 | 에이전트가 연구 실행 | 없음 |
| **패턴 발굴** | R29, R30 신규 패턴 2개 | (기존 보고서 분석 후 파악 필요) | 낮음 |

### 기존 비교 프레임워크 보고서와의 역할 구분

이 보고서(`reports/usecases/details/`)는 `reports/meetup/` 디렉토리의 파일들과 명확히 구분된다:

| 보고서 | 위치 | 역할 |
|--------|------|------|
| `meetup_patterns_report.md` | `reports/meetup/` | 여러 밋업에서 추출된 **반복 패턴** 집약 (패턴 카탈로그) |
| `agent_payment_protocol_report.md` | `reports/meetup/` | X402, ACP, ERC-8004 등 **결제 프로토콜** 기술 분석 |
| 이 보고서 | `reports/usecases/details/` | **단일 이벤트** 상세 분석 — 1차 소스 데이터, 신규 패턴 발굴 |

---

## 7. 한계

**한국어 전용 트랜스크립트**: sessions/ 하위 원본 파일 대부분이 한국어로 작성되어 있어 글로벌 접근성이 제한된다. Logan Kang, Lionel Sim, Zoe Chen의 세션은 영어이나 요약은 한국어로 작성되었다.

**LLM 요약 레이어의 정보 손실**: summary-prompt.md의 존재가 큐레이션 방법론을 투명하게 공개하는 장점이 있는 반면, sessions/(원본)과 summary/(요약) 사이에 LLM이 선택적으로 추출한 정보만 남는다. 발표자의 즉흥 발언, Q&A의 뉘앙스, 청중 반응 등이 손실될 수 있다.

**슬라이드·영상 미보존**: openclaw-meetup.png 외에 발표 슬라이드나 영상이 아카이브에 포함되어 있지 않다. 시각적 다이어그램, 라이브 데모 화면 등은 텍스트 트랜스크립트로 복원 불가능하다.

**단일 시점 스냅샷**: 이 아카이브는 2026년 3월 15일 상태를 반영한다. 발표된 프로젝트들(ResearchClaw, Ultraworker 등)의 현재 상태나 이후 발전 방향은 이 보고서에서 추적 불가능하다.

**자기 선택 편향**: 밋업에 발표 신청한 16명은 OpenClaw 생태계에 적극적으로 참여하는 실무자들이다. 도입 실패 사례, 한계 경험, 비판적 시각은 상대적으로 과소 대표될 수 있다.

**큐레이션 편향**: summary-prompt.md의 구조(개요 → 주요 주제 → 핵심 포인트 → 결론)가 요약 내용의 틀을 사전에 규정한다. 이 틀에 맞지 않는 정보(예: 발표자가 우회적으로 언급한 실패담)는 요약에서 누락될 가능성이 있다.

**RSVP 비율의 해석 한계**: 950+ RSVP / 50-60명 수용이라는 수치는 인기를 보여주지만, RSVP의 실제 참여 의도(단순 흥미 vs. 실제 참석 의지)를 구분하기 어렵다.

---

## 8. 참고 링크

### 소스 파일

- `usecases/openclaw_seoul_meetup_0315/` — 본 보고서의 1차 소스
- `usecases/openclaw_seoul_meetup_0315/summary-prompt.md` — LLM 큐레이션 방법론
- `usecases/openclaw_seoul_meetup_0315/sessions/` — 16개 원본 트랜스크립트
- `repos/openclaw/docs/gateway/network-model.md` — Gateway-Node 공식 아키텍처 문서
- `repos/openclaw/docs/platforms/raspberry-pi.md` — Pi Gateway 공식 가이드
- `repos/openclaw/docs/platforms/oracle.md` — OCI + Tailscale 배포 가이드

### 관련 보고서 (이 프로젝트)

- `reports/meetup/meetup_patterns_report.md` — 밋업 운영 패턴 10개 집약
- `reports/meetup/agent_payment_protocol_report.md` — X402, ACP, ERC-8004 결제 프로토콜 분석
- `reports/usecases/usecases_index.md` — 유스케이스 인덱스
- `reports/repos/memory_architecture_report.md` — 메모리 아키텍처 비교 (R30 배경)
- `reports/repos_research/research_tools_report.md` — ResearchClaw 관련 연구 자동화 도구 분석
- `reports/deployment/` — 서버리스·배포 전략 비교 (Session 4 배경)

### 신규 패턴 등록

본 보고서에서 발굴된 패턴:
- **R29**: TypeScript-as-Tool 동적 실행 패턴 (이상현, Session 4)
- **R30**: 포인터맵 메모리 아키텍처 (허예찬, Session 10)
- **R32**: Gateway-Node 멀티 디바이스 아키텍처 (최재훈, Session 03 + OpenClaw 공식 설계)

→ MEMORY.md New Patterns 섹션에 R29, R30, R32 추가 필요

---

*분석 완료: 2026-03-21*
*다음 분석 대상: ClawCon Seoul (확정 발표됨, 일정 미정)*
