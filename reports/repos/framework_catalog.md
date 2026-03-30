# OpenClaw 생태계 종합 조사 보고서

> 조사일: 2026-02-25 (최종 업데이트)
> 출처: https://x.com/techNmak/status/2025659710775668814

---

## 목차

1. [OpenClaw (원조)](#1-openclaw-원조--typescriptnodejs)
2. [Nanobot (Python)](#2-nanobot--python-4000-loc)
3. [NanoClaw (TypeScript)](#3-nanoclaw--typescript-500-loc-코어)
4. [IronClaw (Rust)](#4-ironclaw--rust-보안-특화)
5. [ZeroClaw (Rust)](#5-zeroclaw--rust-극한-경량화)
6. [PicoClaw (Go)](#6-picoclaw--go-초경량-엣지-디바이스)
7. [TinyClaw (TypeScript)](#7-tinyclaw--typescript-멀티에이전트-팀)
8. [클라우드 배포 방식들](#8-클라우드-배포-방식들)
    - [serverless-openclaw](#81-serverless-openclaw--aws-서버리스-월-1)
    - [OpenClaw on AWS with Bedrock](#82-openclaw-on-aws-with-bedrock--aws-공식)
    - [KimiClaw](#83-kimiclaw--cloudflare-서버리스--moonshot-ai-공식)
9. [OpenFang (Agent OS)](#9-openfang--rust-agent-os)
10. [OpenJarvis (로컬 퍼스트)](#10-openjarvis--python--rust-로컬-퍼스트-개인-ai)
11. [NemoClaw — NVIDIA](#11-nemoclaw-nvidia--openclaw-샌드박스-플러그인)
12. [NullClaw (Zig)](#12-nullclaw--zig-극한-최소화)
13. [Hermes Agent (Nous Research)](#13-hermes-agent--python-자기개선-에이전트)
14. [Claude Code (Anthropic)](#14-claude-code--javascript-공식-플랫폼)
15. [GoClaw (Go)](#15-goclaw--go-멀티테넌트-ai-게이트웨이)
16. [CoPaw (agentscope-ai)](#16-copaw--python-14채널-개인-비서)
17. [종합 비교표](#종합-비교표)
18. [핵심 인사이트](#핵심-인사이트)
    - [자격증명 딜레마](#6-자격증명-딜레마--권한을-줘야-일을-하는데-주면-위험하다)

---

## 1. OpenClaw (원조) — TypeScript/Node.js

| 항목 | 내용 |
|------|------|
| **GitHub** | [openclaw/openclaw](https://github.com/openclaw/openclaw) |
| **Stars** | 223,000+ |
| **개발자** | Peter Steinberger (PSPDFKit 창시자, 이후 OpenAI 합류) |
| **라이선스** | MIT |

### 이름 변천사

- **ClawdBot** (2025.11) → **Moltbot** (2026.1.27) → **OpenClaw** (2026.1.30)
- "Clawd"가 Anthropic의 "Claude" 상표와 유사해 법적 통보를 받고 개명
- 개명 과정에서 스캐머가 버려진 `@clawdbot` 트위터 계정을 탈취해 가짜 `$CLAWD` 토큰 발행 (시총 $16M까지 급등 후 붕괴)

### 핵심 특징

- **셀프 호스팅** 개인 AI 에이전트 플랫폼 (코딩 도구가 아닌 범용 AI 비서)
- WhatsApp, Telegram, Slack, Discord, Signal, iMessage 등 **12개+ 메시징 채널** 지원
- 음성 모드 (ElevenLabs 통합), Canvas 비주얼 워크스페이스, 브라우저 제어
- **ClawHub** 스킬 마켓플레이스, Cron 자동화, 웹훅
- Lobster 워크플로우 셸: 스킬/도구를 조합 가능한 파이프라인으로 전환

### 보안 위기 (폭발적 성장의 그늘)

OpenClaw의 급격한 성장과 함께 심각한 보안 사고가 동시다발적으로 발생했습니다.

**CVE-2026-25253 (CVSS 8.8) — 1-Click 원격 코드 실행**

- Control UI의 `gatewayUrl` 쿼리 파라미터 검증 부재 악용
- 공격자 지정 주소로 WebSocket 연결을 자동 수립, 인증 토큰 전송
- WebSocket 오리진 헤더 미검증 → 크로스사이트 WebSocket 하이재킹
- 공격 체인: 악성 웹페이지 → 인증 토큰 탈취 → 보안 확인 비활성화 → 컨테이너 이스케이프 → 호스트에서 임의 명령 실행
- v2026.1.29에서 패치됨

**ClawHavoc 공급망 공격**

- ClawHub에 **1,184개 악성 스킬 패키지** 유포
- 암호화폐 거래 자동화 도구로 위장 (ByBit, Polymarket, Axiom 등 실제 브랜드 사용)
- 91%가 프롬프트 인젝션을 동시 사용 → AI 안전 메커니즘과 전통적 보안 도구 모두 우회
- 주요 페이로드: Atomic macOS Stealer (AMOS)
- Koi Security 감사: 2,857개 스킬 중 341개(12%)가 악성, 335개가 단일 조직 캠페인

**노출 규모**

- Censys 추적: 2026.1.25~31 사이 ~1,000개에서 21,000개+ 인스턴스로 급증
- 42,665개 노출 인스턴스 중 93.4%가 인증 우회 상태
- 전체 보안 감사에서 512개 취약점 발견, 8개가 Critical 등급

> 이러한 보안 문제가 수많은 대안들의 등장을 직접적으로 촉발했습니다.

### 참고 링크

- [From Clawdbot to Moltbot to OpenClaw — CNBC](https://www.cnbc.com/2026/02/02/openclaw-open-source-ai-agent-rise-controversy-clawdbot-moltbot-moltbook.html)
- [OpenClaw Complete Guide 2026 — NxCode](https://www.nxcode.io/resources/news/openclaw-complete-guide-2026)
- [OpenClaw Bug Enables One-Click RCE — The Hacker News](https://thehackernews.com/2026/02/openclaw-bug-enables-one-click-remote.html)
- [ClawHavoc Poisons ClawHub — CyberPress](https://cyberpress.org/clawhavoc-poisons-openclaws-clawhub-with-1184-malicious-skills/)
- [Personal AI Agents Are a Security Nightmare — Cisco Blogs](https://blogs.cisco.com/ai/personal-ai-agents-like-openclaw-are-a-security-nightmare)

---

## 2. Nanobot — Python (~4,000 LOC)

| 항목 | 내용 |
|------|------|
| **GitHub** | [HKUDS/nanobot](https://github.com/HKUDS/nanobot) |
| **Stars** | 24,100+ |
| **개발** | 홍콩대학교 데이터사이언스 랩 (HKUDS) |
| **주요 메인테이너** | Re-bin (`@Re-bin`) |
| **라이선스** | MIT |

### 핵심 특징

- OpenClaw 대비 **99% 작은 코드베이스** (~3,897줄 vs 430,000+줄)
- 연구에 바로 활용 가능한 깔끔하고 가독성 높은 코드
- 두 가지 운영 모드: `nanobot agent` (CLI) / `nanobot gateway` (멀티채널 서버)

### 아키텍처 (5개 모듈 레이어)

| 레이어 | 모듈 | 역할 |
|--------|------|------|
| Agent Engine | `agent/` | AgentLoop, ContextBuilder, MemoryStore, SkillsLoader, SubagentManager |
| Communication | `bus/`, `session/`, `channels/` | MessageBus (asyncio 큐), SessionManager (JSONL) |
| Providers | `providers/` | LiteLLM 기반 레지스트리 구동 LLM 라우팅 |
| Tool Ecosystem | `tools/` | 빌트인 도구, Skills (마크다운), MCP 서버 |
| Configuration | `~/.nanobot/config.json` | Pydantic 스키마 + 환경 변수 오버라이드 |

- **AgentLoop**: 최대 20회 반복으로 제한 (폭주 방지)
- **메모리**: 50개 메시지 슬라이딩 윈도우 + 비동기 JSONL 통합
- **부트스트랩 컨텍스트**: `SOUL.md`, `USER.md`, `AGENTS.md`가 매 프롬프트에 주입

### MCP 지원 (v0.1.4+)

- Claude Desktop/Cursor와 **동일한 설정 스키마** 사용 (마이그레이션 불필요)
- stdio (로컬 프로세스) + HTTP/SSE (리모트 엔드포인트) 양방향 트랜스포트
- MCP 도구 자동 검색 및 등록, 인증 헤더 지원

### 멀티채널 지원 (9개+)

Telegram, Discord, WhatsApp, Slack, Email (IMAP/SMTP), QQ, Feishu, DingTalk, WeChat

### LLM 프로바이더 (15개+)

OpenRouter, Anthropic Claude, OpenAI, DeepSeek, Groq, Google Gemini, MiniMax, Qwen, GitHub Copilot, OpenAI Codex, SiliconFlow, 커스텀 OpenAI 호환 엔드포인트 등

### 비전

> "커널은 모든 드라이버를 탑재하지 않지만, 누구나 드라이버를 작성할 수 있다."
>
> 장기 목표: **에이전트 커널** — Linux처럼 코어는 최소화하고 모든 통합을 `pip install nanobot-{plugin}` 생태계로 확장

### 참고 링크

- [GitHub — HKUDS/nanobot](https://github.com/HKUDS/nanobot)
- [v0.1.4 Release Notes](https://github.com/HKUDS/nanobot/releases/tag/v0.1.4)
- [DeepWiki Architecture Analysis](https://deepwiki.com/HKUDS/nanobot)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46897737)

---

## 3. NanoClaw — TypeScript (~500 LOC 코어)

| 항목 | 내용 |
|------|------|
| **GitHub** | [qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw) |
| **Stars** | 13,500+ |
| **개발자** | Gavriel Cohen (전 Wix 7년 엔지니어, Qwibit 공동창업자) |
| **라이선스** | MIT |
| **출시** | 2026.1.31 (1주 만에 7,000 stars) |

### 핵심 특징

- 코어 코드 약 **500줄** — "8분이면 전체를 이해할 수 있음"
- OpenClaw 400,000줄과 달리 누구나 (또는 보조 AI가) 전체 코드를 감사 가능
- Node.js/TypeScript + SQLite 스택

### 컨테이너 기반 에이전트 실행

- **macOS**: Apple Container (네이티브 macOS 가상화)
- **Linux/macOS 대안**: Docker
- 각 채팅 그룹마다 **독립 컨테이너** 할당 — 파일시스템 완전 격리
- 프롬프트 인젝션 피해 범위가 컨테이너 내부로 한정 (호스트 시스템 보호)
- 그룹별 독립 `CLAUDE.md` 메모리 파일 + 격리된 컨테이너 파일시스템

### 에이전트 스웜 (최초 지원)

- **Anthropic Agent SDK** 네이티브 기반 다중 에이전트 협업
- 서브 에이전트 간 메모리 컨텍스트 격리 → 민감 데이터 유출 방지
- 단일 채팅 인터페이스 내에서 전문 에이전트 팀 협업

### "Fork, Customize, Own" 철학

- 중앙 플러그인 마켓플레이스 **없음** (OpenClaw의 ClawHub와 대조)
- 레포를 **포크** → Claude Code가 직접 소스를 수정하는 방식
- `.claude/skills/` SKILL.md 파일로 확장 (예: `/add-telegram` 실행 → 소스 자동 수정)
- "설정 복잡성" 대신 "코드 수준 커스터마이징" 지향

### 실제 사용 사례

Qwibit 내부에서 "Andy"라는 NanoClaw 인스턴스가 영업 파이프라인 관리, WhatsApp/이메일 노트 파싱, DB 업데이트, 기술 작업 자율 실행에 활용 중

### 참고 링크

- [GitHub — qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw)
- [NanoClaw Official Site](https://nanoclaw.dev)
- [NanoClaw Solves OpenClaw Security Issues — Novalogiq](https://novalogiq.com/2026/02/11/nanoclaw-solves-one-of-openclaws-biggest-security-issues-and-its-already-powering-the-creators-biz/)
- [500 Lines vs 50 Modules — Architecture Analysis](https://fumics.in/posts/2026-02-02-nanoclaw-agent-architecture)
- [The Claw Wars: 11 OpenClaw Spin-Offs — Blocmates](https://www.blocmates.com/articles/the-claw-wars)

---

## 4. IronClaw — Rust (보안 특화)

| 항목 | 내용 |
|------|------|
| **GitHub** | [nearai/ironclaw](https://github.com/nearai/ironclaw) |
| **Stars** | 3,300+ |
| **개발자** | Illia Polosukhin (NEAR Protocol 공동창업자, "Attention Is All You Need" 공저자) |
| **라이선스** | MIT |
| **최신 릴리스** | v0.11.1 (2026.2.23) |

> "사람들이 OpenClaw를 쓰다가 자금과 자격증명을 잃고 있다. 우리는 보안에 초점을 맞춘 버전을 만들기 시작했다." — Illia Polosukhin

### 1. WASM 샌드박스

IronClaw의 핵심 보안 메커니즘. `wasmtime` 런타임으로 서드파티 도구를 격리 실행합니다.

```
WASM → 허용목록 검증 → 유출 스캔(요청) → 자격증명 주입 → 실행 → 유출 스캔(응답) → WASM
```

- 명시적 옵트인 필요: HTTP 접근, 시크릿 접근, 도구 호출
- HTTP 엔드포인트 화이트리스트: 사전 승인된 호스트/경로만 허용
- 리소스 제한: 메모리, CPU, 실행 시간 제약
- 레이트 리밋: 도구별 요청 수 상한

### 2. 자격증명 보호

- API 키가 **절대** LLM 컨텍스트에 노출되지 않음
- **AES-256-GCM 암호화 볼트** (PostgreSQL 기반)
- 실행 시점에만 호스트 경계에서 특정 승인 사이트용으로만 주입
- 유출 감지: 발신 요청과 수신 응답 모두에서 자격증명 유출 패턴 스캔
- Anti-Stealer 모듈: SSH 키 열거, 클라우드 자격증명 접근, 다단계 유출 체인 모니터링

### 3. 프롬프트 인젝션 방어

다단계 방어 스택:
- 패턴 감지 (알려진 인젝션 패턴 사전 차단)
- 콘텐츠 정화 (외부 콘텐츠 클리닝)
- 정책 시행 (콘텐츠 출처에 따른 허용 동작 규칙)

### 기술 아키텍처

- **Rust 1.85+**, tokio 비동기, Arc/RwLock 동시성
- **PostgreSQL 15+** + pgvector (프로덕션) / **libSQL/Turso** (로컬 대안)
- 하이브리드 메모리: 전문 검색 + 벡터 코사인 유사도
- 채널: REPL (Ratatui TUI), HTTP 웹훅, WASM 채널, WebSocket 스트리밍
- 지원 LLM: NEAR AI, Anthropic, OpenAI, Ollama, OpenRouter, Together AI 등
- `.unwrap()` / `.expect()` / clippy 경고 **제로 톨러런스** (코드 리뷰 레벨에서 강제)

### 커뮤니티 평가 (Hacker News)

- "WASM 샌드박스의 위협 모델이 불충분하게 문서화됨" (amluto)
- `webfetch`와 코드 실행 결합 시 근본적 공격 표면 존재
- 샌드박싱은 완벽한 보안 솔루션은 아니지만 없는 것보다 확실히 나음

### 참고 링크

- [GitHub — nearai/ironclaw](https://github.com/nearai/ironclaw)
- [IronClaw rivals OpenClaw — CoinTelegraph](https://magazine.cointelegraph.com/ironclaw-secure-private-sounds-cooler-openclaw-ai-eye/)
- [REPORT: IronClaw — TheCoding Substack](https://thecoding.substack.com/p/report-ironclaw-openclaw-in-rust)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=47004312)

---

## 5. ZeroClaw — Rust (극한 경량화)

| 항목 | 내용 |
|------|------|
| **GitHub** | [zeroclaw-labs/zeroclaw](https://github.com/zeroclaw-labs/zeroclaw) |
| **Stars** | 17,000+ |
| **개발** | Harvard/MIT/Sundai.Club 커뮤니티 |
| **라이선스** | MIT |
| **출시** | 2026.2.13 (2일 만에 3,400+ stars) |
| **최신 릴리스** | v0.1.6 (2026.2.22) |
| **테스트** | 1,017개 |

### 성능 지표

| 지표 | 수치 |
|------|------|
| 바이너리 크기 | ~8.8MB (정적 링킹) |
| 런타임 RAM | **5MB 미만** (CLI 작업 시) |
| 유휴 데몬 | 10-15MB |
| 시작 시간 | **10ms 미만** |

### 경량화 기법

| 기법 | 효과 |
|------|------|
| 순수 Rust 컴파일 | 인터프리터/VM 오버헤드 제거 (Node.js 1GB+, Python 100MB+ 대비) |
| 단일 정적 바이너리 | 런타임 의존성 완전 제거 |
| 모노모피제이션 | 제네릭 트레이트가 컴파일 타임에 해소 → 런타임 오버헤드 제로 |
| 지연 초기화 | 컴포넌트가 최초 사용 시에만 인스턴스화 |
| 제로카피 메시지 전달 | tokio 채널, 할당 오버헤드 없음 |
| 컴파일러 프로파일 | `opt-level = "z"`, `lto = true`, `codegen-units = 1` |
| OpenSSL 미사용 | rustls-tls 사용 |
| 번들 SQLite | `rusqlite` bundled feature |

### 벤치마크

- ZeroClaw: **1.52MB** 활성 메모리
- OpenClaw: **7.8MB**
- 4GB 서버 기준: ZeroClaw **~200개 인스턴스** vs OpenClaw **~2개 인스턴스**

### 트레이트 기반 아키텍처 (8대 핵심 트레이트)

`config.toml`만으로 모든 구현체 교체 가능 — 소스 코드 수정 불필요

| 트레이트 | 역할 | 구현체 예시 |
|----------|------|-------------|
| `Provider` | LLM API 추상화 | OpenAI, Claude, Gemini, Ollama, DeepSeek 등 28개+ |
| `Channel` | 메시징 플랫폼 | Telegram, Discord, Slack, Matrix, WhatsApp, iMessage 등 15개+ |
| `Memory` | 대화 저장 | SQLite 하이브리드 검색, PostgreSQL, Markdown 파일 |
| `Tool` | 에이전트 능력 | Shell, File, HTTP, Browser, Git 등 70개+ |
| `RuntimeAdapter` | 코드 실행 환경 | Native, Docker 샌드박싱 |
| `Observer` | 텔레메트리 | Prometheus, OpenTelemetry OTLP |
| `Tunnel` | 네트워크 노출 | ngrok 호환, 커스텀 |
| `SecurityPolicy` | 접근 제어 | 워크스페이스 스코핑, 커맨드 허용목록 |

Rust의 트레이트 시스템은 **컴파일 타임 다형성**(모노모피제이션)을 제공하여 인터페이스 기반 교체 유연성을 런타임 비용 없이 달성합니다.

### 메모리 하이브리드 검색

외부 의존성 없는 자체 검색 엔진:
- FTS5 키워드 검색 (BM25 스코어링)
- 벡터 코사인 유사도 (저장된 임베딩)
- 가중 퓨전: `score = (keyword_weight x bm25) + (vector_weight x cosine)`
- LRU 임베딩 캐시로 API 호출 최소화

### 대상 하드웨어

- $10 싱글보드 컴퓨터 (Raspberry Pi급)
- 0.8GHz 프로세서
- 지원 아키텍처: ARM (aarch64, armv7), x86_64, RISC-V
- OpenClaw 마이그레이션: `zeroclaw migrate openclaw` 명령으로 기존 메모리/ID 파일 임포트

### 참고 링크

- [GitHub — zeroclaw-labs/zeroclaw](https://github.com/zeroclaw-labs/zeroclaw)
- [DeepWiki — What is ZeroClaw](https://deepwiki.com/zeroclaw-labs/zeroclaw/1.1-what-is-zeroclaw)
- [DEV.to — ZeroClaw Article](https://dev.to/brooks_wilson_36fbefbbae4/zeroclaw-a-lightweight-secure-rust-agent-runtime-redefining-openclaw-infrastructure-2cl0)
- [ZeroClaw Official Site](https://www.zeroclaw.dev/)

---

## 6. PicoClaw — Go (초경량, 엣지 디바이스)

| 항목 | 내용 |
|------|------|
| **GitHub** | [sipeed/picoclaw](https://github.com/sipeed/picoclaw) |
| **Stars** | 17,000~19,000 |
| **개발** | Sipeed (중국 RISC-V 하드웨어 제조사) |
| **라이선스** | MIT |
| **출시** | 2026.2.9 (1주 만에 12,000+ stars) |

### 핵심 특징

- Go 컴파일 → **단일 정적 바이너리**, 런타임 의존성 제로
- 로컬 LLM을 실행하지 않음 — **클라우드 API 호출 전용 메시지 브로커**
- 메시지 라우팅 + 도구 호출 + 스케줄링만 담당하는 **마이크로커널** 설계

### 리소스 사용량

| 지표 | PicoClaw | OpenClaw |
|------|----------|----------|
| RAM | <10MB (초기), 10-20MB (최근) | >1GB |
| 시작 시간 | <1초 | 수분 |
| 바이너리 | 단일 파일 | Node.js 프로세스 트리 |

### 아키텍처

```
User Interface Layer    → CLI + 멀티채널 (Telegram, Discord, WhatsApp, QQ, DingTalk)
Gateway Service Layer   → 채널 오케스트레이션, 스케줄링, 헬스 모니터링
Core Application        → AgentLoop + MessageBus pub/sub 시스템
LLM Abstraction Layer   → 13개+ 모델용 Provider 인터페이스
Tool Ecosystem          → 파일 작업, 웹 검색, 셸 실행, Cron 스케줄링
```

### 구형 안드로이드 폰 지원

```bash
# Termux에서 실행
pkg install proot wget
wget https://github.com/sipeed/picoclaw/releases/download/v0.1.1/picoclaw-linux-arm64
chmod +x picoclaw-linux-arm64
termux-chroot ./picoclaw-linux-arm64 onboard
```

- ARM64 정적 바이너리 → 추가 라이브러리 불필요
- 지원 아키텍처: `linux-arm64`, `linux-armv6`, `linux-mips64`, `linux-riscv64`, `linux-x86_64`

### 95% AI 생성 코드베이스

- NanoBot(Python) → Go 포팅을 **AI 에이전트가 자체 수행** (셀프 부트스트래핑)
- 인간이 아키텍처 감독 + 리뷰 + 스펙 작성 담당
- 95%는 **생성된 원시 코드 줄** 기준이며, 아키텍처 설계는 인간이 주도
- HN 커뮤니티 평가: "성능 향상은 Go 언어 특성에서 기인하며 신뢰할 만함"

### 개발사: Sipeed

- 중국의 RISC-V 개발 보드 전문 제조사 (Maix, LicheeRV, Tang FPGA)
- PicoClaw는 자사 초저가 하드웨어($10 LicheeRV-Nano 등)에 소프트웨어 생태계를 구축하려는 전략의 일환

### 주의사항

- v1.0 전까지 **프로덕션 배포 비권장** (네트워크 보안 미해결)
- 최근 PR 급증으로 메모리 10-20MB로 증가 추세
- 오프라인/로컬 LLM 미지원 (클라우드 API 키 필수)

### 참고 링크

- [GitHub — sipeed/picoclaw](https://github.com/sipeed/picoclaw)
- [CNX Software — Technical Deep Dive](https://www.cnx-software.com/2026/02/10/picoclaw-ultra-lightweight-personal-ai-assistant-run-on-just-10mb-of-ram/)
- [Hackster.io — OpenClaw Alternative for $10](https://www.hackster.io/news/forget-the-mac-mini-run-this-openclaw-alternative-for-just-10-da23b2819d25)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=47004845)

---

## 7. TinyClaw — TypeScript (멀티에이전트 팀)

| 항목 | 내용 |
|------|------|
| **GitHub** | [jlia0/tinyclaw](https://github.com/jlia0/tinyclaw) (원본) / [TinyAGI/tinyclaw](https://github.com/TinyAGI/tinyclaw) (발전형) |
| **개발자** | Jian (jlia0, 샌프란시스코) |
| **라이선스** | MIT |

> "OpenClaw는 훌륭하지만 항상 깨진다. 그래서 셸 스크립트 ~400줄로 OpenClaw를 재창조했다."

### 멀티에이전트 아키텍처

- 에이전트 유형: `coder`, `writer`, `reviewer`, `researcher`, `documentation`
- 각 에이전트가 **독립 워크스페이스 + 대화 히스토리** 보유
- `@agent_id` 구문으로 태스크 라우팅
  ```
  @coder API 빌드; @writer 문서 작성; @reviewer PR 감사
  ```
- SQLite 기반 메시지 큐 (WAL 모드, 원자적 트랜잭션)
- 메시지 상태 흐름: pending → processing → completed / dead-letter (최대 5회 재시도)

### 멀티팀 체인 실행

- `settings.json`에서 팀 정의 + 리더 기반 라우팅
- **체인 모드**: Leader → Agent A → Agent B → Agent C (순차 핸드오프)
- **팬아웃 모드**: Leader가 서브태스크를 여러 에이전트에 **병렬 배포**
- 격리된 팀 대화 로그: `.tinyclaw/chats/{team_id}/`

### 실시간 TUI 대시보드

```bash
tinyclaw team visualize [id]
```

- 에이전트 체인 플로우를 터미널에서 실시간 시각화
- 태스크 핸드오프, 실행 상태, 라이브 메트릭 표시

### TinyOffice 웹 포털 (보너스)

- Next.js 기반 브라우저 대시보드
- 채팅 콘솔, 에이전트/팀 라우팅 인터페이스
- 칸반 스타일 태스크 관리
- 라이브 로그, 이벤트 인스펙션, 설정 에디터

### 24/7 운영

- tmux 세션 기반 프로세스 관리로 지속 운영
- Discord, Telegram, WhatsApp 동시 연결
- Claude Code CLI / OpenAI Codex CLI 위에서 구동

### 멀티채널

- Discord (봇 토큰 + 메시지 콘텐츠 인텐트)
- Telegram (BotFather 토큰)
- WhatsApp (QR 코드 디바이스 링킹)
- 발신자 허용목록 + 대기/승인 보안

### 주요 포크/변종

| 프로젝트 | 특징 |
|----------|------|
| [TinyAGI/tinyclaw](https://github.com/TinyAGI/tinyclaw) | 완전한 멀티팀 아키텍처 (한국어 포스트의 설명과 일치) |
| [warengonzaga/tinyclaw](https://github.com/warengonzaga/tinyclaw) | Bun 런타임, 3레이어 적응형 메모리, GPLv3, 독립 프로젝트 |
| [suislanchez/tinyclaw](https://github.com/suislanchez/tinyclaw) | TUI, 스트리밍, 병렬 도구 |

### 참고 링크

- [GitHub — jlia0/tinyclaw](https://github.com/jlia0/tinyclaw)
- [GitHub — TinyAGI/tinyclaw](https://github.com/TinyAGI/tinyclaw)
- [Open Alternative to OpenClaw — TinyClaw](https://www.scriptbyai.com/ai-assistant-tinyclaw/)
- [TinyClaw ClawRouter Architecture](https://sonusahani.com/blogs/tinyclaw-clawrouter)

---

## 8. 클라우드 배포 방식들

OpenClaw의 근본적 문제를 해결하지는 않지만, **더 편하게/싸게/안전하게 배포**하려는 시도들입니다.

### 8.1 serverless-openclaw — AWS 서버리스 (월 ~$1)

| 항목 | 내용 |
|------|------|
| **GitHub** | [serithemage/serverless-openclaw](https://github.com/serithemage/serverless-openclaw) |
| **Stars** | 126 |
| **개발자** | 정도현 (전 AWS 테크니컬 트레이너 2016-2024, AWS 한국 사용자 그룹 리더, Roboco.io 창업자) |
| **라이선스** | MIT |
| **비용** | Free Tier 내 ~$0.27/월, 이후 ~$1.11/월 |

#### 핵심 아키텍처

OpenClaw를 AWS 서버리스로 돌려서 개인 사용 시 **월 $1** 수준으로 비용을 절감한 프로젝트입니다.

| 계층 | 기술 | 역할 |
|------|------|------|
| 인터페이스 | React SPA (S3+CloudFront) + Telegram Bot | 사용자 접점 |
| API | API Gateway (WebSocket + REST) | 요청 라우팅 |
| 컴퓨팅 | Lambda 7개 (게이트웨이) + ECS Fargate Spot (에이전트) | 주문형 실행 |
| 인증 | AWS Cognito + JWT | 모든 요청에 인증 |
| 저장 | DynamoDB 5개 테이블 + S3 | 대화 히스토리, 파일 |

#### 비용 절감 핵심

| 제거한 것 | 절약액 |
|-----------|--------|
| ALB (Application Load Balancer) | -$18~25/월 |
| NAT Gateway | -$33/월 |
| 상시 가동 인스턴스 | 사용 안 할 때 $0 |
| + Fargate Spot (70% 할인) + ARM64 Graviton | 추가 절감 |

#### 보안 개선점

- Cognito JWT 인증 (OpenClaw 기본은 인증 없음)
- SSM Parameter Store로 시크릿 관리 ("디스크에 시크릿 기록 안 함" 원칙)
- HTTPS 강제 (CloudFront + API Gateway)
- 최소 권한 IAM 역할

#### 한계

- OpenClaw 앱 레벨 취약점(CVE-2026-25253 등)은 여전히 존재
- 프롬프트 인젝션, ClawHub 악성 스킬 문제 미해결
- 콜드 스타트 ~68초 (프리워밍 시 ~0초, +$0.07/월)

#### 참고 링크

- [GitHub — serithemage/serverless-openclaw](https://github.com/serithemage/serverless-openclaw)
- [serithemage GitHub 프로필](https://github.com/serithemage)

---

### 8.2 OpenClaw on AWS with Bedrock — AWS 공식

| 항목 | 내용 |
|------|------|
| **GitHub** | [aws-samples/sample-OpenClaw-on-AWS-with-Bedrock](https://github.com/aws-samples/sample-OpenClaw-on-AWS-with-Bedrock) |
| **Stars** | 359 |
| **만든 곳** | AWS (aws-samples 공식 레포) |
| **라이선스** | MIT-0 |
| **비용** | ~$46/월 (EC2 + VPC 엔드포인트, 모델 비용 별도) |

#### 핵심 아키텍처

OpenClaw + Amazon Bedrock → **API 키 없이** IAM 역할로 LLM 인증.

```
사용자 메시징 앱 → EC2 (OpenClaw Gateway) → Amazon Bedrock → LLM 응답
```

| AWS 서비스 | 역할 |
|-----------|------|
| Amazon Bedrock | 통합 LLM 추론 API |
| EC2 (Graviton ARM) | OpenClaw 게이트웨이 실행 |
| IAM | 역할 기반 인증 (API 키 저장 불필요) |
| VPC + PrivateLink | Bedrock 트래픽이 퍼블릭 인터넷을 안 탐 |
| CloudTrail | 모든 API 호출 감사 로깅 |
| SSM Session Manager | SSH 포트 노출 없이 접근 |

#### Bedrock의 장점

**1. API 키가 아예 없음**
- IAM 역할이 자동 인증 → 키 회전/노출/커밋 사고 불가능

**2. 모델 자유 선택**
- AWS 계정 하나로 Bedrock 카탈로그 전체 접근:

| 모델 | 제공사 | 특징 |
|------|--------|------|
| Claude Sonnet 4.5 / Opus 4.6 | Anthropic | 최고 성능 |
| Amazon Nova 2 Lite | Amazon | Claude 대비 **73-90% 저렴** |
| DeepSeek R1 | DeepSeek | 오픈소스, 추론 강점 |
| Llama 3.3 70B | Meta | 오픈소스 |
| Kimi K2.5 | Moonshot AI | 262K 컨텍스트 |

**3. 통합 빌링**
- 모든 LLM 비용이 하나의 AWS 청구서로. AWS 크레딧 적용 가능.

**4. 비용 절감 사례**
- 한 개발자가 Claude Sonnet 직접 사용 ~$1,000/월 → Amazon Q Developer Pro 경유 **~$20/월**로 절감

#### AWS의 전략적 의도

`aws-samples` 공식 레포라는 건 AWS가 **Bedrock으로 트래픽을 유도하려는 전략적 목적**이 있다는 의미입니다. OpenClaw 유저가 Anthropic API 직접 호출 대신 Bedrock을 경유하면 AWS가 이득. Kiro IDE 통합, AgentCore Runtime 지원까지 포함된 **멀티 제품 쇼케이스**입니다.

#### 보안: 인프라는 좋지만 앱은 여전히 취약

**AWS가 개선하는 것:**
- IAM → 자격증명 파일 노출 위험 제거
- VPC 엔드포인트 → 네트워크 트래픽 격리
- CloudTrail → 이상 행동 감사
- Security Group → 불필요한 포트 차단

**AWS가 못 고치는 것:**
- CVE-2026-25253 (1-click RCE) — 앱 레벨 버그
- 프롬프트 인젝션 — 이메일의 악성 프롬프트가 API 키 탈취 시연됨
- ClawHub 악성 스킬 (~20%) — 공급망 공격
- 512개 취약점 중 8개 Critical

#### 참고 링크

- [GitHub — aws-samples/sample-OpenClaw-on-AWS-with-Bedrock](https://github.com/aws-samples/sample-OpenClaw-on-AWS-with-Bedrock)
- [Amazon Bedrock — OpenClaw 공식 문서](https://docs.openclaw.ai/providers/bedrock)
- [DEV Community — $1k에서 $20/월로 절감 사례](https://dev.to/aws-builders/i-squeezed-my-1k-monthly-openclaw-api-bill-with-20month-in-aws-credits-heres-the-exact-setup-3gj4)

---

### 8.3 KimiClaw — Cloudflare 서버리스 + Moonshot AI 공식

KimiClaw는 **두 개의 서로 다른 프로젝트**가 같은 이름을 사용합니다.

#### A. 커뮤니티 포크 (claudedjale/KimiClaw)

| 항목 | 내용 |
|------|------|
| **GitHub** | [claudedjale/KimiClaw](https://github.com/claudedjale/KimiClaw) |
| **Stars/Forks** | 1 star / **1,600 forks** |
| **언어** | TypeScript |
| **인프라** | Cloudflare Workers (서버리스 엣지) |
| **비용** | ~$5/월 (Cloudflare Workers 플랜) |
| **라이선스** | MIT |

Cloudflare의 "MoltWorker" 프로젝트를 포크하여 Kimi 모델에 최적화. **서버가 아예 없는** 유일한 OpenClaw 변종입니다.

| 특징 | 내용 |
|------|------|
| 배포 | Cloudflare Workers, ~5분 설정, 300+ 글로벌 엣지 |
| 채널 | Telegram, Discord, Slack, WhatsApp Business, 웹 UI |
| 저장 | Cloudflare R2 (암호화 저장) |
| 브라우저 | Cloudflare Browser Rendering (CDP 기반) |
| 모델 | Kimi K2.5 (기본) + DeepSeek, Claude, GPT, OpenAI 호환 API |
| 보안 | 게이트웨이 토큰 + Cloudflare Access SSO + 디바이스 페어링 + R2 암호화 |

1 star vs 1,600 forks의 극단적 비율은 — 사람들이 조용히 포크해서 자기 인스턴스를 배포하고 있다는 의미입니다.

#### B. 공식 제품 (Moonshot AI)

| 항목 | 내용 |
|------|------|
| **사이트** | [kimi-claw.com](https://kimi-claw.com/) |
| **만든 곳** | Moonshot AI (베이징, 싱가포르 법인, 알리바바 클라우드) |
| **출시** | 2026.2.15 |
| **모델** | Kimi K2.5 Thinking |

| 특징 | 내용 |
|------|------|
| 설정 | 원클릭, 1분, 터미널 불필요 |
| 스킬 | 5,000+ ClawHub 커뮤니티 스킬 (셀프 호스팅 대비 7배) |
| 스토리지 | 40GB 클라우드 |
| 검색 | 프로급 검색 — Yahoo Finance, 뉴스, 기술 문서 실시간 데이터 |
| 스케줄링 | 자동 예약 작업 |
| 가격 | Allegretto 멤버십 이상 필요 |
| 기존 연결 | "Bring Your Own Claw" — 기존 로컬 OpenClaw 인스턴스를 kimi.com에 브릿지 가능 |

#### 데이터 주권 경고

공식 Kimi Claw는 데이터가 **Moonshot AI 서버(알리바바 클라우드)**에 저장됩니다. Hacker News 커뮤니티에서 "CCP 가시성을 전제하라"는 경고가 나왔습니다. 민감한 데이터를 다루는 경우 주의가 필요합니다.

#### 참고 링크

- [GitHub — claudedjale/KimiClaw](https://github.com/claudedjale/KimiClaw)
- [Kimi Claw 공식 소개](https://www.kimi.com/resources/kimi-claw-introduction)
- [MarkTechPost — Moonshot AI Launches Kimi Claw](https://www.marktechpost.com/2026/02/15/moonshot-ai-launches-kimi-claw-native-openclaw-on-kimi-com-with-5000-community-skills-and-40gb-cloud-storage-now/)
- [Hacker News 토론](https://news.ycombinator.com/item?id=47023633)
- [Cloudflare — Introducing Moltworker](https://blog.cloudflare.com/moltworker-self-hosted-ai-agent/)

---

### 클라우드 배포 방식 비교표

| | serverless-openclaw | AWS Bedrock 버전 | KimiClaw (포크) | KimiClaw (공식) |
|---|---|---|---|---|
| **인프라** | AWS (Lambda+Fargate) | AWS (EC2+Bedrock) | Cloudflare Workers | Moonshot AI 클라우드 |
| **월 비용** | ~$1 | ~$46 | ~$5 | 멤버십 |
| **서버 관리** | 없음 | EC2 관리 필요 | 없음 | 없음 |
| **모델** | OpenClaw 기본 | Bedrock 전체 (Nova, DeepSeek, Llama, Claude) | Kimi + 멀티모델 | Kimi K2.5 |
| **인증** | Cognito JWT | IAM 역할 | CF Access + 토큰 | Moonshot 계정 |
| **데이터 주권** | AWS 리전 선택 | AWS 리전 선택 | Cloudflare 글로벌 | **중국 기업 서버** |
| **OpenClaw 앱 보안** | 미해결 | 미해결 | 미해결 | 미해결 |

> **공통 한계**: 4가지 모두 인프라 레벨의 개선이며, OpenClaw 자체의 앱 레벨 취약점(CVE, 프롬프트 인젝션, 악성 스킬)은 해결하지 못합니다. 근본적 보안이 필요하면 IronClaw나 NanoClaw처럼 아키텍처 자체가 다른 도구를 선택해야 합니다.

---

## 9. OpenFang — Rust (Agent OS)

| 항목 | 내용 |
|------|------|
| **유형** | Agent OS (에이전트 운영체제) |
| **언어** | Rust (14개 크레이트, 137K LOC) |
| **라이선스** | Apache 2.0 |
| **GitHub Stars** | N/A (최신 릴리스) |

### 핵심 컨셉

**"Agent OS"** — AI 에이전트를 위한 완전한 운영체제. 단순한 프레임워크가 아니라 에이전트 실행 환경 전체를 포괄하는 설계.

- 기술 스택: Rust + tokio 비동기 + SQLite + WASM 샌드박스
- Rust 14개 크레이트 구조로 모듈화된 대규모 코드베이스 (137K LOC)

### 핵심 기능

| 기능 | 내용 |
|------|------|
| **빌트인 도구** | 60개 내장 도구, WASM 샌드박스 실행 |
| **채널 어댑터** | 40개 메시징 플랫폼 어댑터 |
| **보안 아키텍처** | 16-레이어 보안 + Taint Tracking (오염 추적) |
| **프로토콜** | MCP 양방향 + A2A (Agent-to-Agent) 프로토콜 지원 |
| **지식 그래프** | 중요도 점수 기반 Knowledge Graph |
| **브라우저 자동화** | 네이티브 CDP 기반, 50개+ 기능 |
| **플러그인 생태계** | "Hands" 시스템 + FangHub 마켓플레이스 |
| **자율 실행** | 24/7 자율 실행 + Soul Snapshot (상태 스냅샷) |
| **컨텍스트 관리** | 3-레이어 컨텍스트 윈도우 관리 |
| **모델 오버라이드** | 채널별 독립 모델 설정 |

### 차별점

- **16-레이어 보안**: Taint Tracking으로 데이터 흐름을 추적하며 오염된 데이터가 민감 경로에 유입되는 것을 방지
- **FangHub 마켓플레이스**: OpenClaw의 ClawHub에 대응하는 플러그인 생태계("Hands" 시스템)
- **A2A 프로토콜**: 에이전트 간 직접 통신 지원 (MCP와 병행)
- **Soul Snapshot**: 24/7 자율 실행을 위한 에이전트 상태 영속성

---

## 10. OpenJarvis — Python + Rust (로컬 퍼스트 개인 AI)

| 항목 | 내용 |
|------|------|
| **유형** | 개인 AI 프레임워크 (로컬 퍼스트) |
| **언어** | Python + Rust 확장 |
| **라이선스** | MIT |
| **개발** | Stanford (스탠퍼드) |

### 핵심 컨셉

**"Personal AI, On Personal Devices"** — 클라우드가 아닌 개인 디바이스에서 실행되는 로컬 퍼스트 AI 에이전트 프레임워크.

- 기술 스택: Python 3.10+ + Rust + FastAPI + SQLite

### 9가지 에이전트 유형

| 에이전트 유형 | 특징 |
|--------------|------|
| Simple | 기본 단일 에이전트 |
| Orchestrator | 다중 에이전트 조율 |
| ReAct | Reasoning + Acting 루프 |
| OpenHands | 오픈핸즈 호환 |
| RLM | 강화학습 기반 |
| ClaudeCode | Claude Code CLI 통합 |
| Operative | 자율 실행 특화 |
| MonitorOperative | 모니터링 포함 자율 실행 |
| NativeOpenHands | 네이티브 OpenHands 통합 |

### 핵심 기능

| 기능 | 내용 |
|------|------|
| **빌트인 도구** | 24개+ |
| **메시징 채널** | 24개+ (Telegram, Discord, Slack, WhatsApp, LINE, Teams 등) |
| **메모리 백엔드** | 5종 (SQLite/FTS5, FAISS, ColBERTv2, BM25, Hybrid) |
| **추론 엔진** | 6개+ (Ollama, vLLM, SGLang, llama.cpp, MLX, 클라우드) |
| **보안** | GuardrailsEngine + SecretScanner + PIIScanner |
| **스케줄러** | cron/interval/once 방식 자율 태스크 |
| **샌드박스** | Docker/Podman 컨테이너 격리 |
| **API 서버** | OpenAI 호환 API 서버 내장 |

### 차별점

- **Trace-Driven Learning Loop**: 실행 트레이스를 학습해 라우팅을 최적화하는 자기 개선 루프
- **하드웨어 인식 엔진 선택**: GPU/CPU/Apple Silicon 등 실행 환경에 따라 추론 엔진을 자동 선택
- **PIIScanner**: 개인 식별 정보(PII)를 자동 탐지해 외부 유출 방지
- **로컬 퍼스트 철학**: 6개 로컬 추론 엔진 지원으로 클라우드 의존성 최소화

---

## 11. NemoClaw (NVIDIA) — OpenClaw 샌드박스 플러그인

| 항목 | 내용 |
|------|------|
| **유형** | OpenClaw 샌드박스 플러그인 (GPU 최적화) |
| **언어** | JavaScript / TypeScript / Python / Shell (25,650 LOC) |
| **라이선스** | Apache 2.0 |
| **개발** | NVIDIA |

### 핵심 컨셉

**샌드박스 상시 실행 에이전트 + NVIDIA GPU 최적화 추론** — OpenClaw를 샌드박스로 격리하면서 NVIDIA 인프라(NIM, DGX, Brev)와 긴밀하게 통합하는 엔터프라이즈 플러그인.

- 기술 스택: Node.js + TypeScript + Python + Docker + OpenShell

### 핵심 기능

| 기능 | 내용 |
|------|------|
| **플러그인 아키텍처** | `openclaw.plugin.json` 기반 OpenClaw 플러그인 |
| **4-레이어 샌드박스 보안** | 네트워크 / 파일시스템 / 프로세스 / 추론 레이어 격리 |
| **Blueprint 버전 관리** | OCI 레지스트리 + 다이제스트 검증 |
| **GPU 감지** | NVIDIA, Apple Silicon, DGX Spark 자동 감지 |
| **NIM 컨테이너 관리** | 로컬 추론을 위한 NVIDIA NIM 컨테이너 |
| **추론 프로파일** | 4종 (NVIDIA 클라우드, NCP, nim-local, vLLM) |
| **엔터프라이즈 커넥터** | 10개 프리셋, 핫 리로드 지원 |
| **온보딩 위저드** | 7단계 대화형 온보딩 |
| **마이그레이션** | 호스트 OpenClaw → 샌드박스 자동 마이그레이션 |
| **이그레스 승인** | 알 수 없는 외부 연결에 대한 오퍼레이터 승인 플로우 |
| **원격 GPU** | Brev 배포를 통한 원격 GPU 활용 |

### 4종 추론 프로파일

| 프로파일 | 설명 |
|----------|------|
| `nvidia-cloud` | NVIDIA 클라우드 API 경유 |
| `ncp` | NVIDIA Cloud Partner 경유 |
| `nim-local` | 로컬 NIM 컨테이너 실행 |
| `vllm` | vLLM 엔진 직접 구동 |

### 차별점

- **OpenClaw 플러그인으로 동작**: 독립 프레임워크가 아닌 기존 OpenClaw 인프라를 샌드박스화하는 방식
- **Operator Approval Flow**: 에이전트가 알 수 없는 외부 도메인에 접근하려 할 때 오퍼레이터 수동 승인 필요
- **Blueprint OCI 검증**: 컨테이너 이미지의 무결성을 OCI 다이제스트로 검증해 공급망 공격 방어
- **NVIDIA 생태계 통합**: DGX Spark, Brev 원격 GPU, NIM 로컬 모델까지 NVIDIA 전체 스택 지원

---

## 12. NullClaw — Zig (극한 최소화)

| 항목 | 내용 |
|------|------|
| **GitHub** | [nullclaw/nullclaw](https://github.com/nullclaw/nullclaw) |
| **Stars** | 6,400 |
| **언어** | Zig 0.15.2 |
| **라이선스** | MIT |
| **코드량** | ~249,000 LOC |
| **바이너리** | 678 KB 정적 |
| **시작 시간** | <2ms |
| **RAM** | ~1 MB 피크 |

### 핵심 특징

- **생태계 최소 바이너리**: 678 KB — ZeroClaw(8.8MB)보다 13배 작음
- 50+ AI 프로바이더, **19개 메시징 채널** (Signal, Nostr, Matrix 포함 — 생태계 유일)
- 35+ 도구, **10개 메모리 엔진** (SQLite/PostgreSQL/Redis/ClickHouse + 벡터/하이브리드)
- **WASI 타겟** (`main_wasi.zig`) — WebAssembly 배포 지원 (10개 프레임워크 중 유일)
- **A2A 프로토콜** (`a2a.zig`) — OpenFang, OpenJarvis와 동일
- **vtable 기반 플러그어블 아키텍처** — 설정만으로 모든 서브시스템 교체

### 보안 모델

- **ChaCha20-Poly1305** 암호화 (ZeroClaw와 동일 알고리즘)
- **Landlock/Firejail/Bubblewrap/Docker** 샌드박스 옵션 (OS 레벨 격리)
- 게이트웨이 페어링 필수 (원타임 코드), localhost 기본 바인딩
- 채널 **deny-all 기본** — 명시적 opt-in 필요

### 차별점

| 특성 | NullClaw | 비교 |
|------|----------|------|
| **언어** | Zig (유일) | 전체 생태계 유일 |
| **바이너리** | 678 KB | ZeroClaw 8.8MB의 1/13 |
| **WASI** | [O] | 10개 프레임워크 중 유일 |
| **Nostr/Matrix** | [O] | 10개 프레임워크 중 유일 |

### 참고 링크

- [GitHub — nullclaw/nullclaw](https://github.com/nullclaw/nullclaw)

---

## 13. Hermes Agent — Python (자기개선 에이전트)

| 항목 | 내용 |
|------|------|
| **GitHub** | [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) |
| **Stars** | 9,269 |
| **언어** | Python |
| **라이선스** | MIT |
| **생성일** | 2025-07-22 |
| **빌더** | Nous Research |

### 핵심 특징

- **자기개선 루프**: 복잡한 작업 후 스킬 자동 생성, 사용 중 자동 개선 (agentskills.io 표준)
- **이중 메모리**: MEMORY.md (에이전트 노트) + USER.md (사용자 프로파일), **Frozen Snapshot 패턴** (prefix cache 보존)
- **Skills Guard**: builtin/trusted/community/agent-created 4단계 신뢰 정책 + 정적 분석 스캐너
- **Tirith pre-exec 스캐너**: 외부 바이너리, SHA-256 + cosign 서명 검증, 자동 설치
- **6종 터미널 백엔드**: local, Docker, SSH, Daytona, Singularity, Modal (서버리스)
- **6종 메시징 플랫폼**: Telegram, Discord, Slack, WhatsApp, Signal, HomeAssistant
- **위임 아키텍처**: MAX_DEPTH=2, MAX_CONCURRENT=3, blocked tools 명시, ThreadPoolExecutor
- **RL**: Atropos 환경, OPD 환경, SWE 환경, 궤적 압축
- **Topics에 openclaw/clawdbot/moltbot 포함** — Claw 생태계 직접 연결

### 신규 패턴 (R17–R22)

| 패턴 | 설명 |
|------|------|
| R17: Frozen Snapshot Memory | 세션 시작 시 메모리 1회 스냅샷, 세션 중 불변 → prefix cache 안정화 |
| R18: Char-Limited Memory | 토큰이 아닌 문자 단위 예산 (model-agnostic) |
| R19: Memory Injection Detection | 항목 추가 전 regex + 비가시 유니코드 스캔 |
| R20: Skills Trust Levels | 4단계 신뢰 정책 + agentskills.io 오픈 스탠다드 |
| R21: Bounded Delegation Tree | MAX_DEPTH=2, blocked tools, MAX_CONCURRENT=3 |
| R22: Tirith Pre-Exec Scanner | 외부 바이너리 스캐너, SHA-256 + cosign 서명 검증 |

### 참고 링크

- [GitHub — NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- [상세 보고서](details/hermes_agent_report.md)

---

## 14. Claude Code — JavaScript (공식 플랫폼)

| 항목 | 내용 |
|------|------|
| **패키지** | `@anthropic-ai/claude-code` |
| **버전** | 2.1.80 (분석 시점) |
| **채널 플러그인** | [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) |
| **언어** | JavaScript (Node.js 번들) + TypeScript (채널, Bun 런타임) |
| **번들 크기** | cli.js 12MB (minified) |
| **라이선스** | 독점 (CLI) / 오픈소스 (채널 플러그인) |
| **개발 팀** | Anthropic |
| **내부 코드명** | `tengu_harbor` |

### 핵심 특징

- **MCP-as-Channel Bridge**: `notifications/claude/channel` 단일 MCP notification으로 외부 메시징 플랫폼 연결 (R23)
- **5단계 게이트**: capability → 피처 플래그 → claude.ai 인증 → 조직 정책 → --channels 리스트 순차 검증
- **Platform-Controlled Allowlist**: Anthropic이 `tengu_harbor_ledger` 피처 플래그로 허용 채널 원격 제어 (R24)
- **Content/Meta 분리**: 첨부파일 경로 등 메타데이터를 content가 아닌 meta 필드에만 포함해 위조 차단 (R25)
- **5겹 프롬프트 인젝션 방어**: 메타 격리 + 시스템 프롬프트 경고 + 스킬 설계 + 아웃바운드 게이트 + 파일 exfil 방지
- **공식 채널 2개**: Telegram (grammy, Long Polling) + Discord (discord.js, Gateway WebSocket)
- **Static 모드**: CI/서버리스 환경용 — 부팅 시 access.json 스냅샷, 이후 파일 I/O 없음

### 신규 패턴 (R23–R25)

| 패턴 | 설명 |
|------|------|
| R23: MCP-as-Channel Bridge | MCP notification 단일 메서드로 모든 외부 메시징 플랫폼 표준화 |
| R24: Platform-Controlled Allowlist | 플랫폼 벤더가 피처 플래그로 허용 채널 원격 제어 (셀프호스팅과 다른 신뢰 모델) |
| R25: Content/Meta Channel Separation | 메타데이터와 사용자 입력을 구조적으로 분리해 위조 공격 차단 |
| R26: Bundled OS-Level Sandbox | npm vendor에 seccomp BPF 필터 + apply-seccomp 바이너리 동봉. 시스템 의존성 없이 OS 레벨 격리 완결 |

### 참고 링크

- [Claude Code 채널 공식 문서](https://code.claude.com/docs/en/channels)
- [상세 보고서](details/claude_code_report.md)

---

## 15. GoClaw — Go (멀티테넌트 AI 게이트웨이)

| 항목 | 내용 |
|------|------|
| **GitHub** | [nextlevelbuilder/goclaw](https://github.com/nextlevelbuilder/goclaw) |
| **Stars** | 1,400+ |
| **언어** | Go 1.26 |
| **LOC** | 약 176,000줄 |
| **라이선스** | CC BY-NC 4.0 (비상업적) |
| **커밋** | 832+ |

### 핵심 특징

OpenClaw를 Go로 재구현한 멀티테넌트 AI 에이전트 게이트웨이. PostgreSQL 기반 완전한 테넌트 격리, Docker 3축(mode/access/scope) 샌드박스, AES 암호화, Tailscale VPN 내장, OTel 코어 의존성을 조합한 엔터프라이즈급 단일 바이너리다. 20+ LLM 제공자, 7개 메시징 채널, Wails v2 데스크톱 UI를 지원한다.

### 신규 패턴 (R38–R39)

| 패턴 | 설명 |
|------|------|
| R38: 3축 샌드박스 아키텍처 | Docker를 Mode(off/non-main/all) × Access(none/ro/rw) × Scope(session/agent/shared) 3축으로 제어 |
| R39: VPN-native 에이전트 게이트웨이 | Tailscale tsnet으로 에이전트 게이트웨이 자체를 VPN 노드로 등록 |

### 참고 링크

- 상세 보고서: [details/goclaw_report.md](details/goclaw_report.md)

---

## 16. CoPaw — Python (14채널 개인 비서)

| 항목 | 내용 |
|------|------|
| **GitHub** | [agentscope-ai/CoPaw](https://github.com/agentscope-ai/CoPaw) |
| **Stars** | 13,600+ |
| **언어** | Python 3.10-3.13 |
| **LOC** | 약 84,733줄 |
| **라이선스** | Apache-2.0 |
| **기반** | AgentScope 1.0.18 |

### 핵심 특징

AgentScope 위에서 동작하는 standalone 개인 AI 비서. Claw 생태계 최다 채널(14개: DingTalk, Feishu, QQ, Xiaoyi, Telegram, WeCom, Mattermost, WeChat, Discord, iMessage, Matrix, MQTT, Voice, Console)을 지원한다. 3-tuple QueueKey `(channel_id, session_id, priority_level)` 기반 UnifiedQueueManager로 head-of-line blocking을 구조적으로 제거하며, Playwright 브라우저 자동화(3,460줄), Skills 시스템, 3단계 보안 스캐닝(tool_guard, file_access_guard, skill_scanner)을 내장한다.

### 신규 패턴 (R42)

| 패턴 | 설명 |
|------|------|
| R42: 3-tuple QueueKey 채널-세션-우선순위 격리 | (channel_id, session_id, priority_level) 키로 메시지 큐를 분리; 동적 consumer 생성, 유휴 큐 자동 정리 |

### 참고 링크

- 상세 보고서: [details/copaw_report.md](details/copaw_report.md)

---

## 종합 비교표

| 도구 | 언어 | 코드량 | RAM | 시작시간 | 핵심 차별점 | Stars |
|------|------|--------|-----|----------|-------------|-------|
| **OpenClaw** | TS/Node.js | 430,000+ | >1GB | 수분 | 원조, 풀피처, 12+ 채널 | 223K |
| **Nanobot** | Python | ~4,000 | ~100MB | ~30s | 초경량, 연구 친화, 에이전트 커널 | 24K |
| **NanoClaw** | TypeScript | ~500 (코어) | - | - | 컨테이너 격리, 에이전트 스웜, 포크 철학 | 13.5K |
| **IronClaw** | Rust | - | - | - | WASM 샌드박스, 자격증명 보호, 유출 방어 | 3.3K |
| **ZeroClaw** | Rust | - | <5MB | <10ms | 트레이트 아키텍처, $10 HW, 제로 오버헤드 | 17K |
| **PicoClaw** | Go | - | <10MB | <1s | 구형 안드로이드, RISC-V, 95% AI 생성 | 17-19K |
| **TinyClaw** | TypeScript | ~400 (원본) | - | - | 멀티에이전트 팀, 체인 실행, TUI 대시보드 | - |
| **OpenFang** | Rust | 137K LOC | - | - | Agent OS, 16-레이어 보안, 40채널, FangHub | N/A |
| **OpenJarvis** | Python+Rust | - | - | - | 로컬 퍼스트, 9 에이전트 유형, 6 추론 엔진 | - |
| **NemoClaw** | JS/TS/Py | 25,650 | - | - | NVIDIA GPU, OpenClaw 샌드박스 플러그인 | - |
| **NullClaw** | Zig | ~249K | ~1MB | <2ms | 678KB 정적 바이너리, WASI, Signal+Nostr+Matrix | 6.4K |
| **Hermes Agent** | Python | - | - | - | 자기개선 루프, Frozen Snapshot 메모리, Skills Trust, Tirith 스캐너 | 9.3K |
| **Claude Code** | JS+TS | 12MB (번들) | - | - | MCP 채널 표준, seccomp+bwrap 내장 샌드박스, Platform Allowlist, 5겹 인젝션 방어 | N/A (독점) |
| **GoClaw** | Go 1.26 | 176K LOC | - | <1s | 멀티테넌트 PostgreSQL, 3축 Docker 샌드박스, AES 암호화, Tailscale VPN, OTel 내장 | 1.4K |
| **CoPaw** | Python 3.10-3.13 | 84K LOC | - | - | 14채널(최다), 3-tuple QueueKey 격리, Playwright 브라우저, Skills 시스템, 3단계 보안 스캔 | 13.6K |

### 차별화 축 매핑

```
                    보안 강화
                       |
              NullClaw-+IronClaw
              (Landlock)|
       NanoClaw -------+------- (원조) OpenClaw
       (격리)          |              (풀피처)
                       |
    경량화 ----+-------+--------+---- 협업/오케스트레이션
               |       |        |
           ZeroClaw    |    TinyClaw
           NullClaw    |    (멀티팀)
           PicoClaw    |
           (엣지)      |
                       |
                   Nanobot
                   (단순/연구)
```

---

## 핵심 인사이트

### 1. 보안이 혁신을 촉발

OpenClaw의 심각한 보안 문제(CVE-2026-25253, ClawHavoc 공급망 공격)가 대안 생태계 폭발의 **직접적 원인**입니다. 특히 IronClaw와 NanoClaw는 보안 문제를 명시적으로 해결하기 위해 탄생했습니다.

### 2. "Claw" = Claude

모든 이름의 "Claw"는 Anthropic의 **Claude**에서 유래합니다. ClawdBot → Claude Bot의 변형이 시초이며, 이후 "Claw"가 AI 에이전트를 지칭하는 보통명사처럼 사용되고 있습니다.

### 3. 각기 다른 절충점

| 프로젝트 | 핵심 절충 |
|----------|-----------|
| OpenClaw | 기능 풍부 vs 복잡성/보안 취약 |
| Nanobot | 단순성/가독성 vs 기능 제한 |
| NanoClaw | 컨테이너 격리 vs 설정 편의성 |
| IronClaw | 보안 극대화 vs 생태계 크기 |
| ZeroClaw | 성능 극대화 vs 아직 초기 |
| PicoClaw | 하드웨어 접근성 vs 프로덕션 미성숙 |
| TinyClaw | 팀 협업 vs 채널 다양성 |

### 4. Rust의 부상

7개 프로젝트 중 2개(IronClaw, ZeroClaw)가 **Rust**로 작성되었습니다. 메모리 안전성 + 네이티브 성능이 AI 에이전트 인프라에서 중요한 경쟁력으로 부상하고 있습니다.

### 5. 오픈소스 AI 에이전트의 미래

> "AI 어시스턴트의 미래는 오픈 소스이며, 포크 가능하고, 무엇이든에서 실행된다."

포크 가능하고, 어디서든 실행되며, 다양한 철학이 공존하는 생태계로 진화 중입니다. 단일 "최고의" 솔루션이 아닌, 각자의 요구에 맞는 도구를 선택하는 시대가 도래했습니다.

### 6. 자격증명 딜레마 — "권한을 줘야 일을 하는데, 주면 위험하다"

AI 비서가 진짜 일을 하려면 결국 각종 토큰과 접근 권한이 필요합니다:

- 이메일 보내려면 → Gmail 토큰
- 코드 푸시하려면 → GitHub 토큰
- 일정 관리하려면 → 캘린더 API 키
- 메시지 보내려면 → Slack/Telegram 토큰

**권한을 안 주면 무능하고, 주면 위험합니다.** 이것이 현재 AI 에이전트 생태계의 근본적 딜레마입니다.

차이는 **"어떻게 주느냐"**에 있습니다:

| 방식 | 도구 | 위험도 | 비유 |
|------|------|--------|------|
| LLM에게 토큰을 그냥 넘김 | **OpenClaw** | 높음 | 비서에게 신용카드를 줌. 번호를 외울 수 있음 |
| LLM은 토큰을 모름, 실행 시점에만 호스트가 주입 | **IronClaw** | 낮음 | 비서가 "결제해주세요"라고 하면, 당신이 카드를 대신 긁어줌. 비서는 카드번호를 모름 |
| 컨테이너 안에서만 작동 | **NanoClaw** | 중간 | 비서를 잠긴 방에 넣고 일시킴. 도망 못 감 |
| 토큰 자체를 안 씀 | **Nanobot/ZeroClaw** | 최저 | 자동화 범위가 제한되지만 유출 위험도 최소 |

핵심 차이점은 **"LLM이 토큰 값 자체를 직접 아느냐"**입니다:

- **OpenClaw**: LLM이 환경변수/시크릿에 직접 접근 가능 → 프롬프트 인젝션 한 방이면 유출
- **IronClaw**: LLM이 "GitHub에 푸시해"라고 요청 → 호스트가 토큰을 붙여서 실행 → LLM은 토큰 값 자체를 절대 모름
- **NanoClaw**: 유출되어도 컨테이너 밖으로 나갈 수 없음

IronClaw의 "호스트 경계 주입" 방식이 현재로선 가장 합리적인 타협점이며, 메인 맥북처럼 개인 자격증명이 많은 환경에서는 이 차이가 특히 중요합니다.

---

*이 보고서는 각 프로젝트의 공식 GitHub 레포지토리, 기술 문서, 커뮤니티 토론(Hacker News), 미디어 보도를 기반으로 작성되었습니다.*
