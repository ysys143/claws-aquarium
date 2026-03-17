# TinyClaw 상세 분석 보고서

> **소스**: `reports/repos/framework_catalog.md` §7에서 추출
> **조사일**: 2026-02-25 (최종 업데이트)
> **GitHub**: [jlia0/tinyclaw](https://github.com/jlia0/tinyclaw) / [TinyAGI/tinyclaw](https://github.com/TinyAGI/tinyclaw)

---

## 목차

1. [기본 정보](#기본-정보)
2. [멀티에이전트 아키텍처](#멀티에이전트-아키텍처)
3. [멀티팀 체인 실행](#멀티팀-체인-실행)
4. [실시간 TUI 대시보드](#실시간-tui-대시보드)
5. [TinyOffice 웹 포털](#tinyoffice-웹-포털)
6. [24/7 운영](#247-운영)
7. [멀티채널](#멀티채널)
8. [주요 포크/변종](#주요-포크변종)
9. [참고 링크](#참고-링크)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub** | [jlia0/tinyclaw](https://github.com/jlia0/tinyclaw) (원본) / [TinyAGI/tinyclaw](https://github.com/TinyAGI/tinyclaw) (발전형) |
| **개발자** | Jian (jlia0, 샌프란시스코) |
| **라이선스** | MIT |

> "OpenClaw는 훌륭하지만 항상 깨진다. 그래서 셸 스크립트 ~400줄로 OpenClaw를 재창조했다."

---

## 멀티에이전트 아키텍처

- 에이전트 유형: `coder`, `writer`, `reviewer`, `researcher`, `documentation`
- 각 에이전트가 **독립 워크스페이스 + 대화 히스토리** 보유
- `@agent_id` 구문으로 태스크 라우팅
  ```
  @coder API 빌드; @writer 문서 작성; @reviewer PR 감사
  ```
- SQLite 기반 메시지 큐 (WAL 모드, 원자적 트랜잭션)
- 메시지 상태 흐름: pending → processing → completed / dead-letter (최대 5회 재시도)

---

## 멀티팀 체인 실행

- `settings.json`에서 팀 정의 + 리더 기반 라우팅
- **체인 모드**: Leader → Agent A → Agent B → Agent C (순차 핸드오프)
- **팬아웃 모드**: Leader가 서브태스크를 여러 에이전트에 **병렬 배포**
- 격리된 팀 대화 로그: `.tinyclaw/chats/{team_id}/`

---

## 실시간 TUI 대시보드

```bash
tinyclaw team visualize [id]
```

- 에이전트 체인 플로우를 터미널에서 실시간 시각화
- 태스크 핸드오프, 실행 상태, 라이브 메트릭 표시

---

## TinyOffice 웹 포털 (보너스)

- Next.js 기반 브라우저 대시보드
- 채팅 콘솔, 에이전트/팀 라우팅 인터페이스
- 칸반 스타일 태스크 관리
- 라이브 로그, 이벤트 인스펙션, 설정 에디터

---

## 24/7 운영

- tmux 세션 기반 프로세스 관리로 지속 운영
- Discord, Telegram, WhatsApp 동시 연결
- Claude Code CLI / OpenAI Codex CLI 위에서 구동

---

## 멀티채널

- Discord (봇 토큰 + 메시지 콘텐츠 인텐트)
- Telegram (BotFather 토큰)
- WhatsApp (QR 코드 디바이스 링킹)
- 발신자 허용목록 + 대기/승인 보안

---

## 주요 포크/변종

| 프로젝트 | 특징 |
|----------|------|
| [TinyAGI/tinyclaw](https://github.com/TinyAGI/tinyclaw) | 완전한 멀티팀 아키텍처 |
| [warengonzaga/tinyclaw](https://github.com/warengonzaga/tinyclaw) | Bun 런타임, 3레이어 적응형 메모리, GPLv3, 독립 프로젝트 |
| [suislanchez/tinyclaw](https://github.com/suislanchez/tinyclaw) | TUI, 스트리밍, 병렬 도구 |

---

## 참고 링크

- [GitHub — jlia0/tinyclaw](https://github.com/jlia0/tinyclaw)
- [GitHub — TinyAGI/tinyclaw](https://github.com/TinyAGI/tinyclaw)
- [Open Alternative to OpenClaw — TinyClaw](https://www.scriptbyai.com/ai-assistant-tinyclaw/)
- [TinyClaw ClawRouter Architecture](https://sonusahani.com/blogs/tinyclaw-clawrouter)
