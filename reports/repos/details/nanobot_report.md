# Nanobot 상세 분석 보고서

> **소스**: `reports/repos/framework_catalog.md` §2에서 추출
> **조사일**: 2026-02-25 (최종 업데이트)
> **GitHub**: [HKUDS/nanobot](https://github.com/HKUDS/nanobot) — 24,100+ stars

---

## 목차

1. [기본 정보](#기본-정보)
2. [핵심 특징](#핵심-특징)
3. [아키텍처 (5개 모듈 레이어)](#아키텍처-5개-모듈-레이어)
4. [MCP 지원](#mcp-지원)
5. [멀티채널 지원](#멀티채널-지원)
6. [LLM 프로바이더](#llm-프로바이더)
7. [비전](#비전)
8. [참고 링크](#참고-링크)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub** | [HKUDS/nanobot](https://github.com/HKUDS/nanobot) |
| **Stars** | 24,100+ |
| **개발** | 홍콩대학교 데이터사이언스 랩 (HKUDS) |
| **주요 메인테이너** | Re-bin (`@Re-bin`) |
| **라이선스** | MIT |

---

## 핵심 특징

- OpenClaw 대비 **99% 작은 코드베이스** (~3,897줄 vs 430,000+줄)
- 연구에 바로 활용 가능한 깔끔하고 가독성 높은 코드
- 두 가지 운영 모드: `nanobot agent` (CLI) / `nanobot gateway` (멀티채널 서버)

---

## 아키텍처 (5개 모듈 레이어)

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

---

## MCP 지원 (v0.1.4+)

- Claude Desktop/Cursor와 **동일한 설정 스키마** 사용 (마이그레이션 불필요)
- stdio (로컬 프로세스) + HTTP/SSE (리모트 엔드포인트) 양방향 트랜스포트
- MCP 도구 자동 검색 및 등록, 인증 헤더 지원

---

## 멀티채널 지원 (9개+)

Telegram, Discord, WhatsApp, Slack, Email (IMAP/SMTP), QQ, Feishu, DingTalk, WeChat

---

## LLM 프로바이더 (15개+)

OpenRouter, Anthropic Claude, OpenAI, DeepSeek, Groq, Google Gemini, MiniMax, Qwen, GitHub Copilot, OpenAI Codex, SiliconFlow, 커스텀 OpenAI 호환 엔드포인트 등

---

## 비전

> "커널은 모든 드라이버를 탑재하지 않지만, 누구나 드라이버를 작성할 수 있다."
>
> 장기 목표: **에이전트 커널** — Linux처럼 코어는 최소화하고 모든 통합을 `pip install nanobot-{plugin}` 생태계로 확장

---

## 참고 링크

- [GitHub — HKUDS/nanobot](https://github.com/HKUDS/nanobot)
- [v0.1.4 Release Notes](https://github.com/HKUDS/nanobot/releases/tag/v0.1.4)
- [DeepWiki Architecture Analysis](https://deepwiki.com/HKUDS/nanobot)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=46897737)
