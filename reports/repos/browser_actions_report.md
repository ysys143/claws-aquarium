# 브라우저 자동화 및 액션/도구 아키텍처 — 7개 Claw 코드 기반 비교 분석

> **조사 일자**: 2026-03-05 (최초), 2026-03-14 (OpenJarvis 추가), 2026-03-17 (OpenFang/NemoClaw 추가)
> **조사 방법**: 7개 scientist 에이전트가 각 레포의 브라우저 자동화 및 도구/액션 소스코드를 병렬 심층 분석; OpenJarvis, OpenFang, NemoClaw는 별도 분석 후 추가
> **핵심 질문**: "에이전트가 실세계와 어떻게 상호작용하는가? — 브라우저 자동화와 도구 시스템의 두 축"

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. [브라우저 자동화 비교](#2-브라우저-자동화-비교)
3. [도구/액션 아키텍처 비교](#3-도구액션-아키텍처-비교)
4. [개별 분석 요약](#4-개별-분석-요약)
5. [핵심 설계 패턴](#5-핵심-설계-패턴)
6. [교차 분석 및 논의](#6-교차-분석-및-논의)

---

## 1. Executive Summary

12개 구현체의 소스코드를 분석한 결과, **브라우저 자동화는 4개만 보유**하고, **도구 아키텍처는 9가지 유형**으로 분류된다.

### 브라우저 자동화 보유 현황

| 구현체 | 브라우저 | 용도 | 성숙도 |
|--------|---------|------|--------|
| **OpenClaw** | Playwright + CDP 직접 | 에이전트 도구 (30+ 파일, 50+ 기능) | [5/5] |
| **ZeroClaw** | 3 백엔드 (agent-browser/rust-native/computer_use) | 에이전트 도구 (16 액션) | [4/5] |
| **NanoClaw** | Playwright (호스트 프로세스) | X/Twitter 전용 스킬 | [3/5] |
| **IronClaw** | Playwright Python | E2E 테스트 전용 (에이전트 도구 아님) | [2/5] (테스트용) |
| Nanobot | — | 없음 | — |
| PicoClaw | — | 없음 | — |
| TinyClaw | — | 플레이스홀더 스킬만 | — |
| OpenFang | — | 없음 (MCP 경유 간접 가능, 브라우저를 채널 어댑터로 통합) | — |
| OpenJarvis | — | 직접 구현 없음, openhands 통합 경유 간접 지원 | — |
| NemoClaw | — | 없음 (샌드박스 내 OpenClaw에 위임) | — |

### 도구 아키텍처 유형

| 유형 | 구현체 | 핵심 패턴 |
|------|--------|-----------|
| **Rust Trait + WASM 샌드박스** | IronClaw | `Tool` trait, ToolRegistry, WASM 격리, MCP 프록시 |
| **Rust Trait + 조건부 등록** | ZeroClaw | `Tool` trait, 설정 기반 동적 등록, LangGraph 연동 |
| **Go Interface + 병렬 실행** | PicoClaw | `Tool` interface, goroutine 병렬 실행, MCP 래핑 |
| **TS Plugin + Hook 시스템** | OpenClaw | `AgentTool` 타입, 24개 플러그인 훅, ACP |
| **Python ABC + Registry** | Nanobot | `Tool` ABC, OpenAI 스키마, MCP 래핑 |
| **SKILL.md + 컨테이너 위임** | NanoClaw | 스킬 파일 + manifest.yaml, 3-way merge, IPC |
| **CLI 위임** | TinyClaw | 외부 CLI 서브프로세스, 플러그인 훅 |
| **HAND.toml + Rust Trait** | OpenFang | 60 빌트인 string dispatch, WASM Dual Metering, A2A 프로토콜 |
| **Python Class + AgentRegistry 데코레이터** | OpenJarvis | `@AgentRegistry.register()`, MCP 표준, ThreadPoolExecutor 병렬 실행, A2A 프로토콜 |

**가장 주목할 발견 5가지:**

1. **브라우저 자동화의 설계 트레이드오프가 극명하다.** OpenClaw는 "가능한 모든 것"(50+ 기능), ZeroClaw는 "교체 가능한 백엔드", NanoClaw는 "탐지 회피를 위한 호스트 실행"이라는 완전히 다른 전략을 취했다. OpenFang은 브라우저를 도구가 아닌 채널 어댑터로 통합하는 독자적 접근.

2. **도구 보안과 확장성이 반비례한다.** IronClaw와 OpenFang은 WASM 샌드박스로 최고의 격리를 제공하지만 도구 개발 복잡도가 가장 높다. TinyClaw는 단순히 CLI를 호출해서 확장이 쉽지만 보안이 없다. NemoClaw는 도구 자체를 구현하지 않고 컨테이너 격리로 모든 도구 실행을 샌드박스화하는 독특한 절충안.

3. **MCP가 사실상의 표준 확장 프로토콜이 되었다.** 12개 중 7개(IronClaw, ZeroClaw, PicoClaw, Nanobot, OpenClaw, OpenFang, OpenJarvis)가 MCP 서버 통합을 지원한다. 직접 도구를 구현하지 않고도 외부 MCP 서버를 래핑하여 도구로 등록하는 패턴이 공통적이다. OpenFang은 MCP 양방향(클라이언트+서버) 동시 지원으로 한 단계 더 나아감.

4. **병렬 도구 실행이 두 구현체로 늘었다.** PicoClaw(goroutine+WaitGroup)에 이어 OpenJarvis도 ThreadPoolExecutor로 병렬 tool call을 지원한다. 두 구현체 모두 동일한 동기(LLM이 여러 tool call을 한 번에 요청)에서 출발했지만, Go와 Python이라는 다른 언어 관용구를 사용한다.

5. **A2A 프로토콜이 MCP에 이어 두 번째 표준으로 수렴 중이다.** OpenFang과 OpenJarvis가 Google A2A spec을 독립적으로 채택. 에이전트 간 통신(A2A) + 도구 통합(MCP) 이중 프로토콜 스택이 차세대 에이전트 생태계 표준이 될 가능성 높음. NemoClaw는 별도 프로토콜 없이 OpenClaw 호스트 프로토콜에 편승.

---

## 2. 브라우저 자동화 비교

### 2.1 종합 비교 매트릭스

| 영역 | OpenClaw | ZeroClaw | NanoClaw | IronClaw |
|------|----------|----------|----------|----------|
| **엔진** | Playwright + CDP 직접 | 3 백엔드 선택 | Playwright | Playwright Python |
| **용도** | 에이전트 1급 도구 | 에이전트 1급 도구 | X/Twitter 스킬 | E2E 테스트만 |
| **실행** | 인프로세스 CDP 연결 | 서브프로세스/WebDriver/HTTP | 호스트 프로세스 (IPC) | pytest fixture |
| **네비게이션** | [O] | [O] | [O] (x.com만) | [O] (테스트) |
| **클릭/입력** | [O] (12종) | [O] (Find 포함) | [O] (data-testid) | [O] (테스트) |
| **스크린샷** | [O] (전체/요소/라벨) | [O] (전체/선택) | [X] | [O] (테스트) |
| **PDF** | [O] | [X] | [X] | [X] |
| **네트워크 모니터링** | [O] (500 req 캐시) | [X] | [X] | [X] |
| **모바일 에뮬레이션** | [O] (viewport) | [X] | [X] | [X] |
| **파일 업로드** | [O] | [X] | [X] | [X] |
| **다이얼로그** | [O] | [X] | [X] | [X] |
| **탭 관리** | [O] | [X] | [X] | [X] |
| **Aria 스냅샷** | [O] (3종) | [O] (Snapshot) | [X] | [X] |
| **JS 평가** | [O] (abort/timeout) | [X] | [X] | [X] |
| **Computer Use** | [X] | [O] (사이드카) | [X] | [X] |
| **SSRF 방지** | [O] (DNS 핀닝) | [O] (도메인 허용목록) | [X] | [X] (테스트용) |
| **프로필 격리** | [O] (다중 프로필) | [O] (세션 이름) | [O] (단일 프로필) | [X] |
| **인증 상태 저장** | [O] (쿠키+localStorage) | 세션 기반 | [O] (Chrome 프로필) | [X] |

### 2.2 브라우저 아키텍처 상세

#### OpenClaw — "풀스택 브라우저 에이전트" (50+ 기능)

가장 포괄적인 브라우저 자동화. Playwright와 CDP를 이중으로 사용.

```
LLM -> browser tool -> Browser Tool Server (HTTP)
                          -> Playwright Layer (pw-tools-core.*.ts)
                              -> SSRF Guard (navigation-guard.ts)
                              -> pw-session.ts: connectOverCDP -> Chrome
                          -> Raw CDP Layer (cdp.ts)
                              -> WebSocket -> Chrome DevTools
```

**핵심 파일:**
- `src/browser/pw-session.ts:341` — CDP 연결 (캐시/재시도/뮤텍스)
- `src/browser/pw-tools-core.interactions.ts` — 12종 인터랙션 (클릭/타입/호버/드래그/셀렉트/키입력/스크롤/파일업로드)
- `src/browser/pw-tools-core.snapshot.ts` — 네비게이션 + 3종 스냅샷 (aria/AI/role) + PDF
- `src/browser/navigation-guard.ts` — SSRF 방지 (DNS 핀닝, 프라이빗 IP 차단)
- `src/browser/pw-tools-core.storage.ts` — 인증 상태 저장/복원
- `src/browser/profiles.ts` — 다중 프로필 관리

**페이지 상태 추적** (`pw-session.ts:96-104`): WeakMap으로 페이지별 콘솔(500), 에러(200), 네트워크(500) 이벤트 캐시.

**보안**: `navigation-guard.ts`에서 SSRF 방지 — DNS 룩업 후 프라이빗 IP 차단, `metadata.google.internal` 차단, 프록시 환경 검증. 리다이렉트 후 최종 URL도 재검증.

#### ZeroClaw — "교체 가능한 3중 백엔드"

```
BrowserTool::execute(args)
    -> resolve_backend()
        Auto: RustNative > AgentBrowser > error
        |-- AgentBrowser: tokio::process::Command("agent-browser") --json --session
        |-- RustNative: WebDriver protocol (127.0.0.1:9515)
        +-- ComputerUse: HTTP POST to sidecar endpoint
```

**핵심 파일:**
- `src/tools/browser.rs:77` — `BrowserBackendKind` enum (4종)
- `src/tools/browser.rs:136-199` — `BrowserAction` enum (16종 액션)
- `src/tools/browser.rs:404-460` — SSRF 방지 (도메인 허용목록 + 프라이빗 IP 차단)
- `src/tools/browser.rs:843-963` — Computer Use 사이드카 (좌표 검증, 윈도우 허용목록)
- `src/tools/browser_open.rs` — HTTPS-only 단순 URL 오픈

**고유 기능**: Computer Use 통합 — Anthropic computer_use를 사이드카 HTTP 엔드포인트로 프록시. 좌표 검증(`max_coordinate_x/Y`), 윈도우 허용목록 적용.

#### NanoClaw — "탐지 회피를 위한 호스트 실행"

```
Container (agent) -> IPC 파일 쓰기 (/workspace/ipc/tasks/)
                        ^ 폴링
Host (ipc.ts) -> host.ts -> spawn('npx', ['tsx', script])
                              -> Playwright (실제 Chrome, non-headless)
                              -> JSON 결과 -> IPC 파일 (/workspace/ipc/x_results/)
                        ^ 폴링
Container (agent) -> 결과 수신
```

**핵심 파일:**
- `.claude/skills/x-integration/lib/browser.ts:73-88` — `launchPersistentContext` (실제 Chrome, non-headless)
- `.claude/skills/x-integration/lib/config.ts:50-57` — 안티 자동화 탐지 인자
- `.claude/skills/x-integration/host.ts:29` — 스크립트 서브프로세스 (120s 타임아웃)
- `.claude/skills/x-integration/agent.ts:22-52` — 컨테이너->호스트 IPC 브릿지
- `src/ipc.ts:29` — IPC 워처 (폴링 기반)

**고유 설계**: 브라우저가 **호스트에서** 실행됨 (컨테이너 밖). X가 자동화 탐지/차단을 하기 때문에 실제 Chrome + 실제 사용자 세션을 사용. `--disable-blink-features=AutomationControlled`, `ignoreDefaultArgs: ['--enable-automation']` 등 안티탐지 설정.

**인증**: 1회 수동 로그인(`setup.ts`) -> Chrome 프로필에 세션 저장 -> 이후 재사용.

#### IronClaw — "테스트 전용"

브라우저 자동화가 에이전트 도구가 아님. `tests/e2e/conftest.py`에서 Playwright Python으로 웹 게이트웨이 UI를 E2E 테스트하는 용도. 에이전트는 `web_fetch` (HTTP 기반)로 웹에 접근.

---

## 3. 도구/액션 아키텍처 비교

### 3.1 도구 정의 방식 비교

| 구현체 | 정의 방식 | 스키마 형식 | 코드 |
|--------|-----------|------------|------|
| **IronClaw** | Rust `Tool` trait (async, 10 메서드) | JSON Schema | `src/tools/tool.rs:178-266` |
| **ZeroClaw** | Rust `Tool` trait (5 메서드) + Python `@tool` | JSON Schema | `src/tools/traits.rs:22-43` |
| **PicoClaw** | Go `Tool` interface (5 메서드) | `map[string]any` | `pkg/tools/base.go:5-11` |
| **OpenClaw** | TS `AgentTool<TParams, TResult>` + TypeBox | JSON Schema (TypeBox) | `src/agents/tools/common.ts:8` |
| **Nanobot** | Python `Tool` ABC (4 메서드) | JSON Schema | `nanobot/agent/tools/base.py:7-104` |
| **NanoClaw** | SKILL.md + manifest.yaml | YAML 메타데이터 | `skills-engine/types.ts:1-22` |
| **TinyClaw** | SKILL.md (절차적 문서) | 없음 (CLI 위임) | `.agents/skills/*/SKILL.md` |
| **OpenFang** | Rust `Tool` trait + HAND.toml | JSON Schema + TOML | `crates/tools/src/lib.rs` |
| **OpenJarvis** | Python class + `@AgentRegistry.register()` 데코레이터 | JSON Schema (OpenAI 호환) | `agents/registry.py` |

### 3.2 도구 등록 방식 비교

| 구현체 | 등록 방식 | 동적 등록 | 이름 충돌 방지 |
|--------|-----------|----------|--------------|
| **IronClaw** | `ToolRegistry` (RwLock + HashMap) | [O] WASM/MCP | [O] 30개 빌트인 보호 |
| **ZeroClaw** | `all_tools_with_runtime()` 팩토리 | [X] (설정 기반 조건부) | [X] |
| **PicoClaw** | `ToolRegistry` (RWMutex + map) | [O] MCP | [WARN] 덮어쓰기 경고만 |
| **OpenClaw** | 정적 빌트인 + 플러그인 동적 | [O] 플러그인 팩토리 | [O] 빌트인 섀도잉 차단 |
| **Nanobot** | `ToolRegistry` (dict) | [O] MCP | [X] (마지막 등록 우선) |
| **NanoClaw** | 채널 자동 등록 + IPC 타입 | [O] 스킬 적용 | [X] |
| **TinyClaw** | 없음 (에이전트가 스킬 직접 읽음) | — | — |
| **OpenFang** | string dispatch (60 빌트인) + HAND.toml 선언 | [O] HAND.toml | [O] 빌트인 보호 |
| **OpenJarvis** | `@AgentRegistry.register()` 데코레이터 (동적) | [O] MCP + 데코레이터 | [X] (덮어쓰기) |

### 3.3 도구 실행 방식 비교

| 구현체 | 실행 방식 | 병렬 실행 | 타임아웃 |
|--------|-----------|----------|---------|
| **IronClaw** | WASM 격리 / Docker 컨테이너 / 직접 | [X] (순차) | 60s (도구별 설정) |
| **ZeroClaw** | 직접 async 호출 / LangGraph ToolNode | [X] (순차) | SecurityPolicy 연동 |
| **PicoClaw** | goroutine 병렬 실행 + WaitGroup | [O] | [X] (도구 내부) |
| **OpenClaw** | 직접 호출 + 플러그인 훅 (before/after) | [X] | 도구별 |
| **Nanobot** | async 직접 호출 | [X] | [X] |
| **NanoClaw** | 컨테이너->호스트 IPC | [X] | 60s (IPC 폴링) |
| **TinyClaw** | CLI 서브프로세스 (spawn) | [X] | [X] |
| **OpenFang** | WASM Dual Metering + 직접 Rust 실행 | [X] (순차) | WASM 연료 기반 |
| **OpenJarvis** | ThreadPoolExecutor (max_workers=len(tool_calls)) | [O] | [X] (도구 내부) |
| **NemoClaw** | 컨테이너 내부 전체 실행 (OpenClaw plugin commands) | [X] | Docker 자원 제한 |

**주목**: PicoClaw(goroutine+WaitGroup)와 OpenJarvis(ThreadPoolExecutor) 두 구현체가 도구 병렬 실행을 지원한다. 동일한 동기(LLM의 다중 tool call 요청)에서 출발했지만 각자 언어의 관용구를 따른다.

### 3.4 내장 도구 수량 비교

| 구현체 | 내장 도구 수 | 주요 범주 |
|--------|-------------|----------|
| **IronClaw** | ~35 | I/O, 셸, 메모리, 잡 관리, 스킬, 루틴, 빌더, 메시지 |
| **ZeroClaw** | ~40 (Rust) + 7 (Python) | 파일, 셸, 브라우저, 검색, 메모리, cron, SOP, 하드웨어 |
| **OpenClaw** | ~25+ | 브라우저, 웹, 메시지, 이미지, PDF, TTS, 세션, 캔버스 |
| **PicoClaw** | 17 + 동적 MCP | 파일, 셸, 웹, 메시지, cron, 스폰, I2C/SPI |
| **Nanobot** | 9 + 동적 MCP | 파일, 셸, 웹, 메시지, 스폰, cron |
| **NanoClaw** | 10 IPC 타입 | 스케줄, 리프레시, X 액션, 메시지 |
| **TinyClaw** | 0 네이티브 | CLI 위임 (claude/codex/opencode) |
| **OpenFang** | 60 빌트인 | 파일, 셸, 웹, 메모리, 메시지, 시스템, A2A |
| **OpenJarvis** | 에이전트 위임 중심 | 7종 에이전트 (simple/react/orchestrator/rlm/native_react/claude_code/openhands/monitor_operative) |
| **NemoClaw** | plugin commands만 (launch, migrate, connect, status, logs, eject) | 샌드박스 수명주기 관리 전용 |

### 3.5 확장 메커니즘 비교

| 구현체 | MCP | WASM | 플러그인/스킬 | CLI | REST | A2A |
|--------|-----|------|-------------|-----|------|-----|
| **IronClaw** | [O] (Streamable HTTP + OAuth) | [O] (Component Model, WASI P2) | [X] | [X] | [X] | [X] |
| **ZeroClaw** | [X] | [O] (WasmRuntime) | [X] | [X] | [X] | [X] |
| **PicoClaw** | [O] (ClientSession) | [X] | [X] | [X] | [X] | [X] |
| **OpenClaw** | [O] (플러그인 경유) | [X] | [O] (24 훅 + API) | [X] | [O] (registerHttpRoute) | [X] |
| **Nanobot** | [O] (stdio + HTTP) | [X] | [X] | [X] | [X] | [X] |
| **NanoClaw** | [X] | [X] | [O] (SKILL.md + 3-way merge) | [X] | [X] | [X] |
| **TinyClaw** | [X] | [X] | [O] (Hook 플러그인) | [O] (프로바이더) | [X] | [X] |
| **OpenFang** | [O] (MCP 양방향) | [O] (WASM Dual Metering) | [O] (HAND.toml Hands) | [X] | [X] | [O] (Google A2A spec) |
| **OpenJarvis** | [O] (parallel_tools=True) | [X] | [X] | [X] | [X] | [O] (Google A2A spec) |
| **NemoClaw** | [X] | [X] | [X] | [O] (OpenClaw plugin commands) | [X] | [X] |

---

## 4. 개별 분석 요약

### 4.1 IronClaw — "보안 최우선 아키텍처"

**도구 시스템 하이라이트:**
- Rust `Tool` trait: 10개 메서드 (실행, 스키마, 비용 추정, 승인 요구, 타임아웃, 도메인, 레이트 리밋)
- `ToolDomain`: `Orchestrator` (메인 프로세스) vs `Container` (Docker) 분리
- `ApprovalRequirement`: `Never | UnlessAutoApproved | Always` — 도구별 HITL 승인
- WASM 샌드박스: Wasmtime + WIT 인터페이스 + WASI P2, 메모리 10MB, 연료 10M, 타임아웃 60s
- **Zero-Exposure 크레덴셜**: WASM은 `secret-exists()`만 호출 가능, 값 접근 불가. HTTP 요청 시 프록시 레이어에서 자동 주입.
- 컴파일 캐시: Wasmtime 디스크 캐시 (10-50x 재시작 속도)
- MCP: Streamable HTTP + OAuth PKCE + 자동 토큰 갱신

**핵심 코드:**
- `src/tools/tool.rs:178-266` — Tool trait
- `src/tools/registry.rs` — ToolRegistry (30개 빌트인 보호)
- `src/tools/wasm/limits.rs` — WASM 리소스 제한
- `src/tools/wasm/credential_injector.rs` — 크레덴셜 주입
- `src/sandbox/proxy/http.rs` — Docker 네트워크 프록시

### 4.2 ZeroClaw — "유연한 멀티 런타임"

**도구 시스템 하이라이트:**
- Rust `Tool` trait (5 메서드) + Python `@tool` 데코레이터 (LangChain)
- 설정 기반 조건부 등록: `has_shell_access`, `browser_config.enabled` 등
- 3중 브라우저 백엔드 (교체 가능)
- Computer Use 사이드카 통합 (좌표/윈도우 허용목록)
- Composio 통합: API 키 하나로 수백 개 외부 서비스 접근
- SOP (Standard Operating Procedure) 워크플로우 도구 5종

**핵심 코드:**
- `src/tools/traits.rs:22-43` — Tool trait
- `src/tools/browser.rs:77` — BrowserBackendKind enum
- `src/tools/browser.rs:136-199` — BrowserAction enum (16종)
- `python/zeroclaw_tools/tools/base.py` — Python @tool 데코레이터

### 4.3 OpenClaw — "풀스택 에이전트 플랫폼"

**도구 시스템 하이라이트:**
- TypeBox 기반 JSON Schema + 24개 플러그인 훅
- 플러그인 API: `registerTool`, `registerHook`, `registerHttpRoute`, `registerChannel`, `registerGatewayMethod` 등
- ACP (Agent Client Protocol): 5,000 세션 관리, 20+ 슬래시 커맨드
- 브라우저 도구: 50+ 기능 (업계 최대 범위)
- 외부 콘텐츠 안전성: `wrapExternalContent()` — untrusted 태그로 프롬프트 인젝션 방어
- 크기 제한: web_fetch 50K자, DOM 800노드, aria 2000노드, 네트워크 500요청

**핵심 코드:**
- `src/agents/tools/browser-tool.ts` — 브라우저 도구
- `src/browser/pw-session.ts:341` — CDP 연결
- `src/plugins/types.ts:310-334` — 24 훅 정의
- `src/security/external-content.ts` — 외부 콘텐츠 래핑

### 4.4 NanoClaw — "스킬 패키지 시스템"

**도구 시스템 하이라이트:**
- SKILL.md 2모드: 절차적(문서만) vs 구조적(manifest.yaml + 코드)
- 3-way merge 스킬 적용: `git merge-file`로 사용자 수정 보존
- 컨테이너<->호스트 IPC: 파일 기반 폴링, 그룹별 네임스페이스 격리
- 시크릿 전달: stdin JSON (디스크/환경변수 아님)
- 채널 통합: WhatsApp, Telegram, Discord, Slack, Gmail, X/Twitter

**핵심 코드:**
- `skills-engine/apply.ts` — 스킬 적용 엔진 (3-way merge)
- `skills-engine/types.ts:1-22` — manifest.yaml 스키마
- `src/ipc.ts:29` — IPC 워처
- `src/container-runner.ts:258` — 컨테이너 에이전트 실행

### 4.5 Nanobot — "OpenAI 호환 경량 프레임워크"

**도구 시스템 하이라이트:**
- Python `Tool` ABC: `name`, `description`, `parameters` (JSON Schema), `execute`
- `ToolRegistry`: 동적 등록, 파라미터 검증, OpenAI 스키마 변환
- MCP 래핑: `MCPToolWrapper` — stdio/HTTP MCP 서버를 Tool로 변환
- 결과 크기 제한: 500자 (`_TOOL_RESULT_MAX_CHARS`)
- 셸 안전성: deny 패턴 기반 명령어 차단

**핵심 코드:**
- `nanobot/agent/tools/base.py:7-104` — Tool ABC
- `nanobot/agent/tools/registry.py:8-67` — ToolRegistry
- `nanobot/agent/tools/mcp.py:14-99` — MCP 래퍼

### 4.6 PicoClaw — "Go 네이티브 병렬 실행"

**도구 시스템 하이라이트:**
- Go `Tool` interface + `ContextualTool`, `AsyncTool` 확장 인터페이스
- **유일하게 도구 병렬 실행 지원**: goroutine + WaitGroup
- 결정론적 도구 순서: `sortedToolNames()` — KV 캐시 안정성
- 하드웨어 접근: I2C/SPI 버스 (Linux)
- MCP 래핑: `MCPTool` — 이름 새니타이징 포함
- 4종 결과 타입: Normal, Silent, Error, Async

**핵심 코드:**
- `pkg/tools/base.go:5-11` — Tool interface
- `pkg/tools/registry.go:14-182` — ToolRegistry (RWMutex)
- `pkg/tools/toolloop.go:125-158` — goroutine 병렬 실행

### 4.7 TinyClaw — "CLI 오케스트레이터"

**도구 시스템 하이라이트:**
- 네이티브 도구 시스템 없음 — 외부 CLI(claude/codex/opencode) 서브프로세스에 위임
- 3 프로바이더: Anthropic (claude), OpenAI (codex), OpenCode (opencode)
- JSONL 파싱: Codex/OpenCode의 스트리밍 출력 처리
- 플러그인 훅: `transformOutgoing`/`transformIncoming`
- 스킬: `.agents/skills/*/SKILL.md` — 에이전트가 직접 읽고 해석

**핵심 코드:**
- `src/lib/invoke.ts:1-181` — CLI 서브프로세스 실행
- `src/lib/types.ts:1-8` — AgentConfig (프로바이더/모델)
- `src/lib/plugins.ts:1-223` — 플러그인 시스템

### 4.8 OpenJarvis — "멀티 에이전트 오케스트레이터"

**도구 시스템 하이라이트:**
- **7종 에이전트 타입**: `simple`, `react`, `orchestrator`, `rlm`, `native_react`, `claude_code`, `openhands`, `monitor_operative` — 태스크 유형에 따라 적합한 에이전트를 선택
- **`@AgentRegistry.register()` 데코레이터**: 에이전트 클래스를 런타임에 동적 등록. 에이전트가 곧 도구인 설계 — 하위 에이전트 호출이 도구 호출과 동등한 추상화 레벨
- **병렬 도구 실행**: `concurrent.futures.ThreadPoolExecutor(max_workers=len(tool_calls))`로 LLM이 요청한 여러 tool call을 동시 실행. OpenJarvis와 PicoClaw만이 이 패턴을 채택
- **MCP 표준 지원**: `parallel_tools=True` 옵션으로 MCP 서버와의 병렬 도구 호출 지원
- **A2A 프로토콜**: Google Agent-to-Agent spec 지원 (OpenFang과 동일). 에이전트 간 표준화된 메시지 교환
- **Loop Guard**: 반복 tool call 탐지 및 컨텍스트 압축 — 무한 루프 방지
- **브라우저 자동화**: 직접 구현 없음. `openhands`/`native_openhands` 에이전트 통합을 통해 간접 지원

**핵심 코드:**
- `agents/registry.py` — `@AgentRegistry.register()` 데코레이터 및 에이전트 등록
- `agents/base.py` — 에이전트 기반 클래스 (7종 공통 인터페이스)
- `tools/executor.py` — ThreadPoolExecutor 병렬 도구 실행
- `protocols/a2a.py` — Google A2A 프로토콜 구현

### 4.9 OpenFang — "60 빌트인 + WASM Dual Metering + 채널 어댑터 브라우저"

**도구 시스템 하이라이트:**
- **60개 빌트인 도구**: 파일, 셸, 웹, 메모리, 메시지, 시스템, A2A — 분석 대상 중 최다
- **WASM Dual Metering**: CPU 사이클 + 메모리 할당 이중 계량. 기존 fuel 단일 계량 대비 정밀한 자원 제어 및 과금 가능
- **HAND.toml Hands 시스템**: 플러그인 생태계 + FangHub 마켓플레이스. HAND.toml 선언만으로 도구 등록
- **MCP 양방향**: MCP 클라이언트(외부 서버 소비) + MCP 서버(내부 도구 노출) 동시 지원
- **A2A 프로토콜**: Google Agent-to-Agent spec 구현. OpenJarvis와 함께 A2A를 채택한 2개 구현체 중 하나
- **브라우저를 채널 어댑터로 통합**: 브라우저가 별도 도구가 아닌 채널 시스템의 어댑터로 깊이 통합. 다른 구현체들이 브라우저를 "하나의 도구"로 취급하는 것과 근본적으로 다른 아키텍처
- **18종 Capability + 도구별 권한 선언**: 각 도구가 필요한 Capability를 TOML에 선언. 런타임에 Capability 부여 여부로 도구 실행 허가

**핵심 코드:**
- `crates/tools/src/lib.rs` — Tool trait + 60 빌트인 string dispatch
- `crates/wasm/src/metering.rs` — WASM Dual Metering
- `crates/hands/src/lib.rs` — Hands 플러그인 시스템
- `crates/capability/src/lib.rs` — 18종 Capability 시스템

### 4.10 NemoClaw — "샌드박스 플러그인 — plugin commands만"

**도구 시스템 하이라이트:**
- **도구 시스템 없음**: NemoClaw 자체 도구 정의/등록/실행 시스템 없음. OpenClaw 플러그인 API 커맨드(launch, migrate, connect, status, logs, eject)만 제공
- **10개 네트워크 정책 프리셋 (커넥터)**: 도구가 아닌 "네트워크 환경 설정"으로서의 확장. 커넥터 선택으로 egress 정책 프리셋 적용
- **도구 실행 = 컨테이너 격리**: NemoClaw가 제공하는 핵심 가치는 "도구 구현"이 아니라 "모든 도구 실행을 컨테이너로 격리". OpenClaw의 모든 도구가 자동으로 샌드박스 안에서 실행됨
- **브라우저 없음**: 샌드박스 내부의 OpenClaw가 브라우저 자동화를 처리. NemoClaw 레이어에서는 투명
- **확장 = blueprint 수정**: 새 기능 추가는 도구 코드가 아닌 blueprint 파일 수정으로 이루어짐

**핵심 코드:**
- `nemoclaw/commands/` — plugin commands (launch, migrate, connect, status, logs, eject)
- `nemoclaw/blueprint/` — 환경 정의 (10 네트워크 정책 프리셋)
- `nemoclaw/sandbox/network.py` — 네트워크 네임스페이스 격리

---

## 5. 핵심 설계 패턴

### 패턴 1: "도구 정의의 3가지 철학"

세 가지 근본적으로 다른 접근이 존재한다:

1. **타입 시스템 기반** (IronClaw, ZeroClaw, PicoClaw): Rust trait / Go interface로 컴파일 타임 안전성. 도구가 코드.
2. **스키마 기반** (Nanobot, OpenClaw): JSON Schema로 도구를 선언적으로 정의. 도구가 데이터.
3. **문서 기반** (NanoClaw, TinyClaw): SKILL.md 마크다운으로 절차를 기술. 도구가 지식.

각각의 트레이드오프:
- 타입 기반: 안전하지만 개발 비용 높음
- 스키마 기반: 유연하지만 런타임 검증 필요
- 문서 기반: 가장 쉽지만 실행 일관성 없음

### 패턴 2: "격리 수준의 스펙트럼"

도구 실행의 격리 수준이 5단계로 분포:

```
무격리          프로세스        컨테이너              WASM              프록시+WASM
TinyClaw       PicoClaw       NanoClaw             ZeroClaw          IronClaw
               Nanobot        OpenClaw(Docker)     OpenFang(WASM DM)
               OpenClaw       NemoClaw(Docker+OS4중)
               OpenJarvis
```

IronClaw의 WASM+프록시 이중 격리가 가장 안전하지만, 도구 개발 복잡도도 가장 높다 (WIT 인터페이스 + Component Model 이해 필요). OpenFang의 WASM Dual Metering은 CPU+메모리 이중 계량으로 IronClaw에 근접한 격리 수준을 제공한다. NemoClaw는 Docker+Landlock+seccomp+네트워크 네임스페이스의 4중 OS 격리로 컨테이너 계층에서 최강의 격리를 달성한다. OpenJarvis는 격리 없이 프로세스 내 직접 실행 — 에이전트 위임(openhands 통합)으로 간접 격리를 대신한다.

### 패턴 3: "MCP 래핑 패턴의 수렴"

6개 구현체가 동일한 패턴을 독립적으로 구현:

```
MCP Server (외부) -> list_tools() -> 도구 목록 획득
                  -> Tool 래퍼 생성 (MCPToolWrapper / MCPTool)
                  -> ToolRegistry에 등록
                  -> execute() -> call_tool() -> MCP Server
```

차이점:
- **IronClaw**: Streamable HTTP + OAuth PKCE + 자동 토큰 갱신
- **Nanobot**: stdio + HTTP, `mcp_{server}_{tool}` 네이밍
- **PicoClaw**: `MCPManager` interface + 이름 새니타이징
- **OpenClaw**: 플러그인 시스템 경유
- **OpenFang**: MCP 양방향 (클라이언트 + 서버 동시)
- **OpenJarvis**: `parallel_tools=True`로 MCP 도구 병렬 실행 — 유일하게 MCP 레벨에서 병렬성 명시

### 패턴 4: "크레덴셜 주입의 2가지 전략"

| 전략 | 구현체 | 방법 |
|------|--------|------|
| **프록시 주입** | IronClaw | HTTP 프록시에서 URL 호스트 매칭 -> 헤더 주입. 도구 코드는 시크릿 접근 불가. |
| **환경변수/stdin** | NanoClaw, Nanobot, ZeroClaw | 환경변수 또는 stdin으로 전달. 도구 코드가 시크릿 접근 가능. |
| **없음** | TinyClaw, PicoClaw | 보안 메커니즘 없음 또는 최소. |

IronClaw의 프록시 주입이 "Zero-Exposure" 모델로 가장 안전 — 도구 코드에 버그가 있어도 시크릿이 노출되지 않음.

### 패턴 5: "결과 처리의 안전성 스펙트럼"

| 구현체 | 크기 제한 | 안전성 필터링 | 특수 처리 |
|--------|----------|-------------|----------|
| **OpenClaw** | web_fetch 50K, DOM 800, aria 2000, net 500 | `untrusted` 태그 래핑 | `tool_result_persist` 훅 |
| **Nanobot** | 500자 하드 리밋 | [X] | [X] |
| **IronClaw** | 도구별 설정 | [X] | ToolOutput에 cost/duration 포함 |
| **PicoClaw** | [X] | SilentResult (LLM 미노출) | AsyncResult (비동기) |
| **ZeroClaw** | HttpRequest/WebFetch 설정 | [X] | 페이지네이션 (offset/limit) |
| **NanoClaw** | [X] | [X] | sentinel 마커 기반 파싱 |

---

## 6. 교차 분석 및 논의

### 논의 1: 브라우저 자동화와 도구 아키텍처의 결합 패턴

세 가지 결합 패턴이 발견됨:

1. **도구로서의 브라우저** (OpenClaw, ZeroClaw): 브라우저가 다른 도구(shell, file)와 동등한 레벨의 도구. LLM이 언제 브라우저를 쓸지 자율 판단.
2. **IPC 브릿지** (NanoClaw): 브라우저가 호스트에서 실행되고 컨테이너 에이전트와 파일 기반 IPC로 통신. 보안(격리)과 탐지 회피(실제 Chrome) 두 마리를 잡음.
3. **외부 CLI** (TinyClaw): 브라우저가 스킬 문서로만 존재하고, 에이전트가 bash로 CLI 호출. 가장 느슨한 결합.

### 논의 2: 24시간 메신저 에이전트에서의 실용성 평가

| 요구사항 | 최적 구현체 | 이유 |
|----------|------------|------|
| 웹 리서치 자동화 | OpenClaw | 50+ 브라우저 기능, SSRF 방어, 네트워크 모니터링 |
| 소셜 미디어 자동화 | NanoClaw | 안티탐지 설계, 실제 Chrome 세션, X 통합 |
| 외부 API 통합 | IronClaw | MCP + WASM + Zero-Exposure 크레덴셜 |
| 빠른 프로토타입 | Nanobot | 9개 도구 + MCP 래핑으로 최소한의 셋업 |
| 다중 LLM 프로바이더 | TinyClaw | claude/codex/opencode CLI 호환 |
| 하드웨어 IoT 통합 | PicoClaw | I2C/SPI 네이티브 지원 |
| **멀티 에이전트 오케스트레이션** | **OpenJarvis** | 7종 에이전트 타입, A2A 프로토콜, openhands 통합으로 복잡한 태스크 분산 |
| **60개 빌트인 + Capability 보안** | **OpenFang** | 최다 빌트인 도구 + WASM Dual Metering + 18종 Capability로 보안과 기능의 최적 균형 |
| **기존 에이전트의 즉시 샌드박스화** | **NemoClaw** | OpenClaw 에이전트를 코드 변경 없이 Docker+Landlock 4중 격리 환경으로 이전 |

### 논의 3: 확장성 비교 (새 도구 추가 용이성)

새 도구를 추가하는 데 필요한 단계 수:

| 구현체 | 단계 | 복잡도 |
|--------|------|--------|
| **Nanobot** | 1. Tool 서브클래스 + register | [1/5] |
| **PicoClaw** | 1. Tool interface 구현 + register | [1/5] |
| **TinyClaw** | 1. SKILL.md 작성 | [1/5] |
| **OpenJarvis** | 1. Python class + `@AgentRegistry.register()` | [1/5] |
| **NanoClaw** | 1. manifest.yaml + add/ + modify/ | [2/5] |
| **NemoClaw** | 1. blueprint 파일 수정 (도구 추가 없음, 환경만 변경) | [2/5] |
| **OpenClaw** | 1. 플러그인 패키지 + registerTool | [3/5] |
| **ZeroClaw** | 1. Tool impl + 팩토리 함수 수정 | [3/5] |
| **OpenFang** | 1. Rust Tool impl + HAND.toml 선언 | [4/5] |
| **IronClaw** | 1. WASM 컴포넌트 작성 + WIT + 컴파일 | [5/5] |

### 논의 4: security_report.md와의 교차 검증

보안 보고서의 Tier 분류와 도구 보안이 정확히 일치함:

| 보안 Tier | 도구 격리 수준 | 크레덴셜 보호 | 도구 실행 안전성 |
|----------|--------------|-------------|----------------|
| **Tier S** (OpenFang) | WASM Dual Metering + 18종 Capability | OS Keyring + 암호화 볼트 + per-tool 스코핑 | CPU+메모리 이중 계량 + Capability-gated |
| **Tier A+** (NemoClaw) | Docker + Landlock + seccomp + 네트워크 네임스페이스 (4중 OS 격리) | credentials.json mode 600 + OpenShell 주입 | 컨테이너 내부 전체 격리 |
| **Tier 1** (IronClaw, ZeroClaw) | WASM/Docker + 프록시 | 암호화 볼트 + 프록시 주입 | 연료/메모리/타임아웃 제한 |
| **Tier 2** (NanoClaw, OpenClaw) | Docker 컨테이너 | stdin/환경변수 | 도구 허용목록 |
| **Tier 3** (Nanobot, PicoClaw) | 프로세스 내 | 평문 | deny 패턴 |
| **Tier 4** (TinyClaw, OpenJarvis) | 없음/최소 | 없음/최소 | Loop Guard (OpenJarvis만) |

OpenJarvis는 도구 격리나 크레덴셜 보호 메커니즘이 없으나, Loop Guard(반복 tool call 탐지+컨텍스트 압축)로 무한 루프를 방지하는 최소한의 런타임 안전장치는 갖추고 있다. NemoClaw는 자체 도구 시스템이 없지만, OpenClaw의 모든 도구 실행을 4중 OS 격리로 자동 보호한다.

### 논의 5: session_context_report.md와의 교차 검증

세션 보고서에서 발견한 "프로젝트 세션 추상화 부재"가 도구 아키텍처에서도 확인됨:

- **서브에이전트 스폰 도구를 가진 구현체**: OpenClaw (`sessions_spawn`), Nanobot (`spawn`), NanoClaw (컨테이너 위임), PicoClaw (`spawn`), IronClaw (`create_job`), ZeroClaw (`delegate`), OpenJarvis (7종 에이전트 타입 위임)
- **프로젝트 수명주기 관리 도구를 가진 구현체**: 없음

OpenJarvis는 에이전트 타입 선택(`orchestrator`, `monitor_operative` 등)으로 서브에이전트를 구조적으로 분리하지만, "작업 디렉토리 생성 -> 오케스트레이터 스폰 -> 서브에이전트 할당 -> 결과 수집 -> 아카이브"를 자동화하는 프로젝트 수명주기 도구는 역시 부재하다. 세션 보고서에서 지적한 "아직 아무도 안 만든 층위"는 10개 구현체 모두에서 빈 공간으로 남아 있다.

### 논의 6: idea.md [열린 질문]에 대한 추가 인사이트

**질문 5 (동적 도구 위험도 평가)**: IronClaw의 `ApprovalRequirement` 메커니즘(`Never | UnlessAutoApproved | Always`)이 가장 근접한 답. 하지만 이것도 도구 개발자가 수동으로 설정하는 것이지, 새로 등록되는 MCP 도구의 위험도를 **자동 분류**하지는 않는다.

**질문 8 (E-Stop 메신저 통합)**: NanoClaw의 IPC 아키텍처가 힌트를 준다. IPC 워처(`src/ipc.ts`)가 이미 모든 도구 호출을 중재하고 있으므로, 여기에 E-Stop 로직을 삽입하면 "텔레그램에서 /estop 명령 -> IPC 워처가 모든 pending 작업 거부"가 가능하다.

**OpenJarvis 추가 인사이트 (A2A 프로토콜)**: OpenJarvis와 OpenFang 두 구현체가 Google A2A spec을 독립적으로 채택했다. 에이전트 간 표준 프로토콜이 MCP(도구 표준)에 이어 수렴 중임을 시사한다. 향후 에이전트 생태계에서 MCP(도구) + A2A(에이전트) 이중 프로토콜 스택이 사실상의 표준이 될 가능성이 높다.

**OpenJarvis Loop Guard**: 반복 tool call 탐지 + 컨텍스트 압축 패턴은 다른 9개 구현체에 없는 고유 기능이다. 장기 실행 에이전트에서 LLM이 동일한 도구를 반복 호출하는 루프에 빠지는 문제를 런타임에서 자동 탐지하는 접근법으로, 별도 타임아웃이나 연료 제한 없이도 루프를 차단할 수 있다는 점에서 실용적이다.

---

> **다음 조사 후보:**
> - 각 프레임워크의 프롬프트 엔지니어링 전략 (시스템 프롬프트 구조, few-shot 패턴)
> - 비용 추적/제한 메커니즘 심층 분석
> - 실시간 스트리밍/SSE 아키텍처 비교
