# NanoClaw 상세 분석 보고서

> **소스**: `reports/repos/framework_catalog.md` §3에서 추출
> **조사일**: 2026-02-25 (최종 업데이트)
> **GitHub**: [qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw) — 13,500+ stars

---

## 목차

1. [기본 정보](#기본-정보)
2. [핵심 특징](#핵심-특징)
3. [컨테이너 기반 에이전트 실행](#컨테이너-기반-에이전트-실행)
4. [에이전트 스웜](#에이전트-스웜)
5. ["Fork, Customize, Own" 철학](#fork-customize-own-철학)
6. [실제 사용 사례](#실제-사용-사례)
7. [참고 링크](#참고-링크)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub** | [qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw) |
| **Stars** | 13,500+ |
| **개발자** | Gavriel Cohen (전 Wix 7년 엔지니어, Qwibit 공동창업자) |
| **라이선스** | MIT |
| **출시** | 2026.1.31 (1주 만에 7,000 stars) |

---

## 핵심 특징

- 코어 코드 약 **500줄** — "8분이면 전체를 이해할 수 있음"
- OpenClaw 400,000줄과 달리 누구나 (또는 보조 AI가) 전체 코드를 감사 가능
- Node.js/TypeScript + SQLite 스택

---

## 컨테이너 기반 에이전트 실행

- **macOS**: Apple Container (네이티브 macOS 가상화)
- **Linux/macOS 대안**: Docker
- 각 채팅 그룹마다 **독립 컨테이너** 할당 — 파일시스템 완전 격리
- 프롬프트 인젝션 피해 범위가 컨테이너 내부로 한정 (호스트 시스템 보호)
- 그룹별 독립 `CLAUDE.md` 메모리 파일 + 격리된 컨테이너 파일시스템

---

## 에이전트 스웜 (최초 지원)

- **Anthropic Agent SDK** 네이티브 기반 다중 에이전트 협업
- 서브 에이전트 간 메모리 컨텍스트 격리 → 민감 데이터 유출 방지
- 단일 채팅 인터페이스 내에서 전문 에이전트 팀 협업

---

## "Fork, Customize, Own" 철학

- 중앙 플러그인 마켓플레이스 **없음** (OpenClaw의 ClawHub와 대조)
- 레포를 **포크** → Claude Code가 직접 소스를 수정하는 방식
- `.claude/skills/` SKILL.md 파일로 확장 (예: `/add-telegram` 실행 → 소스 자동 수정)
- "설정 복잡성" 대신 "코드 수준 커스터마이징" 지향

---

## 실제 사용 사례

Qwibit 내부에서 "Andy"라는 NanoClaw 인스턴스가 영업 파이프라인 관리, WhatsApp/이메일 노트 파싱, DB 업데이트, 기술 작업 자율 실행에 활용 중

---

## 참고 링크

- [GitHub — qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw)
- [NanoClaw Official Site](https://nanoclaw.dev)
- [NanoClaw Solves OpenClaw Security Issues — Novalogiq](https://novalogiq.com/2026/02/11/nanoclaw-solves-one-of-openclaws-biggest-security-issues-and-its-already-powering-the-creators-biz/)
- [500 Lines vs 50 Modules — Architecture Analysis](https://fumics.in/posts/2026-02-02-nanoclaw-agent-architecture)
- [The Claw Wars: 11 OpenClaw Spin-Offs — Blocmates](https://www.blocmates.com/articles/the-claw-wars)
