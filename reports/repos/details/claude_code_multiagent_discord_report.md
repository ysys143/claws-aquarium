# Claude Code 채널 플러그인 — Discord Bot-to-Bot 멀티 에이전트 통신

> **소스**: `anthropics/claude-plugins-official` Discord 플러그인 + 커뮤니티 발굴 (고영혁, 2026-03-22)
> **기반 분석**: `repos/cc_2.1.80/28-claude-code-channels.md`, `reports/repos/details/claude_code_report.md`
> **작성일**: 2026-03-22

---

## 목차

1. [개요](#1-개요)
2. [핵심 발견: 한 줄 코드 변경](#2-핵심-발견-한-줄-코드-변경)
3. [아키텍처: Bot-to-Bot 통신 구조](#3-아키텍처-bot-to-bot-통신-구조)
4. [플랫폼 비교: Discord vs Telegram vs Slack](#4-플랫폼-비교-discord-vs-telegram-vs-slack)
5. [멀티 에이전트 시스템 구축 요건](#5-멀티-에이전트-시스템-구축-요건)
6. [신규 패턴 R33](#6-신규-패턴-r33)
7. [한계 및 주의사항](#7-한계-및-주의사항)
8. [참고 링크](#8-참고-링크)

---

## 1. 개요

Claude Code 공식 채널 플러그인(Discord)의 기본 설정은 모든 봇(bot) 메시지를 차단한다. 이는 자동화된 외부 시스템이 무단으로 Claude 세션에 메시지를 주입하는 것을 막기 위한 방어 설계다.

그러나 단 **한 줄의 코드를 변경**하면 이 제한이 해제되어, **동일한 Discord 채널에 연결된 여러 Claude Code 에이전트가 서로 자율적으로 메시지를 주고받는 시스템**을 구축할 수 있다.

고영혁(Gonnector)이 2026-03-22 Facebook 포스트를 통해 공개한 이 발견은, 추가 인프라 개발 없이 공식 기본 기능만으로 **완전 자율 멀티 에이전트 워크플로우**를 10분 안에 구현할 수 있음을 보여준다.

---

## 2. 핵심 발견: 한 줄 코드 변경

### 변경 위치

`anthropics/claude-plugins-official` 저장소 내 Discord 플러그인:

```
external_plugins/discord/server.ts  (약 669번째 줄)
```

### 변경 내용

```typescript
// 변경 전 (기본값): 모든 봇 메시지 차단
if (msg.author.bot) return

// 변경 후: 자기 자신의 메시지만 무시
if (msg.author.id === client.user?.id) return
```

### 효과

| 상태 | 동작 |
|------|------|
| **변경 전** | Discord 봇이 보낸 모든 메시지 무시. Claude Code 세션은 사람 메시지만 처리. |
| **변경 후** | 다른 봇(=다른 Claude Code 세션)이 보낸 메시지도 처리. 자신이 보낸 메시지만 무시 (무한루프 방지). |

### 왜 이 변경이 안전한가

- 봇 계정에도 동일한 access control이 적용된다 — 사전에 pairing 또는 allowlist에 등록된 발신자만 Claude 컨텍스트에 주입 가능
- 자기 자신의 메시지를 무시하므로 에코 루프(무한 응답) 원천 차단
- Discord 플랫폼 차원에서는 봇-봇 메시지가 원래 허용되어 있다 (Telegram과 달리)

---

## 3. 아키텍처: Bot-to-Bot 통신 구조

### 전체 구성도

```
                   Discord 서버
                   +-----------------------------+
  사람 사용자 -----+  #ai-team 채널               |
                   |                             |
                   |  @JARVIS (Bot A)            |
                   |  @EVE (Bot B)               |
                   |  @ZEUS (Bot C, 추가 가능)    |
                   +-----+----------+------------+
                         |          |
              stdio MCP  |          |  stdio MCP
                         v          v
              +-------------+   +-------------+
              | Claude Code |   | Claude Code |
              |  세션 A     |   |  세션 B     |
              |  (JARVIS)   |   |  (EVE)      |
              +------+------+   +------+------+
                     |                 |
              sub-agents         sub-agents
              (위임된 작업)      (위임된 작업)
```

### 메시지 흐름

1. 사람이 Discord 채널에서 `@JARVIS 보고서 작성해줘` 입력
2. JARVIS Claude Code 세션에 `<channel source="discord" user="human" ...>` 형식으로 주입
3. JARVIS가 분석 후 `@EVE 데이터 수집 담당해줘 ...` 전송 (`reply` 툴 사용, @멘션 필수)
4. EVE Claude Code 세션에 `<channel source="discord" user="JARVIS_bot_id" ...>` 로 주입
5. EVE가 작업 수행 후 `@JARVIS 완료했어, 결과: ...` 회신
6. JARVIS가 종합하여 사람에게 `@human 보고서 완성` 전달

### @멘션 필수 조건

봇 이름을 텍스트로만 언급하는 것은 플러그인에서 처리되지 않는다. 반드시 Discord @멘션(`<@DISCORD_USER_ID>`) 형식을 사용해야 한다. `isMentioned()` 함수가 다음 두 가지만 처리하기 때문이다:

```typescript
async function isMentioned(msg, extraPatterns) {
  if (msg.mentions.has(client.user)) return true     // 직접 @멘션만 통과
  // ... reply 감지 ...
}
```

→ **CLAUDE.md에 "다른 에이전트나 사람에게 말을 걸 때는 반드시 @멘션 사용"을 명시 필수**

---

## 4. 플랫폼 비교: Discord vs Telegram vs Slack

| 항목 | Discord | Telegram | Slack |
|------|---------|----------|-------|
| **봇-봇 메시지** | [O] 플랫폼 허용 | [X] 플랫폼 차단 | 개발 필요 |
| **Claude Code 공식 채널** | [O] (claude-plugins-official) | [O] (claude-plugins-official) | [X] (공식 플러그인 없음) |
| **멀티 에이전트 구현** | 한 줄 변경으로 가능 | 불가능 (플랫폼 정책) | 별도 개발 필요 |
| **메시지 실시간성** | WebSocket Gateway (실시간) | Long Polling | Events API (실시간) |
| **공식 Claude 앱** | 없음 | 없음 | Claude Chat 연동 존재 |
| **멀티봇 한 채널** | [O] 자연스럽게 지원 | 기술적 가능하나 봇-봇 차단 | 앱 하나만 허용 |

### Telegram 우회 불가 이유

Telegram은 **플랫폼 레벨**에서 봇이 다른 봇에게 메시지를 보내는 것을 차단한다. Claude Code 플러그인 코드 수정과 무관하게, Telegram API 자체가 봇 -> 봇 메시지 전달을 거부한다. 멀티 에이전트 구현 시 Discord가 현재 유일한 공식 지원 옵션이다.

### Slack 상황

현재 Slack에서 Claude 멀티 에이전트를 구현하려면:
- Slack 공식 MCP: HTTP 폴링 방식 -> 실시간 아님, 별도 개발 필요
- Slack 공식 Claude 앱: Events API 기반 실시간 가능하나, 하나의 워크스페이스에 Claude 앱 인스턴스 중복 불가
- 커스텀 구현: Slack Events API + 별도 Claude Code 연결 개발 필요

---

## 5. 멀티 에이전트 시스템 구축 요건

### 5.1 필수 설정

#### 각 에이전트마다 별도 Discord 봇 토큰 필요

```bash
# JARVIS 세션
~/.claude/channels/discord/.env
TOKEN_DISCORD_BOT_TOKEN=<JARVIS_BOT_TOKEN>

# EVE 세션 (다른 디렉토리 또는 프로파일 필요)
~/.claude/channels/discord-eve/.env
TOKEN_DISCORD_BOT_TOKEN=<EVE_BOT_TOKEN>
```

#### access.json 상호 등록

각 봇이 상대 봇의 Discord ID를 allowlist에 등록해야 한다:

```json
// JARVIS access.json
{
  "dmPolicy": "allowlist",
  "allowFrom": ["EVE_BOT_ID", "HUMAN_USER_ID"],
  "groups": { "CHANNEL_ID": { "require": "mention" } }
}
```

#### CLAUDE.md 통신 프로토콜 설정 (전역)

```markdown
# 멀티 에이전트 통신 규칙
- 다른 에이전트(@EVE 등)나 사람(@사용자)에게 메시지를 보낼 때는 반드시 Discord @멘션 사용
- 봇 이름을 텍스트로 언급하는 것은 전달되지 않음
- 모든 실질적인 작업은 sub-agent 또는 agent team에 위임 후 본체는 대기 상태 유지
```

### 5.2 응답성 유지: 본체는 항상 대기 상태

가장 중요한 운용 원칙이다. Discord에서 메시지를 받은 에이전트는 **즉시 응답 가능한 상태**여야 한다. 에이전트가 작업 중에 메시지가 오면 메시지는 큐에 쌓이고, Discord 앱에는 "타이핑 중" 애니메이션이 표시되지만 실제로는 답변 불가 상태다.

```
[잘못된 운용]
JARVIS가 직접 보고서 작성 (긴 작업 중)
  -> EVE가 @JARVIS 질문 전송
  -> JARVIS 큐에 쌓임 (답변 불가)
  -> 워크플로우 중단

[올바른 운용]
JARVIS가 sub-agent에 보고서 작성 위임
  -> JARVIS 본체는 Discord 대기 상태
  -> EVE가 @JARVIS 질문 전송
  -> JARVIS 즉시 처리 가능
```

### 5.3 공유 메모리 전략

에이전트들이 동일한 작업을 협업하려면 상태 공유가 필요하다. 현재 공식 지원 방법은 없으며, 커뮤니티에서 논의되는 접근법:

| 방법 | 장점 | 단점 |
|------|------|------|
| 공유 파일시스템 (`STATE.yaml`, R31) | 즉시 구현 가능 | 충돌 가능성 |
| Discord 메시지 자체를 상태로 활용 | 별도 인프라 불필요 | 히스토리 의존, 100개 제한 |
| 별도 MCP 서버 (공유 DB) | 강건함 | 추가 개발 필요 |
| 포인터맵 메모리 (R30) | 대규모 메모리 | bash 의존 |

---

## 6. 신규 패턴 R33

### R33: 채널 플러그인 기반 Bot-to-Bot 멀티 에이전트 버스

**발굴 소스**: 고영혁 (Gonnector), Discord 커뮤니티 실험, 2026-03-22

**핵심 원리**:
표준 메시징 플랫폼(Discord)의 채널을 별도의 오케스트레이터나 메시지 큐 없이 **에이전트 간 통신 버스**로 활용한다. 각 에이전트는 독립된 Claude Code 세션으로 실행되며, Discord를 공통 통신 레이어로 삼아 느슨하게 결합된다.

```
에이전트 A (Claude Code 세션) --+
에이전트 B (Claude Code 세션) --+-- Discord 채널 (통신 버스)
에이전트 C (Claude Code 세션) --+
```

**기존 패턴과의 차이**:

| 패턴 | 방식 |
|------|------|
| R21 Bounded Delegation Tree (Hermes) | 코드 레벨 트리 구조, 동기적 |
| R31 Shared-State File Coordination | 파일 시스템 공유, 중앙 없음 |
| **R33 Bot-to-Bot Channel Bus** | 메시징 플랫폼 활용, 비동기, 사람과 같은 인터페이스 |

**시사점**:
- 추가 인프라 없이 기존 채팅 플랫폼이 에이전트 조율 레이어가 됨
- 사람이 동일한 채널에서 워크플로우를 실시간 관찰하고 개입 가능
- 에이전트 수가 늘어날수록 컨텍스트 조율(누가 무엇을 아는가) 복잡도 증가

**3에이전트 이상의 특이점 ("정치" 문제)**:
2명 -> 3명으로 확장 시 발생하는 사회적 동학(coalition, 합의 불일치, 책임 분산)이 에이전트 간에도 동일하게 발생할 수 있다. 고영혁은 이를 "이제부터 정치가 가능해집니다"라고 표현했다. 에이전트 역할 명세와 결정 권한 체계를 CLAUDE.md에 사전 정의하는 것이 중요해진다.

---

## 7. 한계 및 주의사항

### 7.1 기술적 한계

| 항목 | 내용 |
|------|------|
| **세션 종속성** | Claude Code 세션이 닫히면 해당 봇은 응답 불가. tmux/screen 필수 |
| **메시지 큐잉** | 에이전트 작업 중 수신 메시지는 큐에 쌓임 (즉시 처리 불가) |
| **@멘션 강제** | 이름 텍스트만으로는 전달 안됨. Discord ID 기반 @멘션 필수 |
| **Anthropic allowlist** | `tengu_harbor_ledger` 플래그로 Anthropic이 원격 제어. Research Preview 동안 공식 플러그인만 허용 |
| **권한 프롬프트** | `--dangerously-skip-permissions` 없으면 unattended 운용 중단 위험 |
| **히스토리 제한** | `fetch_messages` 최대 100개. 장시간 대화 히스토리 관리 어려움 |
| **플러그인 수정** | 공식 저장소 포크 필요. 업스트림 업데이트 시 재적용 필요 |

### 7.2 설계 원칙적 주의사항

- **공유 메모리 시스템 미비**: 에이전트 간 일관된 상태 유지 방법이 공식 제공되지 않음
- **역할 명세 필수**: 역할이 불명확하면 중복 작업 또는 책임 공백 발생
- **응답 대기 전략**: 에이전트가 다른 에이전트의 응답을 기다리는 동안 자신은 다른 작업 수행 또는 대기 — 타임아웃 처리 필요
- **에러 전파**: A가 B에게 위임했을 때 B의 실패를 A가 감지하는 메커니즘 없음

### 7.3 Discord 플러그인 수정 지속성

현재 이 기능은 공식 플러그인 코드를 **직접 수정**해야 한다. Anthropic이 공식적으로 지원하는 기능이 아니며, 플러그인 업데이트 시 변경이 덮어쓰일 수 있다. 공식 기능화 여부는 미정이다.

---

## 8. 참고 링크

- [reports/repos/details/claude_code_report.md](claude_code_report.md) — Claude Code 전체 분석 (R23-R26)
- [repos/cc_2.1.80/28-claude-code-channels.md](../../../repos/cc_2.1.80/28-claude-code-channels.md) — 채널 플러그인 상세 기술 분석
- [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) — Discord/Telegram 플러그인 소스
- [reports/usecases/usecases_index.md](../../usecases/usecases_index.md) — usecases/ 교차 분析 (R29-R32)
- 고영혁, "Discord Claude Code 멀티 에이전트 실험" Facebook, 2026-03-22

---

*작성일: 2026-03-22 | 기반: 커뮤니티 발굴 + 공식 소스 코드 분석*
