# ClawPort 상세 분석 보고서

> **소스**: `reports/repos_applied/repos_applied_report.md` §3.2에서 추출
> **조사 일자**: 2026-03-07
> **조사 방법**: scientist 에이전트 소스코드 심층 분석
> **기반 프레임워크**: OpenClaw (프록시)

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. ["Zero Own Key" 아키텍처](#2-zero-own-key-아키텍처)
3. [에이전트 자동 발견 — 4단계 폴백 체인](#3-에이전트-자동-발견--4단계-폴백-체인)
4. [채팅 파이프라인 — 텍스트 vs 비전 분기](#4-채팅-파이프라인--텍스트-vs-비전-분기)
5. [메모리 브라우저](#5-메모리-브라우저)
6. [비용 대시보드](#6-비용-대시보드)
7. [UI 특이점](#7-ui-특이점)
8. [테스트](#8-테스트)

---

## 1. Executive Summary

**핵심 철학**: "에이전트 팀을 사람이 볼 수 있게" — 자체 AI 키 없이 OpenClaw를 완전히 프록시

ClawPort는 9개 API 라우트 전체에서 직접 Anthropic API 호출이 **0건**이다. 모든 AI 호출이 OpenClaw 게이트웨이(localhost:18789)를 경유하는 "Zero Own Key" 아키텍처를 채택했다.

---

## 2. "Zero Own Key" 아키텍처

ClawPort는 자체 Anthropic API 키를 전혀 보유하지 않는다. 모든 AI 호출이 OpenClaw 게이트웨이(localhost:18789)를 경유한다:

```
Browser  →  Next.js API Routes
               ├── 텍스트: openai.chat.completions(baseURL="localhost:18789/v1") → 스트리밍 SSE
               ├── 비전: execFile(openclaw CLI) → chat.send → 폴링 chat.history → SSE
               ├── 음성(STT): Whisper via localhost:18789/v1/audio/transcriptions
               ├── 음성(TTS): openclaw TTS → SSE 청크
               └── 로그 스트림: spawn(openclaw logs --follow --json) → SSE
```

9개 API 라우트 전체에서 직접 Anthropic API 호출 0건 (`app/api/chat/[id]/route.ts:9-12`).

---

## 3. 에이전트 자동 발견 — 4단계 폴백 체인

`lib/agents-registry.ts:505`의 `loadRegistry()` 우선순위:

```
1. User Override     $WORKSPACE_PATH/clawport/agents.json
       ↓ 없으면
2. Auto-Discovery    IDENTITY.md → root SOUL.md → agents/*/SOUL.md
                     → sub-agents/*.md / members/*.md
       ↓ 없으면
3. CLI-Only          openclaw agents list --json 결과만 사용
       ↓ 없으면
4. Bundled Fallback  lib/agents.json (빌드/테스트용)
```

어떤 OpenClaw 워크스페이스도 별도 설정 없이 즉시 작동한다. `parseSoulHeading()`이 5가지 SOUL.md 헤딩 포맷을 처리하고, 15가지 색상 팔레트에서 에이전트 색상을 자동 배정한다 (`agents-registry.ts:11-55`).

---

## 4. 채팅 파이프라인 — 텍스트 vs 비전 분기

**비전 파이프라인이 CLI 기반인 이유** (3가지 기술적 제약):

1. 게이트웨이 HTTP 엔드포인트가 `image_url` 컨텐츠 파트를 제거함
2. WebSocket `operator.write` 스코프는 device keypair 서명 필요 — CLI만 보유
3. macOS ARG_MAX(1MB) 제약 → 1200px JPEG(0.85 품질) 리사이징 필수

```
Client Canvas API 리사이징 (최대 1200px)
  → 최신 user 메시지만 이미지 감지 (route.ts:60-61)
  → execFile("openclaw", ["gateway", "call", "chat.send", ...], timeout:15s)
  → 2초 폴링 chat.history (최대 60초, timestamp >= sendTs 매칭)
  → 단일 SSE 청크 반환
```

`lib/anthropic.ts:123,142,161`

---

## 5. 메모리 브라우저

`lib/memory.ts`의 `getMemoryConfig()`가 OpenClaw의 `openclaw.json`을 직접 파싱해 하이브리드 검색 설정을 읽는다:

```json
{ "vectorWeight": 0.7, "textWeight": 0.3, "halfLifeDays": 30, "mmrLambda": 0.7,
  "softThresholdTokens": 80000 }
```

memory_architecture_report.md에서 분석한 OpenClaw Tier 1 메모리 아키텍처 설정을 UI에서 그대로 시각화한다. 단, 읽기 전용 — MEMORY.md 수정 불가 (`lib/memory.ts:131`).

---

## 6. 비용 대시보드

순수 함수 파이프라인으로 구성:

```
toRunCosts() → computeJobCosts() + computeDailyCosts() + computeModelBreakdown()
             + detectAnomalies() + computeWeekOverWeek() + computeCacheSavings()
```

이상 감지 조건: 동일 job 3회 이상 실행 AND 중앙값 토큰의 5배 초과 (`lib/costs.ts:143-148`). 내장 모델 가격표: Claude Opus/Sonnet/Haiku 7개 항목, 미지원 모델은 Sonnet 가격으로 폴백.

---

## 7. UI 특이점

- **Org Map**: React Flow + Dagre, Hierarchy/Teams 두 모드. Teams 모드에서 팀별 독립 서브그래프 배치 (`components/OrgMap.tsx:27-28`)
- **마크다운 렌더링**: 외부 라이브러리 없이 regex 기반 구현 (테이블, 이미지 미지원)
- **대화 저장**: localStorage + base64 data URL (blob URL은 새로고침 시 소멸하므로 불사용)
- **슬래시 명령어**: 6개 (`/clear /help /info /soul /tools /crons`), 완전 클라이언트 사이드, API 전송 필터링
- **5개 테마**: CSS custom properties (`--bg`, `--text-primary`, `--accent` 등 33개 시맨틱 토큰)

---

## 8. 테스트

536개 테스트, 24개 스위트, 모두 `lib/` 디렉토리에 소스 파일과 동일 위치. 핵심 패턴:

```typescript
vi.mock('child_process')                           // CLI subprocess 격리
vi.useFakeTimers({ shouldAdvanceTime: true })      // 폴링 루프 시뮬레이션 필수
vi.stubEnv('WORKSPACE_PATH', '/mock')              // 환경변수 격리
```

`shouldAdvanceTime: true`는 비전 파이프라인의 2초 폴링 루프 테스트를 가능하게 하는 핵심 옵션이다 (`CLAUDE.md:292-304`).
