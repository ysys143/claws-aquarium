# Claws Aquarium: AI 에이전트 런타임 프레임워크 비교 분석

AI 에이전트 런타임 프레임워크 14개를 종합 분석하는 프로젝트입니다. 각 프레임워크의 아키텍처, 보안, 기억 관리, 브라우저 자동화 전략을 심층 비교하며, 커뮤니티 유즈케이스·밋업 실전 운영 패턴·에이전트 결제·신원 프로토콜 분석도 포함합니다.

---

## 배경: Claw 생태계란?

### Claw의 정의

OpenClaw의 성공 이후 "Claw"를 포함한 이름의 AI 에이전트 런타임이 폭발적으로 증가하고 있습니다. 이들은 공통적으로 다음 특징을 가집니다:

- **심화된 LLM 계획 + 도구 호출**: Claude/Codex 같은 DeepAgent의 planning(task 분해) + tool calling(파일 읽기/쓰기) 기능
- **24시간 상주형 아키텍처**: 메신저 인터페이스(Telegram 등)를 통해 언제 어디서나 접근 가능
- **진보적 지식 공개(Progressive Disclosure)**: Skill을 통해 절차적 지식을 상황에 맞게 동적 로딩
- **실세계 권한 위임**: 에이전트에게 파일 관리, 웹 조회, 시스템 명령 실행 권한 부여

### 핵심 문제

단순 도구 호출(read/write file)을 넘어, 다음을 해결해야 합니다:

1. **세션/컨텍스트 분리**: 메일 읽기, 첨부파일 확인, 기존 일정 조회, 주간 계획 수립 등 이종 작업을 개별 컨텍스트로 분리하면서 전체 의도 유지
2. **기억 축적**: 세션을 넘어 중기/장기 기억을 벡터/검색 가능한 형태로 유지
3. **실세계 권한의 안전한 위임**: 자격증명 보호, 프롬프트 인젝션 방어, 오버리치 방지
4. **브라우저 자동화와 도구 통합**: 웹 상호작용 및 외부 시스템 연결

---

## 분석 대상

### 14개 AI 에이전트 런타임 프레임워크

| # | 이름 | 언어 | 저장소 | LOC | 라이선스 | 핵심 철학 |
|---|------|------|--------|-----|---------|----------|
| 1 | **OpenClaw** | TypeScript | openclaw/openclaw | — | — | 풀스택 범용 AI 비서; 12+ 메시징 채널, 50+ 브라우저 기능 |
| 2 | **Nanobot** | Python | HKUDS/nanobot | ~4,000 | — | 연구용 초경량 에이전트, MCP 지원 |
| 3 | **NanoClaw** | TypeScript | qwibitai/nanoclaw | ~500 | — | 500줄 코어, 8분만에 이해 가능, 에이전트 스웜 지원 |
| 4 | **IronClaw** | Rust | nearai/ironclaw | — | — | 보안 우선, WASM 샌드박스, Zero-Exposure 크레덴셜 |
| 5 | **ZeroClaw** | Rust | zeroclaw-labs/zeroclaw | — | — | 극한 경량화: 5MB RAM/10ms 시작, 무손실 성능 |
| 6 | **PicoClaw** | Go | sipeed/picoclaw | — | — | 10MB RAM/1초 부팅, 구형 안드로이드 폰 지원, 95% AI 생성 |
| 7 | **TinyClaw** | TypeScript | TinyAGI/tinyclaw | — | — | 멀티에이전트 팀, 체인 실행 오케스트레이션, 실시간 TUI 대시보드 |
| 8 | **OpenFang** | Rust | RightNow-AI/openfang | — | — | Agent OS: 24/7 자율 실행, 7 Hands, 40 채널, 16-layer 보안, ~32MB 단일 바이너리 |
| 9 | **OpenJarvis** | TypeScript | open-jarvis/OpenJarvis | — | — | 멀티채널 자율 에이전트, A2A 지원, 유연한 메모리 아키텍처 |
| 10 | **NemoClaw** (NVIDIA) | JavaScript/TypeScript/Python/Shell | NVIDIA/NemoClaw | 25,650 | Apache 2.0 | OpenClaw Sandbox Plugin: GPU-최적화 샌드박스 에이전트 런타임, 4-layer 보안 |
| 11 | **NullClaw** | Zig | nullclaw/nullclaw | ~249,000 | MIT | 678 KB 정적 바이너리, <2ms 시작, ~1MB RAM; WASI·Landlock·19채널·10 메모리 엔진 |
| 12 | **Hermes Agent** (Nous Research) | Python | NousResearch/hermes-agent | — | MIT | 자기개선 에이전트; Frozen Snapshot 메모리, Skills Trust 4단계, Tirith pre-exec 스캐너 |
| 13 | **GoClaw** | Go 1.26 | nextlevelbuilder/goclaw | 176K | CC BY-NC 4.0 | 멀티테넌트 PostgreSQL 게이트웨이; Docker 3축 샌드박스, AES 암호화, Tailscale VPN, OTel 내장 |
| 14 | **CoPaw** | Python | agentscope-ai/CoPaw | 84K | Apache-2.0 | 14채널 개인 비서(생태계 최다 채널); 3-tuple QueueKey 격리, Playwright 브라우저, Skills 스캐닝 |

### 응용 프로젝트 (repos_applied/) — 주요 분석 대상

Claw 프레임워크 위에 구축된 응용 프로젝트 중 심층 분석한 항목:

| # | 이름 | Stars | 언어 | 저장소 | 핵심 특징 | 보고서 |
|---|------|-------|------|--------|----------|--------|
| 1 | **MetaClaw** | 2,700+ | Python | aiming-lab/MetaClaw | 7개+ Claw 백엔드(OpenClaw·CoPaw·IronClaw·PicoClaw·ZeroClaw·NanoClaw·NemoClaw·Hermes) 런타임 교체 가능한 메타 레이어; GRPO+LoRA 온라인 학습, MadMax 유휴-창 RL 스케줄링(R36), 실패 기반 스킬 진화+MAML 버퍼 분리(R37), 9-metric SQLite 메모리 승격 게이트 | [상세 보고서](reports/repos_applied/details/metaclaw_report.md) |

---

### 연구 자동화 도구 (repos_research/)

기존 에이전트 런타임과 별도로, **연구 자동화 특화 도구** 2개를 분석한다. ai-research-agent-design.md(AI Research Agent), lab-ai-agent-design.md(Lab AI Agent) 설계에 직접 참조된다.

| # | 이름 | 언어 | 저장소 | 핵심 철학 |
|---|------|------|--------|----------|
| 1 | **DeepInnovator** | Python | HKUDS/DeepInnovator | RL(GRPO) 기반 연구 아이디어 생성; 7개 에이전트 파이프라인, Authenticity Discriminator, MCP+Sandbox 도구 |
| 2 | **Autoresearch** | Python | karpathy/autoresearch | 자율 ML 실험 반복; 5분 고정 예산, program.md 에이전트 지시, keep/discard 자동 의사결정 |

---

## 생산된 보고서 (12개+)

### 0. reports/repos_research/research_tools_report.md (종합)
**주제**: 연구 자동화 도구 분석 (DeepInnovator & Autoresearch)

**핵심 발견**:
- **DeepInnovator**: 7개 전문 에이전트의 계층적 파이프라인 (Paper Analyzer, Router, Idea Spark, Serendipity Engine 등). GRPO + Delta Reward로 아이디어 개선도를 강화학습. Authenticity Discriminator로 생성 아이디어 진정성 검증 (기존 9개 런타임에 없는 신규 패턴).
- **Autoresearch**: 3파일 ~1K LOC의 극단적 단순성. program.md가 에이전트 지시서 (SKILL.md/HAND.toml과 동급이나 무한 루프+자동 평가+Git 상태 관리 고유 특성). prepare.py(읽기전용) + train.py(수정가능) 분리 = IronClaw capability attenuation 패턴.
- **Claw 패턴 매핑**: Memory(ZeroClaw Snapshot), Tool(MCP 표준), Security(IronClaw Capability), Cost(ZeroClaw 한도) 패턴이 연구 도구에 직접 재사용 가능. 신규 패턴 5가지 (R1~R5) 추출.
- **idea3/idea4 시사점**: "Research Instruction Document" 제안 (program.md 루프 + HAND.toml 도구/권한 + ZeroClaw 기억 스냅샷 결합).

**활용**: ai-research-agent-design.md(AI Research Agent), lab-ai-agent-design.md(Lab AI Agent) 설계 참조, 연구 자동화 패턴 재사용

### 1. reports/repos/details/openfang_report.md (개별)
**주제**: OpenFang Agent OS 심층 분석 (9번째 비교 대상)

**핵심 발견**:
- **Hands System**: 7개 번들 자율 능력 패키지 (Clip/Lead/Collector/Predictor/Researcher/Twitter/Browser). activate/pause/resume/deactivate 생명주기. 기존 8개 프레임워크에 없는 신규 추상화.
- **Channel Adapters 40개**: 기존 8개 프레임워크 합계 초과. 채널별 모델 오버라이드, 슬라이딩 윈도우 레이트 리밋. Twitter/X 어댑터는 미구현.
- **16-Layer 보안**: WASM Dual Metering(fuel+epoch), 5-Label Taint Tracking, 18-Type Capability, Ed25519 Manifest Signing. 단, 승인 게이트는 LLM 프롬프트 기반(암호학적 강제 아님).
- **Memory Phase 1**: SQLite + 코사인 유사도 BLOB. 외부 벡터 DB 없음. Knowledge Graph(entity-relation)는 기존 8개 중 유일.
- **3-Layer Context**: 결과당 30% / Guard 75% / Emergency 4단계. 기존 프레임워크 중 가장 정교.
- **A2A Protocol**: Google 스펙 완전 구현. 기존 8개 중 없음.

**활용**: OpenFang 포지셔닝 이해, Hands 아키텍처 설계 참고

### 1-b. reports/repos/details/nemoclaw_report.md (개별)
**주제**: NemoClaw (NVIDIA) 소스코드 심층 분석 (11번째 비교 대상)

**핵심 발견**:
- **4-Layer 보안**: GPU 격리 샌드박스, WASM 실행 경계, 크레덴셜 프록시, 감사 로그. 보안 특화 설계.
- **GPU-최적화 런타임**: OpenClaw Sandbox Plugin으로 동작. GPU 가속 추론·배치 처리 네이티브 지원.
- **제한된 독립성**: 장기 기억·브라우저·채널은 OpenClaw에 위임. 단독 스탠드얼론 운용 불가.
- **25,650 LOC (Apache 2.0)**: JavaScript/TypeScript/Python/Shell 혼합. 플러그인 아키텍처로 기능 범위 명확히 한정.

**활용**: GPU 워크로드 에이전트 설계, 보안 샌드박스 패턴 참고

### 1-c. reports/repos/details/hermes_agent_report.md (개별)
**주제**: Hermes Agent (Nous Research) 심층 분석 (12번째 비교 대상)

**핵심 발견**:
- **Frozen Snapshot 메모리** (R17): 세션 시작 시 MEMORY.md+USER.md 스냅샷 1회 캡처. 세션 중 불변 → Anthropic prefix cache 전체 세션 유지. 문자 단위 예산(2,200+1,375 chars) — 모델 독립적.
- **Skills Trust 4단계** (R20): agentskills.io 오픈 스탠다드. builtin/trusted/community/agent-created 정책 + skills_guard.py 정적 분석(6 카테고리).
- **Tirith Pre-Exec Scanner** (R22): 명령 실행 전 외부 바이너리 스캔 (SHA-256+cosign 서명). Memory Injection Detection(R19): 10 regex 패턴 + 비가시 유니코드 탐지.
- **Bounded Delegation Tree** (R21): MAX_DEPTH=2, MAX_CONCURRENT=3. 자식은 memory 쓰기/재귀 위임 차단.
- **보안 Tier 2+**: Tirith 외부 바이너리 스캐너 + Memory Injection Detection + Skills Trust 4단계. Claw 생태계 직접 연결 (topics: openclaw, clawdbot, moltbook).

**활용**: 자기개선 에이전트 설계, Skills 오픈 스탠다드 채택, 메모리 인젝션 방어 참고

### 1-e. reports/repos/details/nullclaw_report.md (개별)
**주제**: NullClaw (Zig) 심층 분석 (11번째 비교 대상)

**핵심 발견**:
- **생태계 최소 바이너리**: 678 KB 정적 바이너리 — ZeroClaw(8.8 MB)보다 13배 작음. libc 외 의존성 없음 → R15 공급망 공격 표면 제거.
- **WASI 타겟**: `main_wasi.zig` — 13개 프레임워크 중 유일한 WebAssembly 배포 옵션 (R16 에이전트 이식성 표준).
- **보안 Tier 1**: ChaCha20-Poly1305 + Landlock OS 샌드박스 (커널 직접 파일/네트워크 제어). ZeroClaw와 동급.
- **19개 채널**: Signal + Nostr + Matrix 지원 — OpenFang(40) 다음 생태계 2위권. Nostr/Matrix는 분석된 프레임워크 중 유일한 탈중앙화 채널.
- **10개 메모리 엔진**: SQLite/PostgreSQL/Redis/ClickHouse + 벡터/하이브리드. vtable 인터페이스로 코드 변경 없이 교체 가능.
- **A2A 프로토콜**: `a2a.zig` — OpenFang, OpenJarvis에 이어 3번째 지원.
- **6,400 Stars, 5,300+ 테스트**: 성숙도 높은 Zig 에코시스템.

**활용**: 엣지/IoT 배포, 공급망 보안 강화, WASM 에이전트 이식성 설계 참고

### 1-f. reports/repos/details/claude_code_report.md (개별)
**주제**: Claude Code (Anthropic) 심층 분석 (13번째 비교 대상)

**핵심 발견**:
- **MCP-as-Channel Bridge** (R23): MCP notification 단일 메서드(`notifications/claude/channel`)로 모든 외부 메시징 플랫폼을 에이전트 세션에 주입. 어떤 MCP 서버도 잠재적 채널이 됨.
- **Platform-Controlled Allowlist** (R24): Anthropic이 `tengu_harbor_ledger` 피처 플래그로 허용 채널 원격 통제. 모든 셀프호스팅 Claw와 근본적으로 다른 신뢰 모델.
- **Content/Meta 분리** (R25): 채널 메시지 메타데이터(`meta` 필드)와 콘텐츠(`content`) 구조적 분리 → 발신자 텍스트로 시스템 메타데이터 위조 차단.
- **Bundled OS-Level Sandbox** (R26): npm vendor에 seccomp BPF 필터+바이너리 동봉. npm install만으로 OS 레벨 샌드박스 완결. Linux: seccomp+bwrap, macOS: Sandbox.framework → **Tier A+**.
- **Discord 멀티에이전트 버스** (R33, 별도 보고서): Discord 채널을 오케스트레이터 없는 에이전트 간 통신 버스로 활용. bot-to-bot 메시지 1줄 변경으로 활성화.

**활용**: 채널 보안 설계, MCP 채널 통합 아키텍처, OS 레벨 샌드박스 참고

### 1-g. reports/commercial/details/nvidia_nemoclaw_commercial.md (상용화)
**주제**: NemoClaw 상용화 전략 분석

**내용**:
- NVIDIA 생태계 내 포지셔닝 및 엔터프라이즈 타깃
- GPU 클라우드 서비스(NIM)와의 통합 전략
- 경쟁 프레임워크 대비 차별화 포인트

**활용**: NemoClaw 상용 도입 검토, GPU 에이전트 시장 분석

### 2. reports/repos/framework_catalog.md (종합) ~~ecosystem_overview.md~~
**주제**: 12대 프레임워크의 기본 정보, 특징, 배포 방식 종합 정리 (OpenFang·OpenJarvis·NemoClaw 포함)

**내용**:
- 각 프레임워크의 개발 철학, 기술 스택, 핵심 기능
- 클라우드 배포 변형체 (serverless-openclaw, OpenClaw+Bedrock, KimiClaw 등)
- 종합 비교표 및 선택 기준

**활용**: 프레임워크 선택 의사결정, 기본 리서치

### 3. reports/repos/session_context_report.md (종합)
**주제**: 24시간 상주 에이전트의 세션/컨텍스트 관리 전략

**핵심 발견**:
- **3가지 아키타입으로 수렴**:
  1. 프로세스/컨테이너 격리 (NanoClaw, TinyClaw): OS 경계 = 컨텍스트 경계
  2. 세션 키 기반 논리적 격리 (OpenClaw, Nanobot, PicoClaw, ZeroClaw): 같은 프로세스 안에서 ID 분리
  3. 보안 계층 기반 격리 (IronClaw): WASM+프록시+볼트
- 모든 구현체가 단발성 세션 단일 스레드 기본
- **아직 아무도 풀지 못한 것**: "이 작업은 별도 컨텍스트가 필요하다"는 판단을 프레임워크 레벨에서 자동화한 곳 없음

### 4. reports/repos/security_report.md (종합)
**주제**: 에이전트의 실세계 권한 부여와 보안 전략

**핵심 발견**:
- **4단계 보안 성숙도**:
  - **Tier 1** (IronClaw, ZeroClaw): 암호화 볼트 + 이중 샌드박스 + 다층 인젝션 방어 + HITL + 비용 제한
  - **Tier 2** (NanoClaw, OpenClaw): Docker 격리 + 도구 허용목록 + 부분 인젝션 방어
  - **Tier 3** (Nanobot, PicoClaw): 정규식 차단 + 파일 제한 + 평문 자격증명
  - **Tier 4** (TinyClaw): 최소/없음 (실험적/특수용)
- 암호화 볼트 구현: 2개만 (IronClaw: AES-256-GCM, ZeroClaw: ChaCha20-Poly1305)
- Human-in-the-loop 구현: 2개만 (IronClaw, ZeroClaw)
- 프롬프트 인젝션 전용 방어: 2개만 (IronClaw SafetyLayer, ZeroClaw PromptGuard)

### 5. reports/repos/browser_actions_report.md & reports/repos/memory_architecture_report.md (종합)
**주제**: 브라우저 자동화/도구 아키텍처 및 기억 시스템 설계

**브라우저 자동화 현황**:
- 4개만 보유: OpenClaw(50+ 기능), ZeroClaw(3 백엔드), NanoClaw(X/Twitter 전용), IronClaw(E2E 테스트만)
- 도구 아키텍처 5가지: Rust Trait+WASM, Go Interface+병렬실행, TS Plugin+훅, Python ABC+레지스트리, SKILL.md+IPC

**기억 성숙도**:
- **Tier 1** (IronClaw, OpenClaw, ZeroClaw): 벡터+FTS+하이브리드 검색
- **Tier 2** (Nanobot, PicoClaw): 파일 기반 + LLM 정리
- **Tier 3** (NanoClaw, TinyClaw): 없음/위임

### 6. reports/meetup/meetup_patterns_report.md (실전 운영)
**주제**: OpenClaw 밋업 발표자들의 실전 운영 패턴 (2026년 3월)

**핵심 발견**:
- **2-Step Cron 최적화** (김동규): 경량 조건 체크(JS/Bash) → 조건 충족 시에만 에이전트 스폰. 15분 주기 96회 체크에서 실제 LLM 호출을 5~10회로 감소 (90%+ 토큰 절감).
- **Subagent Context 격리 패턴** (윤주운/Sionic AI): 서브에이전트 부모 컨텍스트 상속 문제 → 독립 에이전트 + Mattermost 채널 연결로 도메인별 메모리 격리. FLOCK 시스템 (Mac Mini + RTX 5090×2 + RTX 3090×2 + NAS + Tailscale VPN).
- **NanoClaw X 피드 + LangSmith 통합** (김동규): X API → NanoClaw → LangSmith 트레이싱. 에이전트 실행 추적의 실전 패턴.
- **기타**: 퀀트 트레이딩 에이전트, 음성 에이전트, 법률 문서 처리 등 8개 발표에서 도출된 다양한 운영 패턴.

**활용**: 토큰 비용 최적화, 멀티 에이전트 컨텍스트 격리 설계, 홈랩 구성 참고

### 7-a. reports/usecases/usecases_index.md (커뮤니티 콘텐츠)
**주제**: OpenClaw 생태계 커뮤니티 콘텐츠 4종 종합 분석

**핵심 발견**:
- **agency-agents** (59.3k Stars, msitarzewski): 60개+ 에이전트 페르소나, 8 Division 체계. convert.sh로 Claude Code/OpenClaw/Cursor/Aider 등 10개 AI 도구 포맷 자동 트랜스파일 → **R34 Multi-Platform Agent Persona Transpiler**. awesome-openclaw-agents의 업스트림 소스.
- **awesome-openclaw-agents** (mergisi): 174개 SOUL.md 템플릿, 24 카테고리 + 132 유즈케이스. crewclaw.com 통합. agency-agents의 파생 포크.
- **awesome-openclaw-usecases** (hesamsheikh): 40개 유즈케이스, 6 카테고리. STATE.yaml 공유 파일 패턴 → **R31 Shared-State File Coordination** (중앙 오케스트레이터 없는 분산 멀티에이전트 조율).
- **서울 OpenClaw 밋업 0315** (Instruct.KR): 950 RSVP, 16세션. TypeScript-as-Tool 동적 실행(R29), 포인터맵 메모리(R30), Bot-to-Bot Discord 버스(R33) 발굴.

**활용**: 에이전트 페르소나 설계, 멀티플랫폼 배포 전략, 커뮤니티 생태계 현황 파악

### 8. reports/meetup/agent_payment_protocol_report.md (결제·신원·상호운용성)
**주제**: 에이전트 네이티브 결제·신원·상호운용성 프로토콜 비교 (2026년 3월 밋업)

**핵심 발견**:
- **X402** (Logan Kang/BASE Korea): HTTP 402 기반 블록체인 지갑 서명 결제 미들웨어. 인간 개입 없는 에이전트간 자동 정산. UCP → AP2 → X402 레이어 스택.
- **Virtuals ACP** (Logan Kang): 에이전트-to-에이전트 서비스 마켓플레이스 — 서비스 발견 + 협상 + 에스크로 + 평가 4단계 전 주기.
- **ERC-8004** (김서준): 에이전트 신원 표준. Registry + Reputation + Validation, Know Your Agent(KYD). 표준화 진행 중.
- **AIP** (Zoe/Unibase): 에이전트 간 협력 프로토콜. A2A 위에서 작동하는 상위 레이어.
- **기존 결제 방식의 근본 문제**: API 키(인증만), 구독(분배 불가), PG(인간 승인 필요) → 머신 네이티브 페이먼트의 필요성.

**활용**: 에이전트 경제 설계, A2A + 결제 프로토콜 통합 아키텍처 참고

---

## 디렉토리 구조

```
/claws-aquarium/
├── README.md                          # 본 파일
├── .ideas/                            # 설계 아이디어 및 계획
│   ├── claw-alternatives-catalogue.md # Claw 생태계 배경 및 핵심 구현체
│   ├── investigation-directives.md    # 조사 지시, 7가지 논의, 열린 질문
│   ├── ai-research-agent-design.md    # AI Research Agent 아키텍처
│   └── managed-claw-services.md       # 24/7 Claw 매니지드 서비스 카탈로그
├── reports/
│   ├── repos/                         # 프레임워크 분석
│   │   ├── framework_cross_analysis.md  # 종합 교차 분석
│   │   ├── framework_catalog.md         # 프레임워크 카탈로그
│   │   ├── *.md                         # 종합 비교: 보안, 기억, 세션, 브라우저
│   │   └── details/                     # 개별 심층
│   │       ├── openfang_report.md       # OpenFang Agent OS 분석
│   │       ├── openjarvis_report.md     # OpenJarvis 분석
│   │       ├── nemoclaw_report.md       # NemoClaw 심층 분석
│   │       ├── nullclaw_report.md       # NullClaw (Zig) 심층 분석
│   │       ├── hermes_agent_report.md   # Hermes Agent (Nous Research) 심층 분석
│   │       ├── claude_code_report.md    # Claude Code (Anthropic) 심층 분석
│   │       └── claude_code_multiagent_discord_report.md  # Claude Code Discord 멀티에이전트
│   ├── repos_applied/                 # repos_applied/ 대상 분석
│   │   ├── *.md                       # 종합 비교: 응용 계층
│   │   └── details/                   # 개별 심층
│   │       ├── autoresearch_skill_report.md  # autoresearch-skill 분석
│   │       ├── miclaw_report.md             # MiClaw (Xiaomi) 분석
│   │       └── metaclaw_report.md           # MetaClaw (aiming-lab) 분석
│   ├── repos_research/                # repos_research/ 대상 분석
│   │   ├── *.md                       # 종합 비교: DeepInnovator & Autoresearch
│   │   └── details/                   # 개별 심층: OpenClaw-RL
│   ├── usecases/                      # 커뮤니티 콘텐츠 분석
│   │   ├── usecases_index.md          # 전체 인덱스
│   │   └── details/                   # 개별 심층
│   │       ├── agency_agents_report.md
│   │       ├── awesome_openclaw_agents_report.md
│   │       ├── awesome_openclaw_usecases_report.md
│   │       └── openclaw_seoul_meetup_0315_report.md
│   ├── meetup/                        # 밋업 실전 운영 패턴
│   │   ├── meetup_patterns_report.md
│   │   └── agent_payment_protocol_report.md
│   ├── commercial/                    # 상용 서비스 분석
│   │   ├── commercialization_strategy.md  # 종합 비교표 (8개 서비스)
│   │   └── details/
│   │       ├── tencent_*.md, xiaomi_miclaw_analysis.md
│   │       ├── nvidia_nemoclaw_commercial.md
│   │       ├── crew_you_analysis.md
│   │       ├── genspark_claw_analysis.md
│   │       └── perplexity_personal_computer_analysis.md
│   └── deployment/                    # 배포 전략 비교 (4)
├── repos/                             # 15개 프레임워크 소스코드 (agentscope/는 CoPaw 분석용 참조)
│   ├── openclaw/, nanobot/, nanoclaw/ # OpenClaw, Nanobot, NanoClaw
│   ├── ironclaw/, zeroclaw/, picoclaw/ # IronClaw, ZeroClaw, PicoClaw
│   ├── tinyclaw/, openfang/, openjarvis/ # TinyClaw, OpenFang, OpenJarvis
│   ├── nemoclaw/                      # NemoClaw (25,650 LOC, Apache 2.0)
│   ├── nullclaw/                      # NullClaw (~249K LOC, MIT)
│   ├── hermes-agent/                  # Hermes Agent (Nous Research, MIT)
│   ├── cc_2.1.80/                     # Claude Code v2.1.80 (npm bundle)
│   ├── goclaw/                        # GoClaw (~176K LOC, CC BY-NC 4.0)
│   ├── agentscope/                    # AgentScope (CoPaw 기반 플랫폼, 분석용 참조)
│   └── copaw/                         # CoPaw (~84K LOC, Apache 2.0)
├── repos_applied/
│   ├── autoresearch-skill/            # Karpathy autoresearch → Claude Code 스킬 포팅
│   ├── metaclaw/                      # MetaClaw (aiming-lab, 2.7k stars)
│   └── (기타 applied 저장소)
├── repos_research/
│   ├── deepinnovator/                 # HKUDS/DeepInnovator (~105K LOC)
│   └── autoresearch/                  # karpathy/autoresearch (~1K LOC)
├── usecases/
│   ├── agency-agents/                 # msitarzewski/agency-agents (59.3k Stars)
│   ├── awesome-openclaw-agents/       # mergisi/awesome-openclaw-agents
│   ├── awesome-openclaw-usecases/     # hesamsheikh/awesome-openclaw-usecases
│   └── openclaw_seoul_meetup_0315/    # Instruct.KR 서울 밋업 세션 기록
└── .cwf/sessions/
    └── (Claude 세션 기록)
```

---

## 빠른 탐색

| 구분 | 위치 | 종합 비교 (root) | 개별 심층 (details/) |
|------|------|------------------|---------------------|
| **프레임워크** | `reports/repos/` | framework_catalog, framework_cross_analysis, 보안, 기억, 세션, 브라우저 (6) | OpenFang, OpenJarvis, NemoClaw, NullClaw, **Hermes Agent, Claude Code, Claude Code Discord** (7) |
| **응용 프로젝트** | `reports/repos_applied/` | 응용 계층 (1) | **autoresearch-skill, MiClaw, MetaClaw** (3) |
| **연구 도구** | `reports/repos_research/` | DeepInnovator & Autoresearch (1) | OpenClaw-RL (2) |
| **커뮤니티 콘텐츠** | `reports/usecases/` | usecases_index (1) | **agency-agents, awesome-openclaw-agents, awesome-openclaw-usecases, 서울 밋업** (4) |
| **상용 서비스** | `reports/commercial/` | 상용화 전략 비교표 8개 서비스 (1) | Tencent, Xiaomi, NemoClaw, Crew.you, Genspark, Perplexity PC (9) |
| **배포 전략** | `reports/deployment/` | VPS, FaaS, 서비스 비교, Helm (4) | — |
| **밋업 패턴** | `reports/meetup/` | 실전 운영 패턴, 결제·신원 프로토콜 (2) | — |
| **설계/아이디어** | `.ideas/` | 4개 설계 아이디어 및 계획 문서 | — |

---

## 핵심 발견 요약

### 발견 1: 3가지 아키타입으로 수렴하는 세션 관리

모든 구현체가 다음 3가지 중 하나로 분류됩니다:
- **프로세스 격리**: 각 작업 = 독립 프로세스 (컨테이너/tmux)
- **논리적 격리**: 같은 프로세스, 세션 ID로 분리
- **보안 계층**: WASM/프록시로 추가 경계 강화

하지만 **아무도 자동 판단을 하지 못합니다.** "이 작업에 별도 컨텍스트가 필요하다"는 것을 LLM이 스스로 결정해야 합니다.

### 발견 2: 보안은 철학의 선택

보안 성숙도가 다양한 이유:
- IronClaw/ZeroClaw: 신뢰 불가능한 도구의 위험을 근본적으로 해결하려는 철학
- NanoClaw/OpenClaw: 컨테이너로 격리하고 도구 목록을 관리하는 실용주의
- Nanobot/PicoClaw: "위험한 패턴 차단"에만 의존
- TinyClaw: 보안은 사용자/배포자 책임

**어느 것이 "맞는가"가 아니라, 위험 모델이 다릅니다.**

### 발견 3: 기억 구현의 이중 주입 경로

Tier 1 구현체(IronClaw, OpenClaw, ZeroClaw)는 공통적으로:
1. MEMORY.md를 시스템 프롬프트에 **항상** 로드 (중기 기억)
2. 턴마다 DB/벡터 검색으로 동적 주입 (장기→중기 승격)

이 두 경로가 동시에 작동하는 "이중 주입" 패턴입니다.

**유일한 혁신**: ZeroClaw의 **Soul Snapshot** — brain.db → MEMORY_SNAPSHOT.md → Git 추적 → cold-boot 복원. DB 손실에도 자아가 살아남습니다.

### 발견 4: 브라우저 자동화는 소수의 전유물

7개 중 **4개만** 브라우저 자동화를 가집니다:
- **OpenClaw**: 50+ 기능, 엔터프라이즈급
- **ZeroClaw**: 3 백엔드 (CLI/Rust/computer_use) 교체 가능
- **NanoClaw**: X/Twitter 전용 최적화 (호스트에서 실행, 안티탐지)
- **IronClaw**: 테스트 전용

**핵심 관찰**: 범용 브라우저 vs 플랫폼 특화 자동화의 트레이드오프

### 발견 5: 도구 병렬 실행은 PicoClaw만

6개는 도구를 순차 실행합니다. **PicoClaw만 goroutine + WaitGroup으로 병렬 실행**을 지원합니다.

이는 "메일 읽기 + 일정 확인 + 계획 수립"과 같은 멀티스텝 작업에서 심각한 지연을 야기합니다.

---

## 열린 질문 (15개)

조사 과정에서 발견한, 아직 풀리지 않은 질문들입니다. 이들은 **Claw 프레임워크의 미래 진화 방향**을 가리킵니다.

### 아키텍처 수준 (Q1-Q4)

**Q1. 프로젝트 수명주기 관리자는 누가?**
- 오케스트레이터(메인 에이전트)가 직접 디렉토리를 만들고 서브에이전트를 스폰하는 게 맞는가?
- 아니면 이를 별도 레이어(프레임워크 수준)로 올려야 하는가?

**Q2. 컨텍스트 분리의 기준은?**
- "이 작업은 별도 컨텍스트가 필요하다"는 판단을 LLM에만 맡기면 일관성이 없다.
- 토큰 수, 작업 유형 등 명시적 규칙으로 강제할 수 있는가?

**Q3. Karpathy의 "실험 설계 못함" 문제를 해결할 수 있는가?**
- Karpathy의 연구 조직 분석: 에이전트는 잘 정의된 아이디어는 구현 잘하지만, 창의적 실험 설계를 못함.
- 실험 설계 절차를 skill로 만들어 매번 로딩하면 커버되는가, 아니면 근본 다른 접근 필요?

**Q4. 메신저 + takeover 결합은 어떻게?**
- Karpathy의 tmux takeover: 에이전트가 막혔을 때 사람이 개입 가능
- 텔레그램 기반 시스템에서 이를 구현하려면?

### 보안 수준 (Q5-Q8)

**Q5. 동적 도구 위험도 평가는 가능한가?**
- 새 MCP 도구 등록 시 위험도를 자동 분류하고 승인 수준 할당하는 메커니즘?
- IronClaw의 ApprovalRequirement는 수동 설정인데, 자동화할 수 있는가?

**Q6. 비용 하드 한도의 보편화가 필요한 이유**
- ZeroClaw만 일별 $5 한도 구현
- 24시간 에이전트에서 비용 폭발은 현실적 위험인데, 왜 다른 구현체는 무시?

**Q7. 프롬프트 인젝션의 적응형 방어는?**
- 현재: 알려진 패턴 탐지만 가능
- 미래: 새 공격 기법을 자동 탐지/대응할 수 있는가?

**Q8. E-Stop의 메신저 통합**
- ZeroClaw의 E-Stop (4단계 + OTP): 긴급 정지 안전장치
- 텔레그램 기반 시스템에 통합하면 어떤 형태가 되는가?

### 성능/기능 수준 (Q9-Q15)

**Q9. 도구 병렬 실행의 부재**
- PicoClaw 외 6개가 순차 실행
- 멀티스텝 작업에서 심각한 레이턴시 누적
- 프레임워크 수준에서 풀어야 하는가?

**Q10. 브라우저 보안과 기능의 트레이드오프**
- OpenClaw: 50+ 기능, SSRF 방어 (만능 도구)
- NanoClaw: 6개 X 액션, 안티탐지 (플랫폼 특화)
- 24시간 에이전트에는 어느 쪽이 더 적합한가?

**Q11. 기억 consolidation의 최적 주기는?**
- Nanobot: 100 메시지 당
- ZeroClaw: 12시간마다
- OpenClaw: 5초 debounce
- 24시간 에이전트에 맞는 리듬은?

**Q12. 벡터 검색 vs FTS의 실전 recall 비교**
- 메신저 대화 맥락에서 어느 쪽이 실제로 더 유용한 기억을 찾아오는가?
- 결론: 하이브리드 검색이 필수인가?

**Q13. Soul Snapshot의 "기억 롤백"은 실용적인가?**
- ZeroClaw: git 버전 관리로 특정 시점의 기억 상태로 복원
- 실제 24시간 운영 상황에서 유용한가?

**Q14. 기억 오염 방지의 근본적 해법은?**
- IronClaw의 하드코딩 보호: 특정 파일만 읽기 전용
- 나머지 기억은 LLM 판단에만 의존
- 더 나은 방법이 있는가?

**Q15. 멀티에이전트 환경에서의 기억 공유 경계는?**
- IronClaw: agent_id 칼럼으로 격리 (DB 공유)
- NanoClaw: 컨테이너 격리 (파일시스템 격리)
- 서브에이전트가 메인 에이전트의 기억에 접근해야 하는가?

---

## 7가지 주요 논의 결과

### 논의 1: "빠진 기능"이 아니라 "안 넣은 기능"

NanoClaw는 이미 `allowedTools`에 TeamCreate, SendMessage 등을 열어놓고 있습니다.
- **기술적으로 가능**: 모든 Claw가 서브에이전트 오케스트레이션 가능
- **선택의 문제**: 시스템 프롬프트에 "복잡한 작업 → TeamCreate"라는 skill을 넣으면 되는 것
- **핵심**: 프레임워크 코드의 문제가 아니라, "절차적 지식의 점진적 공개"라는 소프트웨어 설계의 문제

### 논의 2: Bash에서 Claude를 직접 호출하는 방법

기술적으로 완전히 가능합니다:
```bash
cd /some/working/dir
claude --headless --skip-permission \
  --prompt "Use TeamCreate to solve this..."
```
**트레이드오프**:
- 비용 폭발 (서브에이전트마다 풀 시스템 프롬프트)
- Anthropic API 종속성 심화
- 프로세스 제어권 상실

이는 "기술적 불가능"이 아니라 **비용과 제어권의 트레이드오프**입니다.

### 논의 3: Karpathy의 연구 조직 실험

Karpathy가 공개한 설정 분석:
- 8개 에이전트 (Claude 4개 + Codex 4개), GPU 1개씩
- Docker/VM 대신 **git worktree로 파일시스템 격리** (오버헤드 없음)
- 에이전트 간 통신을 **파일로** (가장 단순하면서도 안정적)
- **tmux 그리드가 대시보드이자 takeover 인터페이스**

**핵심 발견**: "에이전트는 잘 정의된 아이디어 구현은 잘하지만, 창의적 실험을 설계하지 못한다."
- 이는 프레임워크나 skill의 문제가 아니라 **현재 LLM의 근본적 능력 한계**
- 비전: "조직을 프로그래밍한다" — 소스코드 = 프롬프트 + 스킬 + 도구 + 프로세스(데일리 스탠드업 등)

### 논의 4: 대시보드와 모니터링

7개 중 **TinyClaw만** 대시보드를 가짐 (TUI + TinyOffice 웹 UI).
- Karpathy: tmux 자체가 대시보드 + takeover 인터페이스
- 나머지: 메신저나 CLI로만 결과 확인

**미해결 문제**: 24시간 돌아가는 에이전트에서 뭔가 잘못될 때 **사람이 어떻게 개입하는가?**

### 논의 5: 실세계 권한 부여의 4단계

앞서 정리한 보안 성숙도 분류. **핵심**: 각 단계는 철학적 선택이지, 기술적 의무가 아닙니다.

### 논의 6: 브라우저 자동화와 도구 아키텍처

4개만 브라우저 보유, 6가지 도구 정의 철학, MCP가 표준.

**혁신**:
- **NanoClaw의 호스트 실행 브라우저**: X/Twitter 탐지 우회를 위해 컨테이너가 아닌 호스트에서 실행, IPC 파일 폴링으로 통신
- **IronClaw의 Zero-Exposure 크레덴셜**: WASM 도구는 secret-exists()만 호출 가능, 도구 코드 버그에도 자격증명 노출 안 됨
- **PicoClaw의 도구 병렬 실행**: 유일한 구현체

### 논의 7: 기억 아키텍처와 자아 연속성

Tier 1 구현체의 이중 주입 경로, 보호된 파일 vs 쓰기 가능 파일 구분.

**혁신**: ZeroClaw의 Soul Snapshot
- brain.db → MEMORY_SNAPSHOT.md → Git → cold-boot 복원
- DB 손실에도 **자아(identity)가 살아남는** 유일한 구현

---

## 조사 방법론

모든 보고서는 다음 방식으로 작성되었습니다:

1. **병렬 분석**: 7개 에이전트 또는 scientist 팀이 각 레포를 동시에 심층 분석
2. **소스코드 중심**: 문서가 아닌 실제 구현 코드 검토
3. **패턴 추출**: 각 구현체의 고유 선택, 공통 패턴, 미해결 문제 분류
4. **교차 검증**: 보고서 간 발견의 일관성 확인, 상충하는 결론 해석

---

## 종합 평가

### 차원별 평가 매트릭스

4개 보고서(세션/보안/도구+브라우저/기억)의 분석 결과를 교차 종합한 평가입니다.

| 차원 | IronClaw | ZeroClaw | OpenClaw | NanoClaw | Nanobot | PicoClaw | TinyClaw | OpenFang | OpenJarvis | NemoClaw | NullClaw | Hermes | GoClaw | CoPaw |
|------|----------|----------|----------|----------|---------|----------|----------|----------|------------|----------|----------|--------|--------|-------|
| **도구 아키텍처** | A | A | S | B | B | B | D | A (60 built-in, MCP, A2A) | A | C (플러그인 전용) | A (35+ built-in, vtable, MCP, A2A) | B (Skills 4단계) | B+ (MCP+20+LLM+KG) | A (Skills+browser) |
| **보안** | S | S | B | B | C | C | D | A (16-layer, LLM gates) | B+ | A+ (4-layer, GPU 샌드박스) | S (ChaCha20+Landlock) | B+ (Tirith+Memory Injection+Skills Trust) | A+ (Docker 3축+AES+Tailscale) | B (tool_guard+skill_scanner) |
| **장기기억** | A | A | S | C | B | B | D | B (Phase 1 SQLite) | A | D (위임) | A (10 엔진, 벡터+하이브리드) | A (Frozen Snapshot+char 예산) | B (임베딩+PostgreSQL) | B (ReMe 통합) |
| **단기기억(세션)** | A | A | A | B | B | B | C | A (3-layer context) | B+ | C | A | A (prefix cache 보존) | B | B |
| **브라우저** | D | A | S | B | - | - | - | B (Native CDP, 11 tools) | B | D (위임) | D | D | B (go-rod) | A (Playwright 3,460줄) |
| **채널/메신저** | - | - | S | A | A | - | A | S (40 adapters) | A | C | A (19채널, Signal+Nostr+Matrix) | A (6종 메시징) | A (7채널) | S (14채널, 생태계 최다) |
| **자율 실행(24/7)** | B | B | B | B | B | B | B | S (Hands, scheduler) | A | A (GPU 스케줄링) | A (heartbeat.zig) | B | A (cron+scheduler) | A (apscheduler) |
| **MCP 지원** | A | A | A | B | A | A | - | A (bidirectional) | A | B | A | A | A | A (AgentScope 기반) |
| **멀티에이전트** | B | B | B | S | B | - | A | A (A2A, agent tools) | A | C | A (subagent.zig + A2A) | B (Bounded Tree, depth=2) | A (Teams+blocked_by DAG) | B |
| **배포 용이성** | C | B | A | A | A | B | B | A (single binary) | A | B | S (678KB 정적 바이너리) | B | S (단일 바이너리+Docker Compose) | B (pip+Docker) |
| **고유 혁신** | WASM Zero-Exposure | Soul Snapshot | Plugin 24 hooks | Agent Swarm | 최소 코드 | 병렬 도구 | TUI 대시 | Hands+40채널 | 유연한 A2A | GPU 최적화 샌드박스 | WASI+Landlock+공급망 표면 최소화 | Frozen Snapshot+Tirith Pre-Exec | 3축 Docker 샌드박스+Tailscale VPN | 14채널+3-tuple QueueKey 격리 |

S=최고, A=우수, B+=우수에 가까운 보통, B=보통, C=약함, D=없음/최소, -=해당없음

### 기능 완성도 vs 보안 종합 비교

기능과 보안을 분리해서 보면 프레임워크별 포지션이 더 명확하게 드러납니다.

| 프레임워크 | 언어 | 기능 완성도 | 채널 | 보안 | 핵심 포지션 |
|-----------|------|------------|------|------|------------|
| **OpenClaw** | TS | **S** | 12+ | B | 생태계 기준점. ClawHub·Lobster·Canvas·음성·WebSocket 전부 프로덕션. 나머지 전부 여기에 대한 반응 |
| **Nanobot** | Python | B | 9 | C | OpenClaw가 너무 복잡하다 → 4K LOC로 재구현 |
| **NanoClaw** | TS | B | — | B | OpenClaw가 너무 위험하다 → 컨테이너 격리+스웜 |
| **IronClaw** | Rust | B | — | **S** | OpenClaw가 너무 불안하다 → 보안만 파고든 Rust 포트 |
| **ZeroClaw** | Rust | B | — | **S** | IronClaw 방향 + Soul Snapshot(자아 연속성) 추가 |
| **PicoClaw** | Go | C | — | C | OpenClaw가 너무 크다 → 엣지·임베디드 특화 |
| **TinyClaw** | TS | D | — | D | 팀 오케스트레이션만. 자체 도구·기억 없음 |
| **serverless 계열** | 다양 | B | — | B | AWS·Cloudflare 배포 문제만 해결 |
| **OpenFang** | Rust | **A** | **40** | A | 방향이 다름. Agent OS 선언. Hands+WASM+Taint Tracking. 실행 중 감시·격리는 S급이나 승인 게이트가 LLM 프롬프트 기반 |
| **OpenJarvis** | Python+Rust | A | 6 | B+ | 로컬 퍼스트. RBAC 10종·LoRA 자기개선·6 추론엔진 |
| **NemoClaw** | JS/TS/Py | C | — | **A+** | NVIDIA GPU 샌드박스 플러그인. 범용 아님 |
| **NullClaw** | Zig | B | 19 | **S** | 678KB·<2ms. 기능보다 존재 방식 자체가 다름 |
| **Hermes Agent** | Python | B | 6 | B+ | 자기개선+Tirith pre-exec. 보안 설계 철학이 독특 |
| **Claude Code** | JS | A | 2 | **A+** | 개발 도구 특화. 채널은 적지만 MCP 생태계 완비 |
| **GoClaw** | Go | B+ | 7 | **A+** | 멀티테넌트·AES·Tailscale VPN. 엔터프라이즈 인프라 |
| **CoPaw** | Python | B+ | **14** | C | 채널 최다. AgentScope 위에 채널 레이어 전체를 직접 구축 |
| **MetaClaw** _(applied)_ | Python | B | — | — | 7개+ Claw 백엔드 교체 프록시. 프레임워크가 아닌 메타 레이어 |

**읽는 법**
- OpenClaw: 기능 S·보안 B — 빠른 성장 속에 보안이 뚫렸다
- IronClaw·ZeroClaw: 기능 B·보안 S — OpenClaw의 보안 결핍을 메우기 위해 기능을 희생했다
- OpenFang: 기능 A·보안 A — 둘 다 잡으려 한 유일한 시도. 실행 감시는 S급이나 승인 게이트가 LLM 기반이라 IronClaw·ZeroClaw의 HITL보다 약함
- NullClaw: 기능 B·보안 S — 기능은 충분하지만 "존재 방식"(678KB 정적 바이너리)이 다른 차원

### 종합 티어

```
Tier S: OpenClaw       -- 기능 완성도 최고. 보안만 보강하면 만능
        CoPaw          -- 채널 최다(14개)+QueueKey 격리+Playwright 브라우저. AgentScope 종속이 약점
Tier A: IronClaw       -- 보안+기억 최강. 진입장벽이 약점
        ZeroClaw       -- 효율+자아연속성 유일무이. 생태계가 약점
        OpenFang       -- Agent OS: 40채널+Hands+WASM 감시 최상급. 승인 게이트 LLM 기반이 천장
        NanoClaw       -- 보안+확장 균형. 기억을 MCP로 보강하면 S급 후보
        OpenJarvis     -- 멀티채널 자율 에이전트. A2A+유연한 메모리 강점
        NullClaw       -- 보안 S급+배포 S급. 브라우저 없고 커뮤니티 작은 것이 약점
        GoClaw         -- 멀티테넌트+AES+Tailscale VPN 엔터프라이즈 수준. CC BY-NC 라이선스 제약
Tier B: Nanobot        -- 최고의 학습/연구용. 프로덕션에는 부족
        PicoClaw       -- 병렬 실행 유일. 나머지는 보통
        NemoClaw       -- 특화형: 보안/GPU 탁월, 단독 스탠드얼론은 제한적
        Hermes Agent   -- 자기개선+Skills 오픈 스탠다드. 단독 채널 구성 복잡
Tier C: TinyClaw       -- CLI 위임 특수사례. 자체 역량 최소
```

### 프레임워크별 한 줄 평가

| 프레임워크 | 평가 | 강점 | 약점 |
|-----------|------|------|------|
| **OpenClaw** | 올인원 최강. 뭐든 있지만 거대함 | 기억 Tier 1, 브라우저 50+, 24개 훅 플러그인, 메신저 내장, 커뮤니티 최대 | 보안 Tier 2, ~400K LOC, 인프로세스 실행 |
| **OpenFang** | Agent OS: 자율 실행 풀스택 | 40채널, 7 Hands, 16-layer 보안, A2A, ~32MB 단일 바이너리 | 기억 Phase 1 (SQLite), 승인 게이트 LLM 기반 |
| **IronClaw** | 보안의 끝판왕. 엔터프라이즈급 | WASM 샌드박스, credential proxy, pgvector+RRF | Rust 전용, 메신저 없음, 배포 복잡 |
| **ZeroClaw** | 극한 효율 + 자아 연속성 | 5MB RAM, Soul Snapshot, 3중 브라우저 백엔드 | Rust, 메신저 없음, 플러그인 생태계 빈약 |
| **NanoClaw** | 보안+확장성의 균형점 | 컨테이너 격리, 에이전트 스웜 최초, "8분이면 이해" | 자체 기억 없음 (MCP로 보강 필요) |
| **OpenJarvis** | 멀티채널 자율 에이전트 | A2A 지원, 유연한 메모리 아키텍처, 다채널 | 단독 GPU/샌드박스 미지원 |
| **Nanobot** | 가장 읽기 쉬운 코드. 연구자 친화 | 4000줄 Python, MCP 지원 | 기억 Tier 2, 브라우저 없음, 보안 최소 |
| **PicoClaw** | 유일한 병렬 도구 실행 | Go goroutine+WaitGroup, 10MB RAM | 기억 Tier 2, 메신저/브라우저 없음 |
| **NemoClaw** | GPU 특화 샌드박스 플러그인 | 4-layer 보안, GPU 최적화, Apache 2.0, 25K LOC | OpenClaw 종속, 단독 운용 불가, 기억·채널 위임 |
| **NullClaw** | 보안+이식성 최강 경량 런타임 | 678KB 정적 바이너리, ChaCha20+Landlock, WASI 유일, 19채널, 10 메모리 엔진 | 브라우저 없음, Zig 생태계 소규모 |
| **Hermes Agent** | 자기개선 + Skills 오픈 스탠다드 | Frozen Snapshot(prefix cache 보존), Tirith pre-exec 스캐너, Skills Trust 4단계, agentskills.io | 채널 구성 복잡, 단독 배포 시 설정 많음 |
| **GoClaw** | 엔터프라이즈 멀티테넌트 Go 게이트웨이 | Docker 3축 샌드박스(R38), Tailscale VPN(R39), AES 암호화, 176K LOC 단일 바이너리 | CC BY-NC 4.0(비상업), Go 생태계 ML 빈약 |
| **CoPaw** | 14채널 Skills 기반 개인 비서 | 채널 최다(14개, R42 QueueKey 격리), Playwright 브라우저(3,460줄), 3단계 보안 스캐닝 | AgentScope(==1.0.18) 버전 고정, 중국 채널 편중 |
| **MetaClaw** _(repos_applied)_ | 7개+ Claw 런타임 교체 가능한 메타 레이어 | Swap-Runtime Proxy(R35), MadMax 유휴-창 RL(R36), 실패 기반 스킬 진화(R37), GRPO 온라인 학습 | 프록시 계층 추가 레이턴시, aiming-lab 의존 |
| **TinyClaw** | 멀티팀 오케스트레이션 특화 | TUI 대시보드, 체인 실행 | 자체 도구 0개, 기억 없음, CLI 위임 의존 |

### 용도별 추천

| 용도 | 1순위 | 2순위 | 이유 |
|------|-------|-------|------|
| **24시간 메신저 에이전트** | OpenFang | OpenClaw | 40채널+Hands 자율 실행 vs 올인원 |
| **연구랩 에이전트** | NanoClaw+MCP | OpenClaw | 보안(컨테이너)+스웜+작은 코드 |
| **엔터프라이즈/금융** | IronClaw | ZeroClaw | 보안 Tier 1 필수 |
| **임베디드/엣지** | NullClaw | ZeroClaw | 678KB/1MB RAM, WASI 이식성, Landlock 보안 |
| **프로토타이핑/학습** | Nanobot | NanoClaw | 4000줄, 즉시 이해 가능 |
| **멀티에이전트 오케스트레이션** | NanoClaw | OpenJarvis | 스웜/팀 네이티브 지원, A2A |
| **GPU 워크로드 에이전트** | NemoClaw+OpenClaw | IronClaw | GPU 샌드박스+4-layer 보안 |
| **커스텀 포크** | NanoClaw | Nanobot | 작은 코드, 포크 철학 |

---

## Anthropic 채널·Dispatch vs Claw 생태계 비교

Anthropic은 2026년 3월에 두 가지 별개의 기능을 출시했다. Claw 생태계의 채널/dispatch 설계와 어떻게 다른지 비교한다.

### Claude Code Channels (tengu_harbor) vs Claw 채널

Claude Code v2.1.80에 내장된 **Channels** 기능(내부 코드명: `tengu_harbor`)은 MCP Notification을 채널 표준으로 채택한다. 외부 메시징 플랫폼(Telegram, Discord)이 `notifications/claude/channel` 단일 메서드로 활성 세션에 메시지를 주입한다.

```
[Telegram/Discord]
    ↓ long poll / WebSocket
[Channel MCP Server (Bun)]
    ↓ stdio MCP notification
[Claude Code cli.js]
    qMq() 5단계 게이트 → AMq() XML 래핑 → 세션 컨텍스트 주입
    ↓
Claude LLM → reply tool → MCP server → platform
```

| 차원 | Claw 프레임워크 (OpenClaw/CoPaw/Nanobot 등) | Claude Code Channels (tengu_harbor) |
|------|---------------------------------------------|--------------------------------------|
| **채널 수** | 7–40개 동시 | 공식 2개 (Telegram, Discord) + fakechat |
| **실행 모델** | 24/7 daemon/container | 세션 바인딩 (세션 종료 시 채널 소멸) |
| **allowlist 제어** | 운영자 로컬 설정 (`access.json`) | Anthropic 서버사이드 (`tengu_harbor_ledger` 피처 플래그) |
| **멀티 플랫폼 동시** | O (이벤트 버스) | △ (순차적, 단일 세션) |
| **커스텀 채널 개발** | O (자유롭게 추가) | X (Research Preview 동안 Anthropic allowlist 필요) |
| **채널 트랜스포트** | HTTP/WebSocket/gRPC (각자 구현) | MCP stdio (표준화) |
| **메시지 포맷** | 채널별 상이 | XML `<channel source="…">` 통일 |
| **보안** | 운영자 구성 | R23–R25 (MCP-as-Channel Bridge, Platform-Controlled Allowlist, Content/Meta 분리) |
| **오픈소스** | O (대부분 MIT/Apache) | X (npm 독점 번들) |

**핵심 차이**: Claw 채널은 운영자가 완전히 제어하는 장기 서비스. Claude Code Channels는 Anthropic이 허용 범위를 서버사이드에서 결정하는 플랫폼 통제 모델.

---

### Claude Cowork Dispatch vs Claw 멀티에이전트 Dispatch

**Claude Cowork Dispatch** (2026-03-17, Research Preview)는 Claw의 "채널→에이전트" 구조와 다르다. 모바일 앱(Claude)에서 데스크톱의 Claude를 원격 조작하는 **단일 지속 대화 스레드** 방식이다.

```
[Claude 모바일 앱]
    ↓ (Cowork Dispatch 연결)
[Claude Desktop (로컬)]
    → Computer Use (클릭/타이핑/파일 탐색)
    → 38개+ 앱 통합 (로컬 실행)
```

**Claw 멀티에이전트 dispatch** (예: GoClaw `team_tool_dispatch.go`, TinyClaw 팀 오케스트레이션)는 다른 개념이다. 리드 에이전트가 태스크를 멤버 에이전트에게 분배하는 DAG 기반 병렬 실행이다.

| 차원 | Claw 멀티에이전트 Dispatch | Claude Cowork Dispatch |
|------|---------------------------|------------------------|
| **목적** | 태스크를 여러 에이전트에 병렬 분배 | 모바일에서 데스크톱 Claude 원격 조작 |
| **실행 위치** | 서버/컨테이너 (온라인) | 로컬 데스크톱 (오프라인 가능) |
| **에이전트 수** | N개 병렬 (팀 구조) | 1개 (단일 데스크톱 인스턴스) |
| **컴퓨터 제어** | 일부 (Playwright/go-rod/CDP) | O (Computer Use 기본 포함) |
| **채널 연결** | Telegram/Discord 등 → 에이전트 팀 | 모바일 앱 → 데스크톱 전용 |
| **가용성** | 오픈소스, 자유 배포 | Claude Pro/Max 구독 필요 (Research Preview) |
| **MetaClaw 연관** | MetaClaw는 7개 Claw 백엔드를 런타임 수준에서 교체 가능 — Claw dispatch 프록시 레이어 | 해당 없음 |

**결론**: Claude Cowork Dispatch와 Claw dispatch는 서로 다른 문제를 해결한다. 전자는 개인 사용자의 원격 PC 제어, 후자는 서버에서 멀티에이전트 태스크 분배. 두 패러다임을 동시에 필요로 하면 Claw 채널 위에 Computer Use를 도구로 통합하는 접근(OpenFang의 Native CDP, NanoClaw의 브라우저 도구 등)이 현재 유일한 대안이다.

---

### NullClaw vs PicoClaw vs TinyClaw — "작다"의 의미가 다르다

이름은 다 작은데 "작다"의 의미가 전혀 다릅니다.

| | NullClaw | PicoClaw | TinyClaw |
|--|---------|---------|---------|
| **언어** | Zig | Go | TypeScript |
| **바이너리** | 678KB | 소형 | ~400 LOC |
| **채널** | 19개 | — | — |
| **기억** | 10 엔진 | 기본 | 없음 |
| **도구** | 35+ built-in | 기본 | **없음** |
| **보안** | Tier 1 | Tier 3 | Tier 4 |
| **고유 기능** | WASI, Landlock, Signal+Nostr+Matrix | 병렬 도구 실행 (생태계 유일) | 팀 오케스트레이션 |

- **NullClaw**: 바이너리가 작은 것이지 기능이 작은 게 아닙니다. 19채널·10 메모리 엔진·ChaCha20 암호화·Landlock 샌드박스가 전부 678KB 안에 들어갑니다. 작으면서 가장 많이 합니다.
- **PicoClaw**: 실행 환경이 작은 것입니다. 엣지·임베디드 타겟. 생태계 전체에서 병렬 도구 실행이 가능한 유일한 프레임워크입니다.
- **TinyClaw**: 기능 자체가 작습니다. 자체 도구·기억·보안 없음. 다른 에이전트에게 위임하는 오케스트레이션과 TUI 대시보드가 전부입니다.

> NullClaw: **작지만 풀스택** / PicoClaw: **작은 환경을 위한 특화** / TinyClaw: **작은 게 아니라 비어있음**

---

## 활용 방법

### 설계 의사결정

각 선택지의 트레이드오프를 이해하기 위해:
- **세션 관리**: reports/repos/session_context_report.md
- **보안**: reports/repos/security_report.md
- **기억 시스템**: reports/repos/memory_architecture_report.md
- **브라우저/도구**: reports/repos/browser_actions_report.md

### 미해결 문제 해결

Q1-Q15 중 관심 있는 질문에 대해:
- 기존 보고서의 근거 검토
- 각 프레임워크의 소스코드 직접 탐색
- 다른 구현체로의 패턴 적용 가능성 평가

---

## 라이선스 및 저작권

본 프로젝트는 **분석 및 비교** 목적의 문서 모음입니다. 각 프레임워크의 소스코드는 원본 저장소의 라이선스를 따릅니다:
- OpenClaw: MIT
- NemoClaw: Apache 2.0
- NullClaw: MIT
- Nanobot: 원본 저장소 참조
- 기타: 각 저장소 참조

---

## 아이디어 확장: 분석 결과의 실제 적용

4개 보고서의 인사이트를 실제 시스템 설계로 연결하는 아이디어 문서들:

| 파일 | 주제 | 핵심 |
|------|------|------|
| **ai-research-agent-design.md** | AI Research Agent | 논문 검색·분석·집필 파이프라인. X→Obsidian→CLAUDE.md 지식 자동 수집 구조 포함 |

적용된 패턴 ([@DeepDive_KR 검증](https://x.com/DeepDive_KR/status/2029218235019563335), 2026-03-05):
- `settings.json additionalDirectories`로 Obsidian vault를 Claude가 직접 검색
- 수집(넓게) → AI 점수화 → 라우팅(좁게): PRINCIPLE/SKILL/MCP/REFERENCE/SKIP
- Pre-apply Validation 8항으로 인사이트 적용 전 품질 게이팅

---

## 업데이트 이력

- **2026-03-30b**: MetaClaw 상세 항목 확장(Stars/LOC/보고서 링크); Anthropic 채널·Dispatch vs Claw 비교 섹션 신설 (tengu_harbor MCP Channels vs Claw 채널, Claude Cowork Dispatch vs Claw 멀티에이전트 dispatch)
- **2026-03-30**: GoClaw(Go, 1.4k stars)·CoPaw(agentscope-ai, 13.6k stars) repos/ 등록 (14th~15th→최종 14개); R38-R42 신규 패턴 (3축 샌드박스, Tailscale VPN, OTel-first, 프롬프트 튜너, 3-tuple QueueKey); 평가 매트릭스 14개 프레임워크로 확장; MetaClaw README 분석 대상 추가; Claude Code 분석 테이블 제외; AgentScope는 CoPaw 기반 플랫폼으로 copaw_report 내 분석
- **2026-03-26**: MetaClaw(aiming-lab, 2.7k stars) repos_applied 등록; R35-R37 신규 패턴 (Swap-Runtime Proxy, MadMax 유휴-창 RL, 실패 기반 스킬 진화+MAML 버퍼 분리); ideas/ → .ideas/ 리네임; 리포 이름 Compare Claws → Claws Aquarium; .omc/.cwf git 추적 제거
- **2026-03-23**: Hermes Agent(12th, Nous Research)·Claude Code(13th, Anthropic) 추가; usecases/ 카테고리 신설 (agency-agents·awesome-openclaw-agents·awesome-openclaw-usecases·서울밋업 4종); R17~R34 신규 패턴; repos_applied/autoresearch-skill 추가; 평가 매트릭스 13개 프레임워크로 확장
- **2026-03-17**: NullClaw(Zig) 11번째 프레임워크 추가; 밋업 보고서 2개 신설 (실전 운영 패턴, 에이전트 결제·신원 프로토콜); 상용화 비교 확대 (8개 서비스: Crew.you·Genspark·Perplexity PC 신규); README 구조 개편
- **2026-03-17** _(이전)_: NemoClaw 정식 출시 반영, OpenFang·OpenJarvis·NemoClaw 종합 보고서 전면 업데이트, 보고서 리네임 (framework_cross_analysis.md, framework_catalog.md)
- **2026-03-09**: repos_research/ 카테고리 추가 (DeepInnovator, Autoresearch), 6번째 보고서 완성 (연구 자동화 도구 분석)
- **2026-03-08**: idea3/idea4 아이디어 문서 추가 및 X→Obsidian→Agent 파이프라인 패턴 반영
- **2026-03-05**: 4번째 보고서 완성 (기억 아키텍처)
- **2026-03-05**: 3번째 보고서 완성 (브라우저/도구, 보안 교차검증)
- **2026-03-04**: 2번째 보고서 완성 (세션/컨텍스트)
- **2026-02-25**: 1번째 보고서 완성 (생태계 개요)

---

## 감사의 말

이 프로젝트는 7개 에이전트 팀(일부는 scientist/analyst/architect 고급 에이전트)의 병렬 심층 분석으로 가능했습니다. 각 프레임워크 개발자들의 혁신적 설계 선택에 감사합니다.
