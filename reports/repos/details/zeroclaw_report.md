# ZeroClaw 상세 분석 보고서

> **소스**: `reports/repos/framework_catalog.md` §5에서 추출
> **조사일**: 2026-02-25 (최종 업데이트)
> **GitHub**: [zeroclaw-labs/zeroclaw](https://github.com/zeroclaw-labs/zeroclaw) — 17,000+ stars

---

## 목차

1. [기본 정보](#기본-정보)
2. [성능 지표](#성능-지표)
3. [경량화 기법](#경량화-기법)
4. [벤치마크](#벤치마크)
5. [트레이트 기반 아키텍처](#트레이트-기반-아키텍처-8대-핵심-트레이트)
6. [메모리 하이브리드 검색](#메모리-하이브리드-검색)
7. [대상 하드웨어](#대상-하드웨어)
8. [참고 링크](#참고-링크)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub** | [zeroclaw-labs/zeroclaw](https://github.com/zeroclaw-labs/zeroclaw) |
| **Stars** | 17,000+ |
| **개발** | Harvard/MIT/Sundai.Club 커뮤니티 |
| **라이선스** | MIT |
| **출시** | 2026.2.13 (2일 만에 3,400+ stars) |
| **최신 릴리스** | v0.1.6 (2026.2.22) |
| **테스트** | 1,017개 |

---

## 성능 지표

| 지표 | 수치 |
|------|------|
| 바이너리 크기 | ~8.8MB (정적 링킹) |
| 런타임 RAM | **5MB 미만** (CLI 작업 시) |
| 유휴 데몬 | 10-15MB |
| 시작 시간 | **10ms 미만** |

---

## 경량화 기법

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

---

## 벤치마크

- ZeroClaw: **1.52MB** 활성 메모리
- OpenClaw: **7.8MB**
- 4GB 서버 기준: ZeroClaw **~200개 인스턴스** vs OpenClaw **~2개 인스턴스**

---

## 트레이트 기반 아키텍처 (8대 핵심 트레이트)

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

---

## 메모리 하이브리드 검색

외부 의존성 없는 자체 검색 엔진:
- FTS5 키워드 검색 (BM25 스코어링)
- 벡터 코사인 유사도 (저장된 임베딩)
- 가중 퓨전: `score = (keyword_weight x bm25) + (vector_weight x cosine)`
- LRU 임베딩 캐시로 API 호출 최소화

---

## 대상 하드웨어

- $10 싱글보드 컴퓨터 (Raspberry Pi급)
- 0.8GHz 프로세서
- 지원 아키텍처: ARM (aarch64, armv7), x86_64, RISC-V
- OpenClaw 마이그레이션: `zeroclaw migrate openclaw` 명령으로 기존 메모리/ID 파일 임포트

---

## 참고 링크

- [GitHub — zeroclaw-labs/zeroclaw](https://github.com/zeroclaw-labs/zeroclaw)
- [DeepWiki — What is ZeroClaw](https://deepwiki.com/zeroclaw-labs/zeroclaw/1.1-what-is-zeroclaw)
- [DEV.to — ZeroClaw Article](https://dev.to/brooks_wilson_36fbefbbae4/zeroclaw-a-lightweight-secure-rust-agent-runtime-redefining-openclaw-infrastructure-2cl0)
- [ZeroClaw Official Site](https://www.zeroclaw.dev/)
