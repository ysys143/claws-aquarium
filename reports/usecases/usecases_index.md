# usecases/ 교차 분析 보고서

> **작성일**: 2026-03-21
> **계층**: 4번째 계층 — 커뮤니티 콘텐츠 & 실사용 모음

---

## 목차

1. [usecases/ 계층이란](#1-usecases-계층이란)
2. [등록 항목 목록](#2-등록-항목-목록)
3. [교차 분析](#3-교차-분석)
4. [기존 분析과의 연결](#4-기존-분析과의-연결)
5. [신규 패턴 요약 (R29–R32)](#5-신규-패턴-요약-r29r32)
6. [열린 질문](#6-열린-질문)

---

## 1. usecases/ 계층이란

compare_claws의 기존 3계층은 **코드**를 분析 대상으로 삼는다:

| 계층 | 대상 | 질문 |
|------|------|------|
| `repos/` | 에이전트 런타임 13개 | OpenClaw 생태계의 엔진은 무엇인가? |
| `repos_applied/` | 응용 프로젝트 5개 | 그 위에 무엇이 쌓였는가? |
| `repos_research/` | 연구 자동화 2개 | AI가 어떻게 연구를 자동화하는가? |

`usecases/`는 **커뮤니티 콘텐츠**를 분析 대상으로 삼는다:

> "실제 개발자들이 OpenClaw로 무엇을 만들고, 어떻게 운영하고, 어떤 문제를 겪는가?"

코드 저장소로는 답할 수 없는 질문 — 생태계의 **수요 신호(demand signal)**와 **실무 패턴** — 을 1차 소스 데이터에서 직접 추출한다.

**usecases/ 항목 유형**:
- **Awesome List**: 커뮤니티가 검증한 활용 사례 또는 에이전트 템플릿 모음
- **Meetup/Event**: 실무자들의 직접 발표 전사(transcript) + 요약 모음

---

## 2. 등록 항목 목록

| 이름 | 유형 | 소스 | 항목 수 | 상세 보고서 |
|------|------|------|---------|------------|
| awesome-openclaw-usecases | Awesome List (활용 사례) | github.com/hesamsheikh/awesome-openclaw-usecases | 40개 use-case | [details/awesome_openclaw_usecases_report.md](details/awesome_openclaw_usecases_report.md) |
| awesome-openclaw-agents | Awesome List (에이전트 템플릿) | github.com/mergisi/awesome-openclaw-agents | 174개 SOUL.md + 132개 use-case | [details/awesome_openclaw_agents_report.md](details/awesome_openclaw_agents_report.md) |
| openclaw_seoul_meetup_0315 | 커뮤니티 밋업 | 로컬 — usecases/openclaw_seoul_meetup_0315/ | 16세션 | [details/openclaw_seoul_meetup_0315_report.md](details/openclaw_seoul_meetup_0315_report.md) |
| agency-agents | 멀티-툴 에이전트 페르소나 컬렉션 | github.com/msitarzewski/agency-agents | 60개+ 에이전트, 8 Division | [details/agency_agents_report.md](details/agency_agents_report.md) |

**총합**: 4개 항목 | 40 + 174 + 16 + 60 = 290개 실사용 데이터 포인트

---

## 3. 교차 분析

### 3.1 도메인 분포 — 무엇을 만드는가

세 항목을 합산하면 OpenClaw 생태계의 실제 활용 도메인 분포가 드러난다:

| 도메인 | usecases (hesamsheikh) | agents (mergisi) | meetup (16세션) | 비고 |
|--------|----------------------|-----------------|----------------|------|
| 생산성 / 개인 자동화 | 18개 (45%) | personal 8 + productivity 12 | 세션 5, 10, 15 | **압도적 1위** — "일상 AI 비서" 수요 |
| 개발 / DevOps | 2개 | development 15 + devops 10 | 세션 3, 4, 12 | 개발자 자동화 두드러짐 |
| 비즈니스 / 마케팅 | 1개 | business 14 + marketing 19 | — | 전문 업무 자동화 강세 |
| 콘텐츠 / 크리에이티브 | 5개 | creative 5 | 세션 7 | 소셜미디어 파이프라인 수요 |
| 온체인 / Web3 | 0개 (명시 제외) | moltbook 2 | 세션 2, 6, 14 | hesamsheikh만 제외 정책 |
| 연구 / 지식 관리 | 6개 | — | 세션 11, 15 | 밋업에서 유독 강조 |
| 인프라 / 하드웨어 | 2개 | devops 10 | 세션 3, 4, 7, 12 | 밋업 실무자층 특징 |

**핵심 발견**: 생산성/개인 자동화 클러스터가 전 소스에서 1위다. 이는 repos/의 13개 프레임워크 중 어느 것도 "개인 사용자용"을 설계 목표로 명시하지 않는다는 점과 대비된다 — 프레임워크 설계 목표와 실제 수요 사이의 간극이 존재한다.

### 3.2 채널 선호도 — 어디서 작동하는가

세 항목에서 반복적으로 등장하는 채널:

| 채널 | 등장 빈도 | 비고 |
|------|----------|------|
| **Telegram** | 높음 (8개+ use-case, 밋업 다수) | 사실상 표준 채널 |
| **Phone/SMS** | 4개 use-case | 새로운 수요 — 프레임워크 지원 미흡 |
| **Slack** | 3개 use-case, 밋업 세션 16 | 업무용 채널 |
| **WhatsApp** | 2개 use-case | 엔터프라이즈 진입 |
| **Discord** | 밋업 구성 채널 | 커뮤니티 특화 |
| **Web UI** | agents (crewclaw.com) | no-terminal 배포 수요 |

→ NullClaw의 19개 채널 전략이 실제 수요와 가장 정렬되어 있다. 대다수 프레임워크가 Telegram+Discord에 집중한 것은 실제 수요 대비 과소 다양성.

### 3.3 배포 패턴 — 어떻게 운영하는가

| 패턴 | 소스 | 대표 사례 |
|------|------|---------|
| 로컬 Mac Mini 24/7 | meetup 세션 7, 15 | M3 Ultra 로컬 LLM, 중고PC 세컨드브레인 |
| 서버리스 클라우드 | meetup 세션 4 | AWS Lambda + MQTT |
| 홈랩 분산 | meetup 세션 9 | FLOCK + Arbiter |
| No-terminal 원클릭 | agents (crewclaw.com) | 177개 SOUL.md → crewclaw deploy |
| n8n 위임 | usecases 세션 | 자격증명 격리 + n8n 워크플로우 |
| 다중디바이스 통합 | meetup 세션 3 | OCI 무료 인스턴스 게이트웨이 |

→ 배포 패턴이 매우 다양하다. 프레임워크 docs가 "로컬 실행" 중심인 것과 달리, 실무자들은 클라우드/서버리스/분산으로 적극 이동 중.

### 3.4 보안 인식 격차

| 소스 | 보안 언급 |
|------|----------|
| awesome-openclaw-usecases | README에 명시적 보안 경고 — "third-party skills may have critical vulnerabilities, not audited" |
| awesome-openclaw-agents | 보안 카테고리(24개 중 1개)만 존재, 템플릿 자체 보안 검증 없음 |
| meetup 세션 10 | 파일 시스템 기반 메모리 → 벡터 DB 없애는 것이 보안 단순화에도 기여 |

→ 프레임워크 계층(Tier 1–4 보안 분류)과 달리, 커뮤니티 콘텐츠 계층에서는 보안이 취약한 고리다. Hermes Agent의 R20 Skills Trust 4단계와 R22 Tirith Pre-Exec Scanner가 가장 직접적으로 이 문제를 해결한다.

---

## 4. 기존 分析과의 연결

| 기존 보고서 | usecases/ 콘텐츠와의 관계 |
|------------|------------------------|
| `reports/repos/security_report.md` | awesome-openclaw-agents의 보안 검증 부재 → Tier 4에 해당. Hermes Agent R20 적용 필요 |
| `reports/repos/details/hermes_agent_report.md` | SOUL.md = Hermes Agent의 USER.md/MEMORY.md 철학과 동일 계보 |
| `reports/repos/details/nullclaw_report.md` | NullClaw 19채널 설계 ↔ Telegram 편중 실수용 패턴 — 설계/실수요 격차 |
| `reports/repos_applied/repos_applied_report.md` | crewclaw.com이 repos_applied/ 진입 자격 검토 필요 (Moltbook과 유사한 배포 플랫폼) |
| `reports/deployment/` | meetup 세션 3·4·7이 deployment 전략 3가지 실증 (다중디바이스, 서버리스, 로컬) |
| `reports/meetup/` | meetup_patterns_report.md (10패턴), agent_payment_protocol_report.md (5프로토콜) 교차 링크 |

**새로 드러난 갭**: 어떤 프레임워크도 "Phone/SMS를 1등급 채널"로 설계하지 않았다. 4개 use-case가 전화 인터페이스를 요구하지만, 13개 프레임워크 중 Voice 채널은 NullClaw 1개뿐이며 그마저도 Voice over IP 기반이다.

---

## 5. 신규 패턴 요약 (R29–R34)

세 usecases/ 항목 분析에서 R1–R28에 없는 패턴 3개를 발굴했다:

| R번호 | 패턴명 | 발굴 소스 | 핵심 원리 |
|-------|--------|----------|----------|
| **R29** | TypeScript-as-Tool 동적 실행 | 이상현, 밋업 세션 04 | LLM이 TypeScript 코드 직접 생성 → tsc 타입 체크 → 실행. 타입 정의 파일이 tool schema를 대체. 별도 tool definition 불필요. |
| **R30** | 포인터맵 메모리 아키텍처 | 허예찬, 밋업 세션 10 | MEMORY.md에 실제 내용 대신 파일 경로 포인터만 저장. 에이전트가 bash/grep/ls로 탐색. 벡터 DB 없이 10만 줄 이상 메모리 관리. |
| **R31** | Shared-State File Coordination | awesome-openclaw-usecases, STATE.yaml 패턴 | 중앙 오케스트레이터 없는 분산 멀티에이전트 조율. STATE.yaml/JSON 공유 파일에 각 서브에이전트가 직접 읽기/쓰기로 상태 동기화. |
| **R32** | Gateway-Node 멀티 디바이스 아키텍처 | OpenClaw 공식 설계, 최재훈 밋업 세션 03 | 에이전트는 Gateway에만 존재. 각 디바이스(macOS/Linux/Windows/RPi)의 Node가 OS별 Capabilities를 Gateway에 Advertisement. Tailscale VPN 메시로 연결. |

| **R34** | Multi-Platform Agent Persona Transpiler | agency-agents | 단일 `.md` 소스 -> convert.sh -> 10개 도구별 포맷. Claude Code/.md, OpenClaw/SOUL.md+AGENTS.md+IDENTITY.md, Cursor/.mdc, Aider/CONVENTIONS.md 등. "Write Once, Deploy Everywhere" for agent personas. |

상세 내용은 각 상세 보고서의 섹션 5 참조.

---

## 6. 열린 질문

- **Q39** (usecases/): Telegram 편중이 심화되면 OpenClaw 생태계가 Telegram 플랫폼 정책 변경에 취약해지는가? Signal/Matrix 대안 수요가 충분한가?
- **Q40** (usecases/): crewclaw.com(awesome-openclaw-agents의 배포 플랫폼)이 OpenClaw 공식 플랫폼인가 서드파티인가? repos_applied/ 등록 자격이 있는가?
- **Q41** (usecases/): 커뮤니티 큐레이션(hesamsheikh, mergisi)과 공식 ClawHub 마켓플레이스 간의 품질·보안 기준 차이가 생태계 분열을 초래하는가?
- **Q42** (usecases/): R29 TypeScript-as-Tool 패턴이 타입 정의 파일 유지보수 부담을 도구 호출 스키마 정의 부담으로 단순히 치환하는가, 아니면 실질적으로 더 단순한가?
- **Q43** (usecases/): R30 포인터맵 메모리 패턴이 bash/grep을 사용하므로 Windows 환경(PowerShell)에서는 이식성 문제가 발생하는가?

---

## 참고 링크

- [details/awesome_openclaw_usecases_report.md](details/awesome_openclaw_usecases_report.md)
- [details/awesome_openclaw_agents_report.md](details/awesome_openclaw_agents_report.md)
- [details/openclaw_seoul_meetup_0315_report.md](details/openclaw_seoul_meetup_0315_report.md)
- [../meetup/meetup_patterns_report.md](../meetup/meetup_patterns_report.md)
- [../meetup/agent_payment_protocol_report.md](../meetup/agent_payment_protocol_report.md)
- [../repos_applied/repos_applied_report.md](../repos_applied/repos_applied_report.md)
