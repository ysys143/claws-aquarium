# agency-agents 상세 분析 보고서

> **작성일**: 2026-03-23
> **계층**: usecases/ -- 커뮤니티 콘텐츠 & 실사용 모음

---

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub URL** | https://github.com/msitarzewski/agency-agents |
| **Stars** | 59,300+ |
| **Forks** | 8,900+ |
| **라이선스** | MIT |
| **작성자** | @msitarzewski |
| **언어** | Markdown (에이전트 파일) + Bash (설치 스크립트) |
| **커밋 수** | 210+ |
| **항목 수** | 8개 Division, 60개 이상 에이전트 |
| **유형** | 멀티-툴 AI 에이전트 페르소나 템플릿 컬렉션 |
| **파생 레포** | awesome-openclaw-agents (mergisi) -- 이 레포의 파생 포크 |

---

## 2. 핵심 특징

**agency-agents**는 "완전한 AI 에이전트 팀을 손끝에"라는 슬로건 아래, 전문화된 AI 페르소나 템플릿을 모은 컬렉션이다. Reddit 스레드에서 시작해 수개월의 반복을 거쳐 성장한 이 프로젝트는 59k+ Stars로 OpenClaw 생태계에서 가장 큰 에이전트 템플릿 저장소 중 하나다.

**핵심 차별점**:
- **멀티-툴 지원**: Claude Code, GitHub Copilot, Antigravity/Gemini, OpenClaw, Cursor, Aider, Windsurf, OpenCode, Qwen Code -- 10개 AI 도구 동시 지원
- **convert.sh 트랜스파일러**: 단일 `.md` 소스 -> 도구별 포맷으로 자동 변환 (R34 신규 패턴)
- **업스트림 소스**: awesome-openclaw-agents(mergisi, 174 SOUL.md)의 원본 저장소
- **에이전트 설계 철학**: 단순 프롬프트가 아닌 "인격(personality) + 워크플로우 + 성공 지표"를 가진 전문가

compare_claws 맥락에서의 위치: awesome-openclaw-agents의 업스트림 소스이므로, 두 레포를 함께 읽어야 OpenClaw 에이전트 템플릿 생태계의 전체 그림을 볼 수 있다.

---

## 3. 구조 分析

### 디렉토리 구조

```
agency-agents/
├── academic/            # 학술 전문가 (5개)
├── design/              # 디자인 전문가
├── engineering/         # 엔지니어링 (9개+)
├── game-development/
│   ├── unity/           # Unity 전문 (4개)
│   └── unreal/          # Unreal Engine 전문
├── integrations/        # 도구별 통합 가이드
│   ├── claude-code/
│   ├── openclaw/
│   ├── antigravity/
│   ├── cursor/
│   └── ...
├── marketing/           # 마케팅 (15개+)
├── paid-media/          # 유료 광고 (6개+)
├── product/             # 프로덕트 (5개)
├── examples/            # 다중 에이전트 협업 예시
│   └── nexus-spatial-discovery.md
├── scripts/
│   ├── convert.sh       # 도구별 포맷 변환기
│   └── install.sh       # 인터랙티브 설치 UI
└── README.md
```

### 8개 Division 구성

| Division | 대표 에이전트 | 특이사항 |
|----------|-------------|---------|
| **Engineering** | Frontend Developer, Backend Architect, AI Engineer, Security Engineer, Solidity Smart Contract Engineer, Autonomous Optimization Architect | 9개+, 가장 풍부 |
| **Marketing** | China E-Commerce Operator, Kuaishou Strategist, Douyin Strategist, Livestream Commerce Coach, SEO Specialist | 중국 시장 에이전트 다수 |
| **Design** | Visual Storyteller, Inclusive Visuals Specialist | AI 이미지 생성 프롬프트 전문가 포함 |
| **Product** | Sprint Prioritizer, Trend Researcher, Feedback Synthesizer, Behavioral Nudge Engine, Product Manager | 행동경제학 기반 Nudge Engine |
| **Academic** | Anthropologist, Geographer, Historian, Narratologist, Psychologist | 세계관 구축·스토리텔링 특화 |
| **Game Development** | Unity 4종, Unreal 1종+, Game Audio Engineer, Narrative Designer | 게임 특화 유일한 컬렉션 |
| **Paid Media** | PPC Campaign Strategist, Search Query Analyst, Paid Media Auditor, Ad Creative Strategist | 200포인트 감사 시스템 |
| **Support** | Analytics Reporter 및 기타 | 크로스 Division 지원 |

### 에이전트 파일 구조 (공통 템플릿)

모든 에이전트 파일은 동일한 8섹션 구조를 따른다:

```markdown
---
name: "Frontend Developer"
description: "React/Vue/Angular expert"
color: "#61DAFB"
---

## Identity & Memory
[에이전트 자기 정의 + 세션 간 패턴 학습 지침]

## Core Mission
[전문 영역 1문단 요약]

## Critical Rules
[도메인별 필수 준수 규칙]

## Technical Deliverables
[구체적 산출물 + 코드 예시]

## Workflow Process
[단계별 프로세스]

## Success Metrics
[측정 가능한 성공 기준]
```

**핵심 철학**: 각 에이전트는 단순 프롬프트 템플릿이 아닌 "인격(personality)"을 가진다. Identity & Memory 섹션이 세션 간 패턴 인식과 지속적 개선을 명시한다.

---

## 4. 콘텐츠 分析

### 4.1 도구별 변환 포맷

convert.sh는 단일 `.md` 소스를 다음 포맷으로 트랜스파일한다:

| 도구 | 변환 포맷 | 설치 경로 |
|------|---------|---------|
| **Claude Code** | 그대로 `.md` | `~/.claude/agents/` |
| **GitHub Copilot** | 그대로 `.md` | `~/.github/agents/`, `~/.copilot/agents/` |
| **OpenClaw** | `SOUL.md` + `AGENTS.md` + `IDENTITY.md` (3파일) | `~/.openclaw/` |
| **Antigravity (Gemini)** | `SKILL.md` per agent | `~/.gemini/antigravity/skills/` |
| **Gemini CLI** | extension + `SKILL.md` | `~/.gemini/extensions/agency-agents/` |
| **OpenCode** | `.md` agent files | `.opencode/agents/` |
| **Cursor** | `.mdc` rule files | `.cursor/rules/` |
| **Aider** | 단일 `CONVENTIONS.md` | `./CONVENTIONS.md` |
| **Windsurf** | 단일 `.windsurfrules` | `./.windsurfrules` |
| **Qwen Code** | `.md` SubAgent files | `~/.qwen/agents/` |

**특이점**: OpenClaw는 3개 파일(SOUL.md + AGENTS.md + IDENTITY.md)로 분리된다. awesome-openclaw-agents(파생 포크)가 SOUL.md만 제공한 것과 달리, 원본은 더 풍부한 OpenClaw 통합을 제공한다.

### 4.2 install.sh 인터랙티브 UI

```
+------------------------------------------------+
|   The Agency -- Tool Installer                 |
+------------------------------------------------+

System scan: [*] = detected on this machine

[x]  1)  [*]  Claude Code     (claude.ai/code)
[x]  2)  [*]  Copilot         (~/.github + ~/.copilot)
[x]  3)  [*]  Antigravity     (~/.gemini/antigravity)
[ ]  4)       Gemini CLI      (gemini extension)
[ ]  5)       OpenCode        (opencode.ai)
[ ]  6)       OpenClaw        (~/.openclaw)
[x]  7)  [*]  Cursor          (.cursor/rules)
[ ]  8)       Aider           (CONVENTIONS.md)
[ ]  9)       Windsurf        (.windsurfrules)
[ ] 10)       Qwen Code       (~/.qwen/agents)

[1-10] toggle   [a] all   [n] none   [d] detected
[Enter] install   [q] quit
```

시스템에 설치된 도구를 자동 감지(`[*]`)하고 체크박스 TUI로 선택 설치한다. `--parallel` 플래그로 병렬 변환/설치도 지원.

### 4.3 다중 에이전트 협업 예시

README는 8개 에이전트가 동시에 작업하는 실제 예시("Nexus Spatial Discovery Exercise")를 제공한다:

- Product Trend Researcher (시장 검증)
- Backend Architect (기술 아키텍처)
- Brand Guardian (브랜드 전략)
- Growth Hacker (GTM)
- Support Responder (지원 시스템)
- UX Researcher (UX 연구)
- Project Shepherd (프로젝트 실행)
- XR Interface Architect (공간 UI)

결과: 단일 세션에서 크로스-기능적 제품 청사진 완성.

### 4.4 업스트림-파생 관계

README에 명시:
> "awesome-openclaw-agents -- Community-maintained OpenClaw agent collection (derived from this repo)"

| 항목 | agency-agents (원본) | awesome-openclaw-agents (파생) |
|------|---------------------|------------------------------|
| Stars | 59,300+ | 미확인 |
| 포맷 | 멀티-툴 `.md` | OpenClaw SOUL.md 특화 |
| OpenClaw 변환 | SOUL.md + AGENTS.md + IDENTITY.md | SOUL.md만 |
| 설치 자동화 | convert.sh + install.sh | 없음 |
| 에이전트 수 | 60개+ | 174개 (확장된 포크) |

파생 포크가 원본보다 에이전트 수가 많다는 점이 흥미롭다 -- mergisi가 OpenClaw 특화 에이전트를 적극적으로 추가했음을 시사.

---

## 5. 신규 패턴 R-번호

### **R34: Multi-Platform Agent Persona Transpiler**

기존 R1-R33에 없는 고유 패턴:

**R34: Multi-Platform Agent Persona Transpiler** -- 단일 표준 `.md` 에이전트 정의에서 10개 AI 도구별 포맷으로 자동 트랜스파일.

```
canonical .md
    |
    v  convert.sh
    |
    ├── Claude Code: .md (direct copy)
    ├── OpenClaw: SOUL.md + AGENTS.md + IDENTITY.md
    ├── Antigravity: SKILL.md
    ├── Cursor: .mdc
    ├── Aider: CONVENTIONS.md (단일 집계)
    ├── Windsurf: .windsurfrules (단일 집계)
    └── Qwen Code: SubAgent .md
```

**원리**: "Write Once, Deploy Everywhere" 패턴을 에이전트 페르소나에 적용. 각 도구의 포맷 차이(단일 파일 vs 다중 파일, 단순 복사 vs 포맷 변환)를 스크립트 레이어에서 추상화.

**구현**: `scripts/convert.sh` (Bash). 병렬 실행 옵션(`--parallel`, 코어 수 자동 감지).

**시사점**: 에이전트 런타임들이 각자 고유 포맷(SOUL.md, SKILL.md, .mdc 등)을 가지면서 생기는 생태계 분절을 클라이언트 측 빌드 도구로 해결. 런타임 수가 늘어날수록 이런 표준화 계층의 중요성이 증가한다.

**기존 비교**: 13개 Claw 프레임워크 중 타 도구 포맷 변환 기능을 제공하는 런타임은 없다. 모두 자신의 포맷만 정의하는 단방향 구조.

---

## 6. 비교 테이블

| 항목 | agency-agents (원본) | awesome-openclaw-agents (파생) | awesome-openclaw-usecases |
|------|---------------------|------------------------------|--------------------------|
| **유형** | 멀티-툴 에이전트 페르소나 컬렉션 | OpenClaw 특화 SOUL.md 컬렉션 | 활용 사례 큐레이션 |
| **Stars** | 59,300+ | 미확인 | 미확인 |
| **에이전트 수** | 60개+ | 174개 | 해당 없음 |
| **OpenClaw 지원** | SOUL.md + AGENTS.md + IDENTITY.md | SOUL.md | 해당 없음 |
| **설치 자동화** | convert.sh + install.sh (TUI) | 없음 | 없음 |
| **다중 도구 지원** | 10개 | 1개 | 해당 없음 |
| **다중 에이전트 예시** | Nexus 8-agent 예시 포함 | crewclaw.com 원클릭 | 없음 |
| **번역** | zh-CN 2개 포크 | 없음 | 없음 |
| **신규 패턴** | R34 | 없음 | R31 |

---

## 7. 한계

- **에이전트 품질 검증 없음**: 컬렉션이 커질수록 개별 에이전트의 실제 효과 검증이 어려움. PR 기반 수동 리뷰만 존재.
- **버전 동기화 문제**: 파생 포크(awesome-openclaw-agents)가 독립적으로 에이전트를 추가하면서 원본과 내용 다이버전스 발생.
- **도구별 포맷 차이 완전 추상화 불가**: Aider/Windsurf는 모든 에이전트를 단일 파일로 병합 -> 에이전트 간 충돌 가능성.
- **에이전트 간 상태 공유 없음**: 8-agent 협업 예시를 제공하지만, 실제 에이전트 간 상태 전달 메커니즘(R31 STATE.yaml 등)은 없다 -- 사용자가 수동으로 컨텍스트를 전달해야 함.
- **OpenClaw 3파일 포맷 버전 불명확**: SOUL.md + AGENTS.md + IDENTITY.md 분리가 어떤 OpenClaw 버전을 기준으로 하는지 문서화 부족.
- **중국 시장 특화 에이전트 증가**: Kuaishou, Douyin, Taobao, Pinduoduo 플랫폼 에이전트가 많아지면서 글로벌 범용성 vs 지역 특화 간 균형 문제.

---

## 8. 참고 링크

- [GitHub](https://github.com/msitarzewski/agency-agents)
- [파생 포크: awesome-openclaw-agents](https://github.com/mergisi/awesome-openclaw-agents)
- [zh-CN 번역 1](https://github.com/jnMetaCode/agency-agents-zh)
- [zh-CN 번역 2](https://github.com/dsclca12/agent-teams)
- [details/awesome_openclaw_agents_report.md](awesome_openclaw_agents_report.md) -- 파생 포크 分析
- [../usecases_index.md](../usecases_index.md) -- usecases/ 교차 分析
- [../../repos/details/hermes_agent_report.md](../../repos/details/hermes_agent_report.md) -- agentskills.io + Skills Trust 4단계
- [../../repos_applied/repos_applied_report.md](../../repos_applied/repos_applied_report.md) -- 응용 계층 分析
