# 실세계 권한 부여 보안 전략 — 10개 Claw 코드 기반 비교 분석

> **조사 일자**: 2026-03-05 (OpenJarvis 추가: 2026-03-14)
> **조사 방법**: 7개 scientist 에이전트가 각 레포의 보안/권한 관련 소스코드를 병렬 심층 분석 (OpenJarvis는 단독 추가 분석)
> **핵심 질문**: "에이전트에게 실세계 권한을 안전하게 부여하기 위해 각 프레임워크가 어떤 보안 전략을 채택했는가?"

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. [7대 보안 영역 비교 매트릭스](#2-7대-보안-영역-비교-매트릭스)
3. [개별 분석 요약](#3-개별-분석-요약)
4. [핵심 보안 패턴 5가지](#4-핵심-보안-패턴-5가지)
5. [session_context_report.md와 교차 검증](#5-교차-검증)
6. [결론 및 논의](#6-결론-및-논의)

---

## 1. Executive Summary

10개 구현체의 보안/권한 코드를 분석한 결과, **4개의 보안 성숙도 계층**으로 분류된다:

| 계층 | 구현체 | 특징 |
|------|--------|------|
| **Tier 1: Defense-in-Depth** | IronClaw, ZeroClaw | 암호화 볼트 + WASM/Docker 이중 샌드박스 + 다층 인젝션 방어 + HITL 승인 + 비용 제한 |
| **Tier 2: Container-First** | NanoClaw, OpenClaw, **OpenJarvis** | Docker/subprocess 격리 + 도구 허용목록 + 자격증명 격리 (암호화 없음) + 부분적 인젝션 방어. OpenJarvis는 Prompt Injection Scanner 명시적 구현으로 Tier 2 상단에 위치 |
| **Tier 3: Denylist-Based** | Nanobot, PicoClaw | 정규식 기반 명령어 차단 + 파일시스템 제한 + 평문 자격증명 + HITL 없음 |
| **Tier 4: Minimal/None** | TinyClaw | 보안 메커니즘 최소 또는 해당 없음 (실험적 용도) |

**가장 주목할 발견 4가지:**

1. **암호화 볼트를 구현한 곳은 IronClaw과 ZeroClaw 뿐이다.** 나머지 8개는 전부 평문 저장. IronClaw는 AES-256-GCM + OS Keychain, ZeroClaw는 ChaCha20-Poly1305.
2. **Human-in-the-loop를 구현한 곳은 2개뿐이다.** IronClaw(도구별 승인), ZeroClaw(3단계 자율성 + E-Stop). 나머지 8개는 에이전트가 도구를 자율 실행.
3. **프롬프트 인젝션 방어에 전용 레이어를 둔 곳은 IronClaw, ZeroClaw, OpenJarvis 3개뿐이다.** IronClaw는 SafetyLayer 4중 방어, ZeroClaw는 PromptGuard 6패턴 탐지, **OpenJarvis는 10개 프레임워크 중 유일하게 regex 기반 Prompt Injection Scanner를 명시적으로 구현** (4개 위협 수준: LOW/MEDIUM/HIGH/CRITICAL). 나머지는 레이블링(Nanobot) 또는 마커 래핑(OpenClaw) 수준.
4. **Taint Tracking을 구현한 곳은 OpenJarvis뿐이다 (10개 중 유일).** 4-label 분류(PII/SECRET/USER_PRIVATE/EXTERNAL)와 SINK_POLICY로 데이터 흐름을 제어한다.

---

## 2. 7대 보안 영역 비교 매트릭스

### 2.1 권한/인가 시스템

| 구현체 | 접근 방식 | 도구 수준 제어 | 핵심 코드 |
|--------|-----------|---------------|-----------|
| **OpenClaw** | deny/allow 목록 + ACP 에이전트 정책 + 채널별 DM/그룹 정책 | O (도구별 deny/allow) | `src/agents/sandbox/tool-policy.ts:16` |
| **Nanobot** | 채널별 `allow_from` 허용목록 | X (발신자만 제어) | `nanobot/channels/base.py:61-72` |
| **NanoClaw** | 2-tier IPC 그룹 권한 (main vs non-main) + `allowedTools` 화이트리스트 | O (도구 화이트리스트) | `src/ipc.ts:78-93`, `container/agent-runner/src/index.ts:427-436` |
| **IronClaw** | Bearer + ephemeral 토큰 + Skills 신뢰 모델 (Trusted/Installed) | O (신뢰 수준별 감쇠) | `src/skills/attenuation.rs:55` |
| **ZeroClaw** | 3단계 AutonomyLevel (ReadOnly/Supervised/Full) + 명령어 위험도 분류 | O (High/Medium/Low 분류) | `src/security/policy.rs:10-18` |
| **PicoClaw** | AllowFrom + 서브에이전트 spawn 허용목록 + 에이전트 바인딩 | △ (spawn만 제어) | `pkg/tools/spawn.go:78-82` |
| **TinyClaw** | Pairing 시스템만 | X | `src/lib/pairing.ts:73-81` |
| **OpenJarvis** | RBAC 10종 (Python+Rust 이중 구현) + Taint Tracking SINK_POLICY | O (도구별 taint label 기반 차단) | RBAC 모듈, `sink_policy.py` |

### 2.2 자격증명 관리

| 구현체 | 저장 방식 | 암호화 | OS Keychain | 환경변수 격리 | 핵심 코드 |
|--------|-----------|--------|-------------|-------------|-----------|
| **OpenClaw** | `auth-profiles.json` (평문) | X | keyRef 참조 지원 | 20+ 패턴 자동 차단 | `src/agents/sandbox/sanitize-env-vars.ts:1-19` |
| **Nanobot** | `config.json` (평문) | X | 문서 권고만 | NANOBOT_ prefix | `nanobot/config/schema.py:239-244` |
| **NanoClaw** | stdin JSON 전달 → 즉시 삭제 | X | X | Bash hook으로 unset 강제 + .env shadow mount | `src/container-runner.ts:312-317` |
| **IronClaw** | AES-256-GCM 암호화 볼트 | **O** | macOS/Linux Keyring | secrecy 크레이트 제로화 + Zero-Exposure 프록시 주입 | `src/secrets/crypto.rs:71` |
| **ZeroClaw** | ChaCha20-Poly1305 AEAD | **O** | X (파일 기반 .secret_key) | 로그 자동 redact | `src/security/secrets.rs:1-22` |
| **PicoClaw** | `auth.json` (평문, 0600) | X | X | env 태그 지원 | `pkg/auth/store.go:41-44` |
| **TinyClaw** | `settings.json` (평문) | X | X | X (API로 노출) | `lib/setup-wizard.sh:364-386` |
| **OpenJarvis** | 평문 (환경변수/config) | X | X | Taint Tracking으로 SECRET label 전파 추적 | `taint_tracker.py`, `sink_policy.py` |

### 2.3 샌드박싱

| 구현체 | Docker | WASM | OS 격리 | 파일시스템 제한 | 핵심 코드 |
|--------|--------|------|---------|---------------|-----------|
| **OpenClaw** | O (non-root, seccomp/AppArmor 검증) | X | Docker 소켓 마운트 차단 | 호스트 경로 차단 목록 | `src/agents/sandbox/validate-sandbox-security.ts:18-33` |
| **Nanobot** | △ (root 실행) | X | X | `restrict_to_workspace` (기본 off) | `Dockerfile:34` |
| **NanoClaw** | O (non-root, 16개 민감경로 차단) | X | 그룹별 IPC 네임스페이스 | mount-security allowlist (fail-secure) | `src/mount-security.ts:29-47` |
| **IronClaw** | O (cap_drop ALL, ro rootfs) | **O** (wasmtime, 10MB/1천만fuel) | UID 1000 | 정책별 파일시스템 (RO/WS/Full) | `src/sandbox/container.rs:274`, `src/tools/wasm/limits.rs` |
| **ZeroClaw** | O | **O** (wasmi, 10억fuel) | Landlock/Firejail/Bubblewrap | workspace_only 기본 활성 | `src/security/detect.rs:8-71`, `src/runtime/wasm.rs:1-80` |
| **PicoClaw** | △ (기본 격리만) | X | `os.Root` 기반 샌드박스 | restrictToWorkspace 기본 활성 | `pkg/tools/filesystem.go:281-390` |
| **TinyClaw** | X | X | X | 디렉터리만 분리 | `--dangerously-skip-permissions` 항상 사용 |
| **OpenJarvis** | X (subprocess_sandbox) | X | SSRF 방지 + 감사로그 | SINK_POLICY 기반 데이터 흐름 제한 | `subprocess_sandbox.py`, `audit_log.py` |

### 2.4 도구 실행 안전성

| 구현체 | 셸 명령 제어 | 네트워크 제어 | 경로 순회 방어 | 출력 제한 | 핵심 코드 |
|--------|------------|-------------|--------------|----------|-----------|
| **OpenClaw** | 3단계(deny/allowlist/full) + 코드 정적 스캔 | ACP 채널 차단 | 심링크 우회 방지 | — | `src/infra/exec-approvals.ts:11` |
| **Nanobot** | 정규식 denylist + workspace 경로제한 | http/https만 허용 | `_resolve_path()` | 10,000자 | `nanobot/agent/tools/shell.py:26-36` |
| **NanoClaw** | allowedTools 화이트리스트 | 컨테이너 네트워크 | 폴더명 정규식 + ensureWithinBase | 10MB stdout/stderr | `src/group-folder.ts:5` |
| **IronClaw** | 프록시 경유 + 17개 도메인 화이트리스트 | **완전 프록시** | WASM 경로탐색 차단 | — | `src/sandbox/config.rs:134` |
| **ZeroClaw** | **5중 방어**(서브쉘/리다이렉션/tee/백그라운드/화이트리스트) | WASM 호스트 화이트리스트 | null바이트/URL인코딩/~user 차단 | — | `src/security/policy.rs:726-816` |
| **PicoClaw** | **77개 정규식** denylist + 커스텀 패턴 | — | `filepath.IsLocal()` + 심링크 추적 | 10,000자 | `pkg/tools/shell.go:29-77` |
| **TinyClaw** | X (제한 없음) | X | X | — | `--dangerously-skip-permissions` |
| **OpenJarvis** | RBAC 10종 + SINK_POLICY (web_search: PII+SECRET 금지, channel_send: SECRET 금지, code_interpreter: SECRET 금지) | SSRF 방지 | Taint label 기반 경로 제한 | 속도 제한 | `rbac.py`, `sink_policy.py` |

### 2.5 Human-in-the-Loop

| 구현체 | HITL 구현 | 승인 대상 | 긴급 정지 | 핵심 코드 |
|--------|----------|----------|----------|-----------|
| **OpenClaw** | **O** (실행 승인 요청 + 페어링) | 셸 명령, 위험 도구 | 타임아웃 fallback | `src/agents/bash-tools.exec-approval-request.ts:88-111` |
| **Nanobot** | X (자율 실행) | — | `/stop` 명령 | `nanobot/agent/loop.py:191-257` |
| **NanoClaw** | X (headless 설계) | — | @트리거 패턴만 | `allowDangerouslySkipPermissions: true` |
| **IronClaw** | **O** (`requires_approval()` → `AwaitingApproval`) | 승인 필요 도구, 자율잡 차단 | — | `src/agent/thread_ops.rs:356, 659` |
| **ZeroClaw** | **O** (Supervised 모드 + Yes/No/Always) | 중/고위험 명령 | **E-Stop** (4단계 + OTP) | `src/approval/mod.rs:156-185`, `src/security/estop.rs` |
| **PicoClaw** | X (자율 실행) | — | — | `pkg/tools/toolloop.go:134-158` |
| **TinyClaw** | △ (Pairing만, 도구 승인 없음) | 채널 접근만 | — | `src/lib/pairing.ts:73-81` |
| **OpenJarvis** | X (자율 실행) | — | — | RBAC + 감사로그로 사후 추적 |

### 2.6 비용/속도 제한

| 구현체 | Rate Limiting | 비용 예산 | 토큰/반복 제한 | 핵심 코드 |
|--------|-------------|----------|--------------|-----------|
| **OpenClaw** | 인증 실패 슬라이딩 윈도우 + ACP 세션 고정 윈도우 | 세션 비용 추적 (하드 한도 없음) | 프롬프트 2MB, 컨텍스트 16K 최소 | `src/gateway/auth-rate-limit.ts:78-81` |
| **Nanobot** | X | X | max_tool_iterations=40, max_tokens=8192 | `nanobot/config/schema.py:228-229` |
| **NanoClaw** | X | X | 동시 컨테이너 5개, 타임아웃 30분 | `src/config.ts:40-49` |
| **IronClaw** | **(user,tool)별 분당/시간당** | LLM 비용 DB 기록 | WASM 1천만 fuel, 메모리 10MB | `src/tools/rate_limiter.rs` |
| **ZeroClaw** | 시간당 20회 쓰기/실행 | **일별 $5 하드 한도** | — | `src/cost/tracker.rs:50-100` |
| **PicoClaw** | 제공자 지수 백오프 쿨다운 | X | RPM 설정, max_tokens=32768 | `pkg/providers/cooldown.go:183-207` |
| **TinyClaw** | X | X | 대화당 50메시지, 재시도 5회, TTL 30분 | `src/lib/conversation.ts:12` |
| **OpenJarvis** | 속도 제한 (구현) | X | — | `rate_limiter.py` |

### 2.7 프롬프트 인젝션 방어

| 구현체 | 전용 방어 레이어 | 접근 방식 | 핵심 코드 |
|--------|----------------|-----------|-----------|
| **OpenClaw** | △ (외부 콘텐츠 래핑) | 랜덤 마커 경계 + 유니코드 동형문자 정규화 + 15개 의심 패턴 탐지 (로깅만) | `src/security/external-content.ts:17-32` |
| **Nanobot** | X | 런타임 컨텍스트 레이블링, HTML 이스케이프 | `nanobot/agent/context.py:19` |
| **NanoClaw** | X | 스케줄 작업 신뢰 레이블, XML 이스케이프, 구조적 격리 | `container/agent-runner/src/index.ts:529-531` |
| **IronClaw** | **O** (SafetyLayer 4중) | Sanitizer(18패턴) + Validator + Policy + LeakDetector(15+시크릿패턴) | `src/safety/mod.rs:28` |
| **ZeroClaw** | **O** (PromptGuard 6패턴) | 시스템 오버라이드/역할혼란/JSON인젝션/시크릿추출/커맨드인젝션/탈옥 탐지 | `src/security/prompt_guard.rs` |
| **PicoClaw** | X | 없음 (셸 denylist이 간접 방어) | — |
| **TinyClaw** | X | 없음 (`[@teammate:]` 태그 파싱이 인젝션 벡터) | `src/lib/routing.ts:66` |
| **OpenJarvis** | **O** (Prompt Injection Scanner) | regex 기반 스캔 + 4개 위협 수준 분류 (LOW/MEDIUM/HIGH/CRITICAL) — 10개 프레임워크 중 명시적 Scanner 유일 | `prompt_injection_scanner.py` |

---

## 3. 개별 분석 요약

### 3.1 OpenClaw — "실용적 다층 방어"

**보안 철학**: 도구 수준 deny/allow + Docker 격리 + 환경변수 위생 + 외부 콘텐츠 마킹. 실용적이고 운영 가능한 수준의 보안.

- **강점**: 실행 승인 2단계 등록(race condition 방지), 스킬 코드 정적 스캔(child_process, eval, 크립토마이닝 탐지), 랜덤 마커+유니코드 정규화 인젝션 방어
- **약점**: 자격증명 평문 저장, allow 목록 비어있으면 전체 허용, 패턴 탐지가 로깅만

### 3.2 Nanobot — "최소 방어의 개인 에이전트"

**보안 철학**: 채널 허용목록 + 정규식 denylist로 최소한의 보호. 개인 사용 가정.

- **강점**: 중앙화된 ACL, deny→allow→경로 순서의 명확한 가드 체인
- **약점**: HITL/암호화/인젝션 방어 전무, Dockerfile root 실행, restrict_to_workspace 기본 off

### 3.3 NanoClaw — "컨테이너 격리의 모범"

**보안 철학**: Docker 컨테이너가 핵심 보안 경계. 시크릿 노출 최소화에 특히 주력.

- **강점**: stdin 시크릿 전달→즉시 삭제, Bash hook으로 환경변수 자동 unset, .env shadow mount(/dev/null), mount-security fail-secure, IPC 인가 테스트 12+
- **약점**: HITL 완전 부재, API 비용 제한 없음, 프롬프트 인젝션 전용 방어 없음

### 3.4 IronClaw — "엔터프라이즈급 보안"

**보안 철학**: Defense-in-Depth. 모든 7개 영역에서 최고 수준의 보안 구현.

- **강점**: AES-256-GCM 볼트 + OS Keychain + secrecy 제로화, WASM+Docker 이중 샌드박스, SafetyLayer 4중 인젝션 방어, Zero-Exposure 프록시(컨테이너는 시크릿에 접근 불가), 네트워크 17개 도메인 화이트리스트
- **약점**: 복잡성으로 인한 운영 부담, 비용 하드 한도 없음

### 3.5 ZeroClaw — "가장 정교한 정책 엔진"

**보안 철학**: 세밀한 정책 기반 제어 + AEAD 암호화 + E-Stop 긴급 제어.

- **강점**: 3단계 AutonomyLevel, 5중 셸 방어, ChaCha20-Poly1305 암호화, E-Stop(4단계+OTP), 일별 $5 비용 하드 한도, PromptGuard 6패턴 탐지, 다중 샌드박스 백엔드(Landlock/Firejail/Bubblewrap/Docker/WASM)
- **약점**: OS Keychain 미연동 (파일 기반 키), 인메모리 상태

### 3.6 PicoClaw — "Go 생태계의 실용주의"

**보안 철학**: 정규식 기반 광범위 차단 + Go의 `os.Root` 파일시스템 샌드박스.

- **강점**: 77개 denylist 패턴(분석 대상 중 최다), `os.Root` 기반 경로 탈출 방지, OAuth 2.0+PKCE, 지수 백오프 폴백
- **약점**: HITL/인젝션 방어 없음, OAuth 클라이언트 시크릿 소스코드 하드코딩, 차단 패턴 비활성화 가능

### 3.7 TinyClaw — "실험적 프레임워크"

**보안 철학**: 보안보다 기능 편의성 우선. 개인 실험 용도.

- **약점(전면적)**: 인증 없는 REST API, CORS 와일드카드, 평문 자격증명 API 노출, `--dangerously-skip-permissions` 항상 사용, 프롬프트 인젝션 방어 없음, `[@teammate:]` 태그 파싱이 인젝션 벡터
- **유일한 강점**: Pairing 코드 `crypto.randomBytes` 생성

### 3.8 OpenJarvis — "Taint Tracking + Prompt Injection Scanner의 선구자"

**보안 철학**: RBAC 기반 접근 제어 + Taint Tracking으로 데이터 흐름 추적 + Prompt Injection Scanner로 입력 위협 탐지. Docker 없이도 소프트웨어 계층 보안을 다층으로 구현.

- **강점**:
  - **Prompt Injection Scanner**: 10개 프레임워크 중 유일한 명시적 구현. regex 기반으로 4개 위협 수준(LOW/MEDIUM/HIGH/CRITICAL) 분류
  - **Taint Tracking 4-label**: PII/SECRET/USER_PRIVATE/EXTERNAL 레이블 전파. SINK_POLICY로 도구별 금지 레이블 지정 (web_search: PII+SECRET 금지, channel_send: SECRET 금지, code_interpreter: SECRET 금지)
  - **RBAC 10종 이중 구현**: Python+Rust 양측에서 권한 검사. 단일 언어 구현 대비 우회 난이도 높음
  - **SSRF 방지**: 서버사이드 요청 위조 방어 명시적 구현
  - **감사 로그**: 모든 도구 실행 기록
- **약점**:
  - WASM/Docker 컨테이너 격리 없음 → subprocess_sandbox만으로는 Tier 1 수준 미달
  - 자격증명 암호화 없음 (평문 저장)
  - HITL 없음 (자율 실행)
  - 비용/예산 하드 한도 없음

---

## 4. 핵심 보안 패턴 5가지

### 패턴 1: "자격증명 노출 최소화" 스펙트럼

```
Zero-Exposure (IronClaw)
  ↓ 프록시가 런타임 주입, 컨테이너는 값에 접근 불가
AEAD 암호화 (ZeroClaw)
  ↓ ChaCha20-Poly1305로 디스크 암호화
stdin 전달+즉시 삭제 (NanoClaw)
  ↓ 메모리에만 존재, 디스크/로그/환경변수 노출 방지
환경변수 차단 (OpenClaw)
  ↓ 20+ 패턴 자동 제거, 기본 저장은 평문
평문 저장 (Nanobot, PicoClaw, TinyClaw)
  ↓ OS 파일 권한에만 의존
```

**핵심 인사이트**: IronClaw의 "Zero-Exposure" 모델이 가장 안전하다. 컨테이너가 시크릿 값 자체에 절대 접근하지 못하고, 프록시가 HTTP 요청에 런타임으로 주입한다. 설령 컨테이너가 탈취되어도 시크릿은 안전하다.

### 패턴 2: "도구 실행 제어" 3가지 접근법

| 접근법 | 구현체 | 원리 | 우회 난이도 |
|--------|--------|------|-----------|
| **화이트리스트** | NanoClaw, IronClaw | 명시적으로 허용된 도구/명령만 실행 | 높음 |
| **블랙리스트** | Nanobot, PicoClaw | 위험 패턴을 정규식으로 차단 | 중간 (인코딩/별칭 우회 가능) |
| **무제한** | TinyClaw | 모든 명령 허용 | 방어 없음 |

ZeroClaw는 독특하게 **5중 파싱 기반 방어**(서브쉘/리다이렉션/tee/백그라운드/세그먼트별 화이트리스트)로 구문 수준에서 차단한다.

### 패턴 3: "샌드박스 전략" 매핑

```
                    격리 수준 높음
                         ↑
    IronClaw ─────── WASM + Docker (cap_drop ALL, ro rootfs)
    ZeroClaw ─────── WASM + Landlock/Firejail/Bubblewrap/Docker
    NanoClaw ─────── Docker (non-root, mount-security)
    OpenClaw ─────── Docker (seccomp/AppArmor 검증)
    OpenJarvis ───── subprocess_sandbox + SSRF 방지 (Docker 없음)
    PicoClaw ─────── os.Root 파일시스템 + Docker (기본)
    Nanobot ──────── Docker (root 실행!)
    TinyClaw ─────── 디렉터리 분리만
                         ↓
                    격리 수준 낮음
```

### 패턴 4: "Human-in-the-Loop" 설계 선택

| 설계 | 구현체 | 특징 |
|------|--------|------|
| **세밀한 도구별 승인** | IronClaw, ZeroClaw | 위험도별 승인/자동허용/항상승인 |
| **실행 승인 요청** | OpenClaw | 채널(Discord 등)로 승인 요청 발송 |
| **진입점 제어만** | TinyClaw (Pairing) | 접근 자체만 제한, 내부 실행은 자율 |
| **없음** | Nanobot, NanoClaw, PicoClaw | 완전 자율 실행 |

**가장 정교한 HITL**: ZeroClaw의 E-Stop 시스템. `KillAll` / `NetworkKill` / `DomainBlock` / `ToolFreeze` 4단계 긴급 정지 + OTP 없이 해제 불가.

### 패턴 5: "프롬프트 인젝션 방어" 계층 구조

```
Layer 5: 전용 탐지 + 차단 + 위협 수준 분류 (OpenJarvis Prompt Injection Scanner: LOW/MEDIUM/HIGH/CRITICAL)
Layer 4: 전용 탐지 + 차단 (IronClaw SafetyLayer, ZeroClaw PromptGuard)
Layer 3: 경계 마커 + 패턴 탐지 (OpenClaw external-content)
Layer 2: 레이블링 + 이스케이프 (Nanobot 컨텍스트 태그, NanoClaw 스케줄 태그)
Layer 1: 구조적 격리만 (PicoClaw denylist 간접 방어)
Layer 0: 방어 없음 (TinyClaw — 인젝션 벡터 존재)
```

---

## 5. 교차 검증

session_context_report.md(세션/컨텍스트 관리 조사)와 교차 검증한 결과:

| 이전 분류 | 보안 조사 결과 | 일치 여부 |
|-----------|-------------|----------|
| "IronClaw: 보안 계층 기반 격리" | WASM+Docker 이중 샌드박스 + SafetyLayer 4중 방어 확인 | **일치** — 보안이 격리의 핵심 수단 |
| "NanoClaw: 프로세스/컨테이너 격리" | Docker + mount-security + 시크릿 stdin 전달 확인 | **일치** — 컨테이너가 주 보안 경계 |
| "OpenClaw: 세션 키 기반 논리적 격리" | 도구 정책 + 환경변수 위생 + 실행 승인이 추가 | **보완** — 논리적 격리 위에 도구 수준 보안 레이어 존재 |
| "Nanobot: 세션 키 기반 논리적 격리" | 보안은 최소 수준 (denylist + ACL만) | **보완** — 격리는 논리적이지만 보안 레이어 부재 |
| "TinyClaw: 프로세스/컨테이너 격리" | 실제로는 `--dangerously-skip-permissions`로 격리 우회 | **수정 필요** — 격리 "우회"가 기본 설정 |

---

## 6. 결론 및 논의

### 논의 1: "실세계 권한 부여"는 아직 미해결 문제이다

idea.md에서 제기한 "에이전트에게 실세계 권한을 안전하게 주는가"에 대해:

- **완전히 해결한 구현체는 없다.** IronClaw과 ZeroClaw가 가장 가깝지만, 둘 다 "사전 정의된 도구 목록"에 대한 보안이지 "임의의 실세계 액션"에 대한 보안은 아니다.
- **핵심 갭**: 에이전트가 새로운 유형의 도구(예: 이메일 발송, 결제 실행)를 동적으로 사용해야 할 때, 해당 도구의 위험도를 자동 평가하고 적절한 승인 수준을 결정하는 메커니즘이 없다.

### 논의 2: 보안과 자율성의 트레이드오프

| 자율성 높음 | ← 스펙트럼 → | 보안 높음 |
|-------------|-------------|----------|
| TinyClaw (모든 것 자동) | Nanobot, PicoClaw | NanoClaw, OpenClaw | IronClaw, ZeroClaw (모든 것 제어) |

24시간 상주 에이전트에서 이 트레이드오프는 더 극명해진다:
- **높은 자율성** = 빠른 응답, 사용자 개입 불필요 → 비용 폭발, 보안 사고 위험
- **높은 보안** = 안전한 실행 → 지연, 사용자 피로(승인 알림 폭탄)

**ZeroClaw의 해법이 가장 우아하다**: 3단계 AutonomyLevel + E-Stop. 평시에는 `Full`로 자율 실행하되, 문제 발생 시 `Supervised`로 전환하거나 E-Stop으로 즉시 정지. OTP 없이는 재개 불가.

### 논의 3: session_context_report.md의 "빠진 기능이 아니라 안 넣은 기능" 테제 재검증

보안 조사 결과 이 테제는 **부분적으로만 맞다**:

- **맞는 부분**: NanoClaw, PicoClaw 같은 Tier 2-3 프레임워크는 기술적으로 보안 기능을 추가할 수 있는 구조를 이미 갖추고 있다. allowedTools, denylist, Docker 격리 등 기반이 존재.
- **틀린 부분**: IronClaw의 Zero-Exposure 프록시, ZeroClaw의 E-Stop+OTP, IronClaw의 SafetyLayer 같은 기능은 **아키텍처 수준에서 설계된 것**이지 시스템 프롬프트에 지시를 추가해서 해결할 수 있는 것이 아니다.

### 논의 4: Karpathy 실험과의 교차점

Karpathy의 "git worktree로 격리 + 파일 기반 통신 + tmux 대시보드" 접근과 비교하면:

- **git worktree 격리** ≈ NanoClaw의 그룹별 독립 디렉터리 + read-only 프로젝트 마운트
- **파일 기반 통신** ≈ NanoClaw의 IPC (실제로 파일 기반 JSON 교환)
- **tmux takeover** ≈ ZeroClaw의 E-Stop (사람이 언제든 개입 가능)

Karpathy의 접근은 보안 관점에서 보면 **Tier 2 수준**(Docker 없는 NanoClaw)에 해당한다. 파일시스템 격리는 있지만 암호화, HITL, 인젝션 방어는 없다.

### 논의 5: OpenJarvis의 Taint Tracking — 새로운 보안 패러다임

OpenJarvis가 도입한 Taint Tracking은 기존 10개 프레임워크에 없던 **데이터 중심 보안 모델**이다:

- **기존 모델**: "이 도구를 실행해도 되는가?" (도구 중심)
- **Taint Tracking 모델**: "이 데이터를 이 도구에 전달해도 되는가?" (데이터 흐름 중심)

SINK_POLICY 예시:
- `web_search`: PII + SECRET label 데이터 전달 금지 → 개인정보/시크릿이 외부 검색에 노출되는 경로 차단
- `channel_send`: SECRET label 금지 → 메신저 채널로 시크릿 유출 방지
- `code_interpreter`: SECRET label 금지 → 코드 실행 컨텍스트로 시크릿 주입 방지

이 패턴은 IronClaw의 Zero-Exposure(프록시 주입)와 목표는 같지만 구현 방식이 다르다. Zero-Exposure는 "에이전트가 값을 볼 수 없게"이고, Taint Tracking은 "에이전트가 값을 받더라도 특정 경로로 흘려보낼 수 없게"이다.

### 열린 질문 (보안 관점)

1. **동적 도구 위험도 평가**: 새로 등록되는 MCP 도구의 위험도를 자동 분류하고 적절한 승인 수준을 할당하는 메커니즘이 가능한가?
2. **비용 하드 한도의 보편화**: ZeroClaw만 일별 $5 한도를 구현했다. 24시간 에이전트에서 비용 폭발은 현실적 위험인데, 왜 다른 구현체는 이를 무시하는가?
3. **프롬프트 인젝션의 근본적 한계**: IronClaw의 18개 패턴, ZeroClaw의 6개 패턴, OpenJarvis의 regex Scanner는 알려진 공격만 탐지한다. 새로운 인젝션 기법에 대한 적응형 방어가 가능한가?
4. **E-Stop의 메신저 통합**: ZeroClaw의 E-Stop을 Telegram 기반 시스템에서 구현하면 어떤 형태가 되는가? 메시지 하나로 에이전트를 즉시 정지시킬 수 있는가?
5. **Taint Tracking의 완전성**: OpenJarvis의 4-label 분류(PII/SECRET/USER_PRIVATE/EXTERNAL)는 실제 데이터 흐름을 얼마나 커버하는가? LLM이 label을 우회하는 방식으로 데이터를 재포장(paraphrase)하면 방어가 무력화되지 않는가?
