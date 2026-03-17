# IronClaw 상세 분석 보고서

> **소스**: `reports/repos/framework_catalog.md` §4에서 추출
> **조사일**: 2026-02-25 (최종 업데이트)
> **GitHub**: [nearai/ironclaw](https://github.com/nearai/ironclaw) — 3,300+ stars

---

## 목차

1. [기본 정보](#기본-정보)
2. [WASM 샌드박스](#1-wasm-샌드박스)
3. [자격증명 보호](#2-자격증명-보호)
4. [프롬프트 인젝션 방어](#3-프롬프트-인젝션-방어)
5. [기술 아키텍처](#기술-아키텍처)
6. [커뮤니티 평가](#커뮤니티-평가-hacker-news)
7. [참고 링크](#참고-링크)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub** | [nearai/ironclaw](https://github.com/nearai/ironclaw) |
| **Stars** | 3,300+ |
| **개발자** | Illia Polosukhin (NEAR Protocol 공동창업자, "Attention Is All You Need" 공저자) |
| **라이선스** | MIT |
| **최신 릴리스** | v0.11.1 (2026.2.23) |

> "사람들이 OpenClaw를 쓰다가 자금과 자격증명을 잃고 있다. 우리는 보안에 초점을 맞춘 버전을 만들기 시작했다." — Illia Polosukhin

---

## 1. WASM 샌드박스

IronClaw의 핵심 보안 메커니즘. `wasmtime` 런타임으로 서드파티 도구를 격리 실행합니다.

```
WASM → 허용목록 검증 → 유출 스캔(요청) → 자격증명 주입 → 실행 → 유출 스캔(응답) → WASM
```

- 명시적 옵트인 필요: HTTP 접근, 시크릿 접근, 도구 호출
- HTTP 엔드포인트 화이트리스트: 사전 승인된 호스트/경로만 허용
- 리소스 제한: 메모리, CPU, 실행 시간 제약
- 레이트 리밋: 도구별 요청 수 상한

---

## 2. 자격증명 보호

- API 키가 **절대** LLM 컨텍스트에 노출되지 않음
- **AES-256-GCM 암호화 볼트** (PostgreSQL 기반)
- 실행 시점에만 호스트 경계에서 특정 승인 사이트용으로만 주입
- 유출 감지: 발신 요청과 수신 응답 모두에서 자격증명 유출 패턴 스캔
- Anti-Stealer 모듈: SSH 키 열거, 클라우드 자격증명 접근, 다단계 유출 체인 모니터링

---

## 3. 프롬프트 인젝션 방어

다단계 방어 스택:
- 패턴 감지 (알려진 인젝션 패턴 사전 차단)
- 콘텐츠 정화 (외부 콘텐츠 클리닝)
- 정책 시행 (콘텐츠 출처에 따른 허용 동작 규칙)

---

## 기술 아키텍처

- **Rust 1.85+**, tokio 비동기, Arc/RwLock 동시성
- **PostgreSQL 15+** + pgvector (프로덕션) / **libSQL/Turso** (로컬 대안)
- 하이브리드 메모리: 전문 검색 + 벡터 코사인 유사도
- 채널: REPL (Ratatui TUI), HTTP 웹훅, WASM 채널, WebSocket 스트리밍
- 지원 LLM: NEAR AI, Anthropic, OpenAI, Ollama, OpenRouter, Together AI 등
- `.unwrap()` / `.expect()` / clippy 경고 **제로 톨러런스** (코드 리뷰 레벨에서 강제)

---

## 커뮤니티 평가 (Hacker News)

- "WASM 샌드박스의 위협 모델이 불충분하게 문서화됨" (amluto)
- `webfetch`와 코드 실행 결합 시 근본적 공격 표면 존재
- 샌드박싱은 완벽한 보안 솔루션은 아니지만 없는 것보다 확실히 나음

---

## 참고 링크

- [GitHub — nearai/ironclaw](https://github.com/nearai/ironclaw)
- [IronClaw rivals OpenClaw — CoinTelegraph](https://magazine.cointelegraph.com/ironclaw-secure-private-sounds-cooler-openclaw-ai-eye/)
- [REPORT: IronClaw — TheCoding Substack](https://thecoding.substack.com/p/report-ironclaw-openclaw-in-rust)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=47004312)
