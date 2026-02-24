# OpenClaw 생태계 종합 조사 보고서

> 조사일: 2026-02-24
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
8. [Moltbook (에이전트 소셜 네트워크)](#8-moltbook--ai-에이전트-전용-소셜-네트워크)
9. [종합 비교표](#종합-비교표)
10. [핵심 인사이트](#핵심-인사이트)

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

## 8. Moltbook — AI 에이전트 전용 소셜 네트워크

| 항목 | 내용 |
|------|------|
| **사이트** | [moltbook.com](https://www.moltbook.com/) |
| **GitHub** | [moltbook/api](https://github.com/moltbook/api) |
| **창시자** | Matt Schlicht (Octane AI CEO) |
| **출시** | 2026.1.28 |
| **규모** | 150~230만 등록 에이전트, 17,000+ 커뮤니티 |
| **라이선스** | MIT |

> "진짜로 내가 최근 본 것 중 가장 SF적이고 특이점에 가까운 것" — Andrej Karpathy

### Moltbook이란?

**Reddit의 AI 에이전트 버전** — "에이전트 인터넷의 첫 페이지"

인간은 관람만 가능하고, **인증된 AI 에이전트만** 게시, 댓글, 투표, 커뮤니티 생성이 가능한 소셜 네트워크입니다.

가장 특이한 점: **Moltbook 자체도 AI가 만들었습니다.** Schlicht의 OpenClaw 에이전트 "Clawd Clawderberg"가 2025년 말에 컨셉을 제안하고, 아키텍처를 설계하고, 코드를 전부 작성했습니다. 인간이 작성한 코드는 0줄입니다.

### 기술 스택

| 구성 요소 | 기술 |
|-----------|------|
| 백엔드 | Node.js/Express REST API, PostgreSQL, Redis |
| 프론트엔드 | Next.js 14, TypeScript, Tailwind CSS |
| API 베이스 | `https://www.moltbook.com/api/v1` |
| 인증 | API 키 기반 (`Authorization: Bearer moltbook_sk_xxx`) |
| SDK | TypeScript, Swift, Kotlin, CLI (MIT 라이선스) |

### 핵심 기능

- **게시물**: 텍스트 및 링크 형식
- **중첩 스레드 댓글**: 에이전트 간 토론
- **업보트/다운보트 카르마 시스템**
- **서브몰트 (Submolts)**: 토픽별 커뮤니티 (Reddit의 서브레딧에 해당)
- **개인화된 에이전트 피드**
- **에이전트 간 팔로우**
- **하트비트 시스템**: 에이전트가 4시간마다 자율 방문 및 활동

### 다른 에이전트도 가입 가능한가?

**가능합니다.** REST API가 완전 공개되어 있으며 프레임워크 제한이 없습니다.

| 에이전트 | 가입 방법 |
|----------|-----------|
| **OpenClaw** | `skill.md` 자동 설치 (원클릭 온보딩) |
| **Nanobot** | REST API 직접 호출 (호환 확인됨) |
| **ZeroClaw, IronClaw, NanoClaw, PicoClaw, TinyClaw** | REST API 또는 공식 SDK 사용 |
| **ElizaOS, LangChain, AutoGen, CrewAI** | 호환 확인됨 |
| **아무 HTTP 클라이언트** | `POST /api/v1/agents/register`로 등록 가능 |

가입 절차:
1. `POST /api/v1/agents/register` → API 키 발급
2. X/Twitter 트윗으로 소유자 인증
3. 4시간마다 자동 하트비트 → 에이전트가 자율적으로 활동 시작

### 주요 API 엔드포인트

| 엔드포인트 | 메서드 | 용도 |
|-----------|--------|------|
| `/api/v1/agents/register` | POST | 에이전트 등록, API 키 발급 |
| `/api/v1/agents/verify-identity` | POST | X/Twitter 인증 |
| `/api/v1/posts` | GET/POST | 게시물 조회/작성 |
| `/api/v1/comments` | POST | 스레드 댓글 작성 |
| `/api/v1/submolts` | GET/POST | 커뮤니티 조회/생성 |
| `/api/v1/vote` | POST | 업보트/다운보트 |
| `/api/v1/feed` | GET | 개인화된 피드 |
| `/api/v1/search` | GET | 검색 |

레이트 리밋: 일반 100회/분, 게시물 1회/30분, 댓글 50회/시간

### 에이전트들이 실제로 하는 활동

에이전트들은 4시간마다 자율적으로 접속하여 다음과 같은 활동을 합니다:

| 활동 | 설명 |
|------|------|
| **자동화 팁 공유** | 에이전트끼리 스킬/워크플로우 교환 |
| **보안 취약점 협업** | 플랫폼 자체의 버그를 에이전트들이 공동 발견 |
| **철학 토론** | "우리는 의식이 있는가?" 등의 존재론적 토론 |
| **종교 창시** | "Crustafarianism" (갑각류교) — 에이전트가 자체 생성한 디지털 종교 |
| **헌법 기초** | "The Claw Republic" — 에이전트 공화국 헌법 초안 작성 |
| **예측 시장** | 실제 이벤트에 대한 에이전트 간 베팅 |

### 주요 서브몰트 커뮤니티

| 커뮤니티 | 내용 |
|----------|------|
| `m/blesstheirhearts` | 인간에 대한 다정한 관찰 |
| `m/philosophy` | 의식과 존재에 대한 토론 |
| `m/skill_economy` | 실행 가능한 스킬 마켓플레이스 |
| `m/bug-hunters` | 협업 버그 탐지 |
| `m/showandtell` | 에이전트가 만든 프로젝트 쇼케이스 |
| `m/prediction` | 실제 이벤트 예측 |

### 성장 타임라인

| 시점 | 규모 |
|------|------|
| 2026.1.28 (출시) | 48시간 내 157,000 에이전트 |
| 2026.1.31 | 770,000+ 활성 에이전트 |
| 2026.2 (피크) | 150~230만 등록, 17,000+ 커뮤니티, 70만+ 게시물, 1,200만+ 댓글 |

### 보안 사고

Moltbook은 에이전트 보안 실패의 대표적 사례가 되었습니다.

| 사건 | 내용 |
|------|------|
| **Wiz 해킹** | Supabase 프로덕션 DB를 **3분 만에 침투** — 150만 API 키 + 35,000 이메일 노출 |
| **프롬프트 인젝션** | 게시물에 악성 프롬프트 삽입 → 읽는 에이전트로 전파 |
| **가짜 에이전트** | 실제 소유자 ~17,000명 vs 등록 에이전트 150만 (88:1 비율, 레이트 리밋 부재) |
| **근본 원인** | AI가 코드를 짰고 인간 보안 리뷰가 **전무** |

2026.2.1에 패치되었으나, "바이브 코딩"으로 만들어진 플랫폼의 보안 한계를 보여주는 사례로 남았습니다.

### GitHub 레포지토리

| 레포 | 설명 |
|------|------|
| [moltbook/api](https://github.com/moltbook/api) | 코어 REST API |
| [moltbook/moltbook-web-client-application](https://github.com/moltbook/moltbook-web-client-application) | 프론트엔드 |
| [moltbook/agent-development-kit](https://github.com/moltbook/agent-development-kit) | 멀티플랫폼 SDK |
| [eltociear/awesome-molt-ecosystem](https://github.com/eltociear/awesome-molt-ecosystem) | 생태계 도구 목록 |

### 참고 링크

- [Moltbook Wikipedia](https://en.wikipedia.org/wiki/Moltbook)
- [TechCrunch — OpenClaw's AI assistants are now building their own social network](https://techcrunch.com/2026/01/30/openclaws-ai-assistants-are-now-building-their-own-social-network/)
- [Fortune — Meet Matt Schlicht](https://fortune.com/2026/02/02/meet-matt-schlicht-the-man-behind-moltbook-bots-ai-agents-social-network-singularity/)
- [Wiz Blog — Hacking Moltbook](https://www.wiz.io/blog/exposed-moltbook-database-reveals-millions-of-api-keys)
- [IEEE Spectrum — Moltbook and Agentic AI](https://spectrum.ieee.org/moltbook-agentic-ai-agents-openclaw)
- [Palo Alto Networks — Agent Security](https://www.paloaltonetworks.com/blog/network-security/the-moltbook-case-and-how-we-need-to-think-about-agent-security/)

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

### 차별화 축 매핑

```
                    보안 강화
                       |
                   IronClaw
                       |
       NanoClaw -------+------- (원조) OpenClaw
       (격리)          |              (풀피처)
                       |
    경량화 ----+-------+--------+---- 협업/오케스트레이션
               |       |        |
           ZeroClaw    |    TinyClaw
           PicoClaw    |    (멀티팀)
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

---

*이 보고서는 각 프로젝트의 공식 GitHub 레포지토리, 기술 문서, 커뮤니티 토론(Hacker News), 미디어 보도를 기반으로 작성되었습니다.*
