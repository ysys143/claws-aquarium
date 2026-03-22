# NullClaw 상세 분석 보고서

> **소스**: GitHub [nullclaw/nullclaw](https://github.com/nullclaw/nullclaw) 직접 분석
> **조사일**: 2026-03-17
> **언어**: Zig (생태계 유일)

---

## 목차

1. [기본 정보](#기본-정보)
2. [핵심 특징](#핵심-특징)
3. [아키텍처](#아키텍처)
4. [보안 모델](#보안-모델)
5. [메모리 시스템](#메모리-시스템)
6. [채널 및 도구](#채널-및-도구)
7. [차별점](#차별점)
8. [한계](#한계)
9. [참고 링크](#참고-링크)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub** | [nullclaw/nullclaw](https://github.com/nullclaw/nullclaw) |
| **Stars** | 6,400 |
| **언어** | Zig 0.15.2 (정확한 버전 필요) |
| **라이선스** | MIT |
| **코드량** | ~249,000 LOC (~250 소스 파일) |
| **테스트** | 5,300+ |
| **바이너리** | 678 KB 정적 바이너리 |
| **시작 시간** | <2ms |
| **RAM** | ~1 MB 피크 |
| **외부 의존성** | libc만 (SQLite 선택적) |

---

## 핵심 특징

> "678 KB binary · <2 ms startup · 5,300+ tests · 50+ providers · 19 channels · Pluggable everything."

- **생태계 최소 바이너리**: 678 KB 정적 바이너리 — ZeroClaw(8.8 MB)보다 **13배 작음**
- **리소스**: <2ms 시작, ~1MB RAM — $5 보드에서도 동작
- **50+ AI 프로바이더**: OpenRouter, Anthropic, Ollama, OpenAI 호환 전체
- **19개 메시징 채널**: Telegram, Discord, Signal, Slack, Nostr, Matrix, Voice 등
- **35+ 도구** 내장
- **10개 메모리 엔진**: SQLite, PostgreSQL, Redis, ClickHouse + 벡터/키워드 하이브리드
- **WASI 타겟**: `main_wasi.zig` — WebAssembly 배포 옵션 (10개 프레임워크 중 유일)
- **A2A 프로토콜**: `a2a.zig` — OpenFang, OpenJarvis와 동일하게 Google A2A 지원

---

## 아키텍처

### vtable 기반 플러그어블 설계

모든 서브시스템이 **vtable 인터페이스**로 추상화됨 — 코드 변경 없이 설정 파일만으로 교체 가능:

| 서브시스템 | 교체 가능한 구현체 예시 |
|-----------|----------------------|
| Provider | Anthropic, OpenAI, Ollama, OpenRouter 등 50+ |
| Memory | SQLite, PostgreSQL, Redis, ClickHouse + 벡터 변형 |
| Security sandbox | Landlock, Firejail, Bubblewrap, Docker |
| Channel | Telegram, Discord, Signal, Nostr, Matrix, Voice 등 19개 |

→ ZeroClaw의 Rust trait 기반 교체 철학과 동일하되, Zig vtable로 구현.

### 소스 구조

```
src/
├── agent/             # 에이전트 루프
├── channels/          # 채널 어댑터
├── memory/            # 메모리 엔진
├── providers/         # LLM 프로바이더
├── security/          # 보안 레이어
├── tools/             # 빌트인 도구
├── a2a.zig            # Google A2A 프로토콜
├── heartbeat.zig      # 24/7 상주 루프
├── mcp.zig            # MCP 통합
├── main_wasi.zig      # WASI (WebAssembly) 타겟
├── skillforge.zig     # 동적 스킬 시스템
├── subagent.zig       # 서브에이전트 지원
├── rag.zig            # RAG 파이프라인
├── cron.zig           # 예약 작업
├── hardware.zig       # 하드웨어 감지 (엣지/IoT)
└── voice.zig          # 음성 채널
```

### 주요 설계 패턴

- **`heartbeat.zig`**: 명시적 24/7 상주 루프 — R9 Sleep Consolidation Loop 패턴
- **`skillforge.zig`**: 동적 스킬 컴파일/로딩 — OpenFang Hands 시스템 유사
- **`hardware.zig`**: 하드웨어 환경 자동 감지 (R10 Intelligence Per Watt 패턴 유사)
- **Nix flake**: `flake.nix`로 재현 가능한 빌드 보장

---

## 보안 모델

**Tier 1급** — ZeroClaw와 동급:

| 보안 영역 | 내용 |
|----------|------|
| **암호화** | **ChaCha20-Poly1305** (ZeroClaw와 동일) |
| **샌드박스** | Landlock, Firejail, Bubblewrap, Docker (설정으로 선택) |
| **인증** | 게이트웨이 원타임 코드 페어링 (기본 필수) |
| **네트워크** | 채널 허용목록 **기본 deny-all** (명시적 opt-in) |
| **파일시스템** | 워크스페이스 스코핑 + 격리 (기본 활성화) |
| **감사 로그** | 보존 기간 설정 가능 |
| **노출 범위** | localhost 기본 바인딩, 터널 명시 설정 필요 |

### 특이점: Landlock 네이티브 지원

- Linux 커널 **Landlock LSM** 직접 지원 (NemoClaw도 Landlock 사용, 그러나 별도 계층)
- WASM 샌드박스보다 OS 레벨 격리 — 커널이 직접 파일/네트워크 접근 제어

---

## 메모리 시스템

10개 메모리 엔진 지원:

| 백엔드 | 유형 |
|--------|------|
| SQLite | 키워드/벡터 |
| PostgreSQL | 관계형 + 벡터 |
| Redis | 인메모리 캐시 |
| ClickHouse | 분석용 컬럼 스토어 |
| + 6개 추가 | 하이브리드 변형 포함 |

- 벡터/키워드 검색 가중치 설정 가능 (ZeroClaw linear fusion 유사)
- **외부 의존성 최소**: SQLite는 선택적, 기본은 libc만

---

## 채널 및 도구

### 19개 메시징 채널 (생태계 2위)

| 채널 | 특이점 |
|------|--------|
| Telegram | 포럼 토픽별 바인딩 지원 |
| Discord | |
| **Signal** | 전용 `docker-compose.signal.yml` + `SIGNAL.md` |
| Slack | |
| **Nostr** | 탈중앙화 프로토콜 (10개 프레임워크 중 유일) |
| **Matrix** | 탈중앙화 메시징 (10개 프레임워크 중 유일) |
| Voice | `voice.zig` |
| Web | 페어링 또는 토큰 인증 |
| + 11개 | |

→ OpenFang(40채널) > OpenJarvis(24채널+) > **NullClaw(19채널)** > OpenClaw(12채널+)

### 도구

- **35+ 빌트인 도구**
- MCP 통합 (`mcp.zig`)
- RAG 파이프라인 (`rag.zig`)
- Cron 스케줄러 (`cron.zig`)

---

## 차별점

| 특성 | NullClaw | 비교 대상 |
|------|----------|-----------|
| **언어** | Zig (유일) | Rust 2개, Go 1개, TS 3개, Python |
| **바이너리 크기** | 678 KB | ZeroClaw 8.8MB (13배 차이) |
| **시작 시간** | <2ms | ZeroClaw <10ms |
| **WASI 지원** | [O] | 전체 10개 중 유일 |
| **Nostr/Matrix** | [O] | 전체 10개 중 유일 |
| **Landlock 네이티브** | [O] | NemoClaw에도 있으나 레이어 방식 다름 |
| **A2A 프로토콜** | [O] | OpenFang, OpenJarvis와 동일 |

### 신규 패턴 후보

- **R15: 정적 컴파일 = 보안 표면 최소화**: libc만 의존하는 Zig 정적 바이너리는 동적 링킹에서 오는 공급망 공격 표면 자체를 제거. WASM과 다른 접근.
- **R16: WASI = 에이전트 이식성**: WebAssembly System Interface 타겟 지원으로 브라우저/엣지/서버 동일 바이너리. 에이전트 배포 표준화 가능성.

---

## 한계

- **Zig 생태계 크기**: Rust/Python/TypeScript 대비 커뮤니티 작음, 라이브러리 부족
- **Zig 버전 고정**: 0.15.2 정확히 요구 — 버전 관리 오버헤드
- **Stars 6,400**: 타 프레임워크 대비 낮음 (ZeroClaw 17K, PicoClaw 17-19K)
- **WASM 샌드박스 없음**: OpenFang/IronClaw의 WASM 이중 미터링 대신 OS 레벨 격리(Landlock)
- **브라우저 자동화**: OpenClaw(50+), OpenFang(CDP 50+)과 달리 명시적 브라우저 모듈 미확인

---

## 참고 링크

- [GitHub — nullclaw/nullclaw](https://github.com/nullclaw/nullclaw)
- [SECURITY.md](https://github.com/nullclaw/nullclaw/blob/main/SECURITY.md)
- [SIGNAL.md](https://github.com/nullclaw/nullclaw/blob/main/SIGNAL.md)
- [AGENTS.md](https://github.com/nullclaw/nullclaw/blob/main/AGENTS.md)

---

*분석일: 2026-03-17 | 소스: repos/nullclaw/ 직접 코드 분석*
