# 실세계 권한 부여 보안 전략 — 10개 Claw 코드 기반 비교 분석

> **조사 일자**: 2026-03-05 (OpenJarvis 추가: 2026-03-14, OpenFang/NemoClaw 추가: 2026-03-17, Claude Code 추가: 2026-03-20)
> **조사 방법**: 7개 scientist 에이전트가 각 레포의 보안/권한 관련 소스코드를 병렬 심층 분석 (OpenJarvis, OpenFang, NemoClaw는 별도 추가 분석)
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

16개 구현체의 보안/권한 코드를 분석한 결과, **4개의 보안 성숙도 계층**으로 분류된다:

| 계층 | 구현체 | 특징 |
|------|--------|------|
| **Tier S: Next-Gen Defense** | **OpenFang** | WASM Dual Metering + 18종 Capability + Taint Tracking + Keyring + HITL 승인 흐름 — 기존 Tier 1을 능가하는 다층 방어 |
| **Tier 1: Defense-in-Depth** | IronClaw, ZeroClaw | 암호화 볼트 + WASM/Docker 이중 샌드박스 + 다층 인젝션 방어 + HITL 승인 + 비용 제한 |
| **Tier A+: Sandbox-First** | **NemoClaw**, **Claude Code**, **GoClaw** | OS 레벨 격리 + HITL 또는 소프트 보안 병행. NemoClaw: Docker+Landlock+seccomp+namespace+추론 게이트웨이 격리. **Claude Code**: seccomp BPF+bwrap(Linux)/native sandbox(macOS), npm vendor에 BPF 동봉, 5겹 채널 프롬프트 인젝션 방어, Platform-Controlled Allowlist. **GoClaw**: Docker 3축(mode/access/scope) 샌드박스 + AES 암호화 + OS Keyring + Tailscale VPN + OAuth/RBAC. 암호화 볼트 있음(AES), WASM 없음 |
| **Tier 2: Container-First** | NanoClaw, OpenClaw, **OpenJarvis**, **Hermes Agent** | Docker/subprocess 격리 + 도구 허용목록 + 자격증명 격리 (암호화 없음) + 부분적 인젝션 방어. OpenJarvis는 Prompt Injection Scanner 명시적 구현, Hermes Agent는 Tirith 외부 바이너리 pre-exec 스캐너 + Memory Injection 탐지 + Skills Trust 4단계 |
| **Tier 3: Denylist-Based** | Nanobot, PicoClaw, **CoPaw** | 정규식/규칙 기반 명령어 차단 + 파일시스템 제한 + 평문 자격증명 + HITL 없음. CoPaw: tool_guard(YAML 규칙) + file_access_guard + skill_security_scanning(정적 분석) |
| **Tier 4: Minimal/None** | TinyClaw | 보안 메커니즘 최소 또는 해당 없음 (실험적 용도) |

**가장 주목할 발견 6가지:**

1. **암호화 볼트를 구현한 곳은 IronClaw, ZeroClaw, OpenFang 3개뿐이다.** 나머지 10개는 전부 평문 저장. IronClaw는 AES-256-GCM + OS Keychain, ZeroClaw는 ChaCha20-Poly1305, OpenFang은 OS Keyring 통합 + 암호화 볼트 + 도구별 자격증명 스코핑.
2. **Human-in-the-loop를 구현한 곳은 4개뿐이다.** IronClaw(도구별 승인), ZeroClaw(3단계 자율성 + E-Stop), OpenClaw(실행 승인 요청), **OpenFang(Capability별 설정 가능 승인 흐름)**, **NemoClaw(미지 egress 요청 운영자 승인)**. 나머지 8개는 에이전트가 도구를 자율 실행.
3. **프롬프트 인젝션 방어에 전용 레이어를 둔 곳은 IronClaw, ZeroClaw, OpenJarvis, OpenFang 4개뿐이다.** IronClaw는 SafetyLayer 4중 방어, ZeroClaw는 PromptGuard 6패턴 탐지, **OpenJarvis는 regex 기반 Prompt Injection Scanner** (4개 위협 수준: LOW/MEDIUM/HIGH/CRITICAL), **OpenFang은 다층 방어 + Taint Tracking으로 데이터 흐름 추적**. NemoClaw는 추론 게이트웨이 격리로 직접 인터넷 접근 차단.
4. **Taint Tracking을 구현한 곳은 OpenJarvis와 OpenFang 2개뿐이다.** OpenJarvis는 4-label 분류(PII/SECRET/USER_PRIVATE/EXTERNAL)와 SINK_POLICY, OpenFang은 Taint Tracking이 WASM Capability 시스템과 결합되어 데이터 흐름 제어.
5. **WASM 샌드박스를 도구 실행에 사용하는 곳은 IronClaw, ZeroClaw, OpenFang 3개뿐이다.** OpenFang의 WASM Dual Metering(CPU 사이클 + 메모리 할당 이중 계량)은 기존 구현 대비 가장 세밀한 자원 제어.
6. **NemoClaw는 "샌드박스 플러그인" 특화 설계로 Tier A+를 달성한다.** Docker + Landlock MAC + seccomp + 네트워크 네임스페이스의 4중 OS 격리. 자체 메모리/기억 시스템 없이 보안에 집중하는 단일 책임 원칙.

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
| **OpenFang** | 18종 Capability 타입 + Taint Tracking + WASM Dual Metering | O (Capability-gated, 도구별 권한 선언) | `crates/capability/src/lib.rs` |
| **NemoClaw** | 4-layer 정책 (network/filesystem/process/inference) + binary-scoped 네트워크 규칙 | O (샌드박스 내 모든 도구 실행) | `nemoclaw/policy/`, `nemoclaw/sandbox/` |

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
| **OpenFang** | Keyring 통합 + 암호화 볼트 | **O** | **O** (OS Keyring) | 도구별 자격증명 스코핑 (per-tool credential scoping) | `crates/secrets/src/keyring.rs`, `crates/secrets/src/vault.rs` |
| **NemoClaw** | `~/.nemoclaw/credentials.json` (mode 600) | X | X | OpenShell 주입 방식 (샌드박스 내부에 값 노출 없음) | `nemoclaw/credentials.py` |

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
| **OpenFang** | X (프로세스 격리) | **O** (WASM Dual Metering — CPU cycles + memory allocation) | WASM 샌드박스 내 도구 실행 | Capability-gated 파일시스템 접근 | `crates/wasm/src/metering.rs` |
| **NemoClaw** | **O** (Docker container + Landlock MAC + seccomp + 네트워크 네임스페이스) | X | OS 4중 격리 — 분석 대상 중 최강 | 컨테이너 내부로만 파일시스템 접근 제한 | `nemoclaw/sandbox/container.py`, `nemoclaw/sandbox/landlock.py` |

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
| **OpenFang** | WASM 샌드박스 내 실행 + Capability-gated (18종) + Dual Metering (CPU + 메모리) | Capability 기반 네트워크 정책 | WASM 경로탐색 차단 | WASM 연료 기반 자원 제한 | `crates/capability/src/lib.rs`, `crates/wasm/src/metering.rs` |
| **NemoClaw** | 컨테이너 내부 전체 샌드박스 실행 | 네트워크 네임스페이스 격리 + 미지 egress 운영자 승인 | 컨테이너 파일시스템 격리 | Docker 자원 제한 | `nemoclaw/sandbox/network.py`, `nemoclaw/policy/egress.py` |

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
| **OpenFang** | **O** (Capability별 설정 가능 승인 흐름) | Capability 유형별 구성 가능 | — | `crates/capability/src/approval.rs` |
| **NemoClaw** | **O** (미지 egress 요청 운영자 승인 흐름) | 알 수 없는 외부 네트워크 요청 | — | `nemoclaw/policy/egress.py` |

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
| **OpenFang** | WASM Dual Metering — CPU 사이클 + 메모리 할당 이중 계량 | X (Metering으로 자원 제한) | WASM 연료 기반 실행 상한 | `crates/wasm/src/metering.rs` |
| **NemoClaw** | X | X | 단일 추론 제공자 라우팅 (비용 제어 간접) | `nemoclaw/inference/gateway.py` |

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
| **OpenJarvis** | **O** (Prompt Injection Scanner) | regex 기반 스캔 + 4개 위협 수준 분류 (LOW/MEDIUM/HIGH/CRITICAL) | `prompt_injection_scanner.py` |
| **OpenFang** | **O** (다층 방어 + Taint Tracking) | 다층 인젝션 방어 + 데이터 흐름 전체의 Taint Tracking — WASM 격리로 인젝션 영향 범위 물리적 제한 | `crates/safety/src/lib.rs`, `crates/taint/src/lib.rs` |
| **NemoClaw** | **O** (추론 게이트웨이 격리) | 샌드박스에서 직접 인터넷 접근 차단 → 인젝션 소스 자체를 격리. 외부 데이터는 추론 게이트웨이를 통해서만 유입 | `nemoclaw/inference/gateway.py` |

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

### 3.9 OpenFang — "WASM Capability 기반 차세대 보안"

**보안 철학**: WASM 샌드박스 + 18종 Capability 시스템 + Taint Tracking의 결합. 기존 Tier 1(IronClaw, ZeroClaw)의 WASM 격리에 더해, 데이터 흐름 추적(Taint)과 세밀한 Capability 권한 부여를 통합한 Tier S 아키텍처.

- **강점**:
  - **18종 Capability 타입**: 도구별 세밀한 권한 선언. 파일 읽기/쓰기/네트워크/프로세스 등 각 Capability를 독립적으로 부여하거나 거부
  - **Taint Tracking**: 데이터 흐름 전체 추적. WASM 격리와 결합되어 "도구가 데이터를 볼 수 있어도 특정 경로로 흘릴 수 없음"
  - **WASM Dual Metering**: CPU 사이클 + 메모리 할당을 동시에 계량. 기존 연료(fuel) 단일 계량 대비 자원 남용 탐지 정밀도 향상
  - **OS Keyring 통합 + 암호화 볼트 + per-tool 자격증명 스코핑**: 자격증명 보안의 3중 방어
  - **HITL Capability 승인 흐름**: Capability 유형별로 구성 가능한 승인 절차
  - **A2A 프로토콜**: 에이전트 간 통신에도 Capability 기반 보안 적용
- **약점**:
  - Docker 컨테이너 격리 없음 (WASM만)
  - 일별 비용 하드 한도 없음

### 3.10 NemoClaw — "OS 4중 격리 특화 샌드박스 플러그인"

**보안 철학**: 단일 책임 원칙 — 보안/격리에만 집중. 자체 메모리/기억/브라우저 시스템 없이, Docker + Landlock MAC + seccomp + 네트워크 네임스페이스의 4중 OS 격리로 OpenClaw 호스트를 보호하는 플러그인 형태.

- **강점**:
  - **Docker + Landlock MAC + seccomp + 네트워크 네임스페이스**: 4중 OS 격리 — 분석 대상 13개 중 OS 레벨 격리 최강
  - **HITL egress 승인**: 알 수 없는 외부 네트워크 요청에 대해 운영자 승인 흐름 트리거
  - **OpenShell 자격증명 주입**: 자격증명 값이 샌드박스 내부에 직접 노출되지 않음
  - **추론 게이트웨이 격리**: 샌드박스에서 직접 인터넷 접근 불가 → 프롬프트 인젝션 소스 격리
  - **10개 네트워크 정책 프리셋 (커넥터)**: binary-scoped 네트워크 규칙으로 egress 세밀 제어
  - **마이그레이션 스냅샷**: 런 상태 tar 아카이브로 보안 환경 이식 가능
- **약점**:
  - WASM 격리 없음 (Docker에 의존)
  - 자격증명 암호화 없음 (credentials.json mode 600만)
  - 자체 HITL 도구 승인 없음 (egress 승인만)
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
    NemoClaw ─────── Docker + Landlock MAC + seccomp + 네트워크 네임스페이스 (4중 OS 격리)
    OpenFang ─────── WASM Dual Metering + Capability-gated (CPU+메모리 이중 계량)
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
| **Capability별 설정 가능 승인** | OpenFang | 18종 Capability 유형별 승인 흐름 구성 |
| **미지 egress 승인** | NemoClaw | 알 수 없는 외부 네트워크 요청에만 운영자 승인 |
| **실행 승인 요청** | OpenClaw | 채널(Discord 등)로 승인 요청 발송 |
| **진입점 제어만** | TinyClaw (Pairing) | 접근 자체만 제한, 내부 실행은 자율 |
| **없음** | Nanobot, NanoClaw, PicoClaw, OpenJarvis | 완전 자율 실행 |

**가장 정교한 HITL**: ZeroClaw의 E-Stop 시스템. `KillAll` / `NetworkKill` / `DomainBlock` / `ToolFreeze` 4단계 긴급 정지 + OTP 없이 해제 불가.

### 패턴 5: "프롬프트 인젝션 방어" 계층 구조

```
Layer 6: 소스 격리 + Taint Tracking + WASM 물리적 제한 (OpenFang — 인젝션 소스 자체를 WASM 격리로 봉쇄 + 데이터 흐름 추적)
Layer 5: 소스 격리 (NemoClaw — 추론 게이트웨이로 직접 인터넷 접근 차단, 인젝션 경로 원천 봉쇄)
Layer 4: 전용 탐지 + 차단 + 위협 수준 분류 (OpenJarvis Prompt Injection Scanner: LOW/MEDIUM/HIGH/CRITICAL)
Layer 3: 전용 탐지 + 차단 (IronClaw SafetyLayer, ZeroClaw PromptGuard)
Layer 2: 경계 마커 + 패턴 탐지 (OpenClaw external-content)
Layer 1: 레이블링 + 이스케이프 (Nanobot 컨텍스트 태그, NanoClaw 스케줄 태그)
Layer 0: 구조적 격리만 (PicoClaw denylist 간접 방어)
Layer -1: 방어 없음 (TinyClaw — 인젝션 벡터 존재)
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

---

## 실전 보안 패턴 추가 (2026-03 meetup 추가)

### 3-Layer Approval 패턴

**소스**: 최재훈 발표 (03_최재훈_Remote_OpenClaw_다중디바이스제어), 정세민 발표 (16_정세민_Ultraworker_AI_Orchestration)

#### OpenClaw 다중 디바이스 승인 체계 (최재훈)

에이전트가 다중 디바이스를 제어할 때의 계층적 승인:

```
[Layer 1: 디바이스 승인]
  어떤 기기에 명령을 내릴 것인가?
  예) macOS 기기 vs Linux 라즈베리파이 vs Windows PC

[Layer 2: 노드 승인]
  해당 기기의 어떤 노드(기능 서버)를 통해 실행할 것인가?
  예) 카메라 노드 vs 브라우저 노드 vs 시스템 노드

[Layer 3: Capability 승인]
  해당 노드의 어떤 기능을 실행할 것인가?
  예) 스크린샷 vs URL 변경 vs 파일 접근
```

**중요**: `system_run`/`exec` 등 정의되지 않은 기능을 허용하려면 리모트 게이트웨이에서 allow list를 수동으로 열어줘야 함 → 기본값은 최소 권한.

#### Ultraworker HITL 승인 체계 (정세민)

Slack BlockKit UI 기반 3단계 사람 검토:

```
[1단계 승인] 작업 리스트 승인
  → Slack BlockKit "좋아요 버튼"으로 승인/거부

[2단계 승인] 테크 스펙 승인
  → 복잡한 업무에서만 적용

[3단계 승인] 구현 완료 보고
  → 결과 검수 후 승인
```

**에이전트 강제 종료**: 잘못된 명령 시 Slack 대시보드에서 terminate 가능.

---

### .env 자동 차단 패턴

**소스**: 김동규 발표 (12_김동규_nanoclaw_사용기)

nanoclaw는 `.env` 등 민감 파일에 대한 에이전트 접근을 **자동 차단 패턴**으로 원천 차단:

```
에이전트가 접근 시도 시:
  .env → 자동 차단
  .env.local → 자동 차단
  secrets.json → 자동 차단
  (설정 가능한 deny list)
```

**기존 프레임워크와 비교**:

| 프레임워크 | 민감 파일 보호 방식 |
|----------|----------------|
| IronClaw | WASM 능력 attenuation (proxy injection) |
| ZeroClaw | autosave 키 블랙리스트 |
| nanoclaw | 자동 차단 패턴 (deny list) |
| OpenClaw | untrusted-data 면책 선언 |

---

### 로컬 모델 민감 데이터 처리 패턴

**소스**: 윤주운 발표 (09_윤주운_Arbiter_FLOCK_워크플로우)

**원칙**: 보안상 외부 클라우드에 올리기 어려운 개인·기업 데이터는 로컬 모델에서만 처리.

```
[클라우드 모델] 사용 가능 데이터
  - 코드 생성 (비즈니스 로직 제외)
  - 일반 오케스트레이션
  - 공개 데이터 분석

[로컬 모델] 필수 데이터
  - 민감한 개인 정보
  - 기업 기밀 데이터
  - 장기 메모리 (대화 이력)
  - 내부 slack/이메일 내용
```

**실전 구성 (윤주운 FLOCK)**:
- 코딩·오케스트레이션: 고성능 클라우드 모델
- 민감 데이터·장기 메모리: RTX 3090 기반 로컬 QwQ 3.2 7B (8비트 양자화, ~50 토큰/초)
- 모든 대화 기록 NAS에 저장 → 향후 로컬 에이전트 파인튜닝 데이터로 활용
