# NVIDIA NemoClaw 심층 분석 보고서 -- Agent OS 비교 연구

> **조사 일자**: 2026-03-17
> **조사 방법**: 소스코드 직접 분석 (bin/, nemoclaw/, nemoclaw-blueprint/ 3개 컴포넌트)
> **대상 레포**: `repos/nemoclaw/` (NVIDIA NemoClaw v0.1.0 Alpha, Apache 2.0)
> **핵심 질문**: "NemoClaw는 OpenClaw Plugin으로서 어떻게 GPU 최적화 샌드박스 에이전트를 구현하는가?"

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. [아키텍처 개요](#2-아키텍처-개요)
3. [Tool/Action 아키텍처](#3-toolaction-아키텍처)
4. [보안 아키텍처 (4-Layer)](#4-보안-아키텍처-4-layer)
5. [메모리/상태 관리](#5-메모리상태-관리)
6. [채널/통신 아키텍처](#6-채널통신-아키텍처)
7. [GPU 최적화 및 추론 시스템](#7-gpu-최적화-및-추론-시스템)
8. [엔터프라이즈 커넥터](#8-엔터프라이즈-커넥터)
9. [Pre-launch 분석 vs 실제 소스코드 비교](#9-pre-launch-분석-vs-실제-소스코드-비교)
10. [종합 평가 및 결론](#10-종합-평가-및-결론)

---

## 1. Executive Summary

NVIDIA NemoClaw v0.1.0 Alpha는 **"Sandboxed Always-On Agent with GPU-Optimized Inference"**를 핵심 명제로 내세운다. 총 25,650 LOC (JavaScript 6,155 / TypeScript 3,041 / Python 3,629 / Shell ~8,000+ / YAML ~525), 3개 주요 컴포넌트(Host CLI · OpenClaw Plugin · Blueprint)로 구성된다.

기존 분석 대상 프레임워크들이 독립 실행형 에이전트 런타임으로 설계된 것과 달리, NemoClaw는 **OpenClaw의 Plugin**으로 작동한다. OpenClaw가 이미 제공하는 대화·메모리·채널 인프라를 재사용하고, NemoClaw는 그 위에 **격리된 샌드박스 컨테이너**와 **GPU 추론 경로**를 추가하는 구조다.

**가장 주목할 발견 5가지:**

1. **샌드박스 우선 설계가 핵심 차별점이다.** Docker 컨테이너(Node 22 + Python 3, non-root sandbox:sandbox user) 기반 4-Layer 보안(Network Policy · Filesystem Policy · Process Policy · Inference Policy)은 기존 비교 프레임워크 중 컨테이너 격리를 정식 레이어로 채택한 유일한 구현이다.

2. **Blueprint 시스템은 에이전트 환경의 IaC(Infrastructure-as-Code)다.** OCI 레지스트리에서 Blueprint를 pull하고, SHA-256 digest 검증 후, Python 오케스트레이터가 plan/apply/status/rollback를 실행한다. Terraform과 유사한 선언적 에이전트 환경 관리가 가능하다.

3. **추론 경로가 OpenShell 게이트웨이로 완전히 중앙화된다.** 샌드박스 내부에서는 모델 API 키에 접근할 수 없다. 자격증명은 OpenShell이 주입하고, 모든 모델 호출은 `integrate.api.nvidia.com/v1`을 통과한다. 기존 프레임워크들이 에이전트 프로세스 내에 API 키를 보관하는 것과 구조적으로 다르다.

4. **GPU 감지와 NIM 컨테이너 관리가 온보딩 단계에서 자동화된다.** nvidia-smi / system_profiler / DGX Spark 감지 후 VRAM 용량에 따라 추론 프로파일(NVIDIA cloud · NCP partner · nim-local · vLLM)이 자동 선택된다. Nemotron 3 Super 120B가 기본 모델이다.

5. **Pre-launch 예측과 실제 구현의 괴리가 크다.** 사전 분석은 독립 배포형 H100/A100 on-premise 솔루션, 50+ 엔터프라이즈 커넥터, HashiCorp Vault 통합을 예측했다. 실제는 OpenClaw Plugin 모델, 10개 네트워크 정책 프리셋, JSON 파일 자격증명 관리로 훨씬 단순하고 실용적이다.

---

## 2. 아키텍처 개요

### 2.1 전체 구조 (ASCII 다이어그램)

```
┌─────────────────────────────────────────────────────────────────┐
│                        사용자 인터페이스                           │
│  ┌──────────────────┐  ┌────────────────┐  ┌──────────────────┐ │
│  │  OpenClaw Chat   │  │  Host CLI      │  │ Telegram Bridge  │ │
│  │ /nemoclaw slash  │  │ nemoclaw.js    │  │ scripts/         │ │
│  │    command       │  │ (dispatcher)   │  │ telegram-bridge  │ │
│  └────────┬─────────┘  └───────┬────────┘  └────────┬─────────┘ │
└───────────┼────────────────────┼────────────────────┼───────────┘
            │                    │                    │
┌───────────▼────────────────────▼────────────────────▼───────────┐
│                     OpenShell Gateway Layer                      │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Policy Approval Flow  │  Credential Injection          │   │
│   │  (Operator TUI)        │  (API Key 주입, 샌드박스 외부)  │   │
│   └─────────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│              NemoClaw OpenClaw Plugin (nemoclaw/)                │
│  ┌─────────────────────┐   ┌─────────────────────────────────┐  │
│  │  openclaw.plugin.json│   │  /nemoclaw slash command        │  │
│  │  (Plugin Manifest)  │   │  status/launch/connect/         │  │
│  │                     │   │  logs/eject                     │  │
│  └─────────────────────┘   └─────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Blueprint Orchestrator (Python)                 │ │
│  │   plan → apply → status → rollback                         │ │
│  │   OCI Registry pull + SHA-256 digest 검증                  │ │
│  │   Cache: ~/.nemoclaw/blueprints/<version>/                 │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                  Docker Sandbox Container                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Runtime: Node 22 + Python 3                                │ │
│  │  User: sandbox:sandbox (non-root)                           │ │
│  │                                                             │ │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  │ │
│  │  │ Network Policy │  │  FS Policy    │  │Process Policy │  │ │
│  │  │ deny-by-default│  │  Landlock MAC │  │seccomp, no    │  │ │
│  │  │ 10 presets     │  │  /sandbox rw  │  │priv escalation│  │ │
│  │  └───────────────┘  └───────────────┘  └───────────────┘  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│  Run ID: nc-YYYYMMDD-HHMMSS-<uuid8>                             │
│  State:  ~/.nemoclaw/state/runs/<run-id>/                       │
└───────────────────────────────┬─────────────────────────────────┘
                                │  Inference Policy
                                │  (OpenShell 경유 강제)
┌───────────────────────────────▼─────────────────────────────────┐
│                   NVIDIA Inference Backend                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  integrate.api.nvidia.com/v1                             │   │
│  │  build.nvidia.com (NVIDIA cloud profile)                 │   │
│  │  NCP Partner / nim-local / vLLM (GPU 환경별)             │   │
│  │  Default Model: Nemotron 3 Super 120B                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 3개 컴포넌트 역할 분리

| 컴포넌트 | 위치 | 언어 | 역할 |
|---------|------|------|------|
| **Host CLI** | `bin/nemoclaw.js` | JavaScript | onboard · deploy · connect · status · logs · policy 커맨드 dispatcher |
| **OpenClaw Plugin** | `nemoclaw/` | TypeScript + Python | Plugin manifest, slash command handler, Blueprint 오케스트레이터 |
| **Blueprint** | `nemoclaw-blueprint/` | Shell + YAML | 샌드박스 환경 정의, Docker image 빌드, 정책 설정 |

### 2.3 기존 프레임워크 대비 포지셔닝

NemoClaw는 **OpenClaw의 확장 모듈**이다. 이것은 아키텍처 선택의 결과이기도 하고 제약이기도 하다.

| 측면 | NemoClaw | 기존 독립형 프레임워크 |
|------|----------|-----------------|
| 채널/메모리 | OpenClaw에 위임 | 자체 구현 |
| 추론 격리 | 샌드박스 컨테이너 완전 격리 | 프로세스 내 실행 |
| GPU 최적화 | 온보딩 자동 감지 + 프로파일 선택 | 없음 |
| 배포 단위 | Blueprint (OCI 이미지) | 바이너리 / npm 패키지 |
| 자격증명 격리 | OpenShell 게이트웨이 외부 관리 | 에이전트 프로세스 내 보관 |

---

## 3. Tool/Action 아키텍처

### 3.1 Plugin API 통합 방식

NemoClaw는 자체 Tool 시스템을 구현하지 않는다. 대신 OpenClaw Plugin API를 통해 3가지 진입점을 등록한다:

```
openclaw.plugin.json (Plugin Manifest)
  ├── slash_commands: ["/nemoclaw"]
  ├── cli_commands: ["nemoclaw"]
  ├── provider_plugin: true   (추론 프로바이더 교체)
  └── service_registration: true  (백그라운드 서비스)
```

### 3.2 /nemoclaw Slash Command 핸들러

OpenClaw 채팅에서 호출 가능한 5개 서브커맨드:

| 서브커맨드 | 기능 |
|-----------|------|
| `/nemoclaw status` | 현재 샌드박스 상태 조회 |
| `/nemoclaw launch` | 새 샌드박스 인스턴스 시작 |
| `/nemoclaw connect` | 실행 중인 샌드박스에 연결 |
| `/nemoclaw logs` | 샌드박스 로그 스트리밍 |
| `/nemoclaw eject` | 샌드박스 종료 및 정리 |

### 3.3 Host CLI 커맨드 구조

`bin/nemoclaw.js` dispatcher가 라우팅하는 6개 커맨드:

| 커맨드 | 기능 |
|--------|------|
| `nemoclaw onboard` | GPU 감지 → 추론 프로파일 선택 → 초기 설정 |
| `nemoclaw deploy` | Blueprint 적용 (apply) → 샌드박스 구동 |
| `nemoclaw connect` | 대화형 TUI (openshell term) 세션 |
| `nemoclaw status` | 샌드박스 런타임 상태 조회 |
| `nemoclaw logs` | 실시간 로그 출력 |
| `nemoclaw policy` | 네트워크/파일시스템 정책 관리 |

### 3.4 Blueprint Orchestrator 액션 시스템

Python 오케스트레이터(`nemoclaw/`)가 구현하는 4개 액션:

```
plan     -> 변경사항 미리 계산 (dry-run, 실제 적용 없음)
apply    -> Blueprint 실제 적용 (OCI pull → digest 검증 → 컨테이너 시작)
status   -> 현재 Blueprint 상태 및 실행 중인 인스턴스 조회
rollback -> 이전 Blueprint 버전으로 복구 (tar 스냅샷 활용)
```

**OCI Registry 다운로드 플로우:**
```
1. Blueprint 버전 지정
2. OCI 레지스트리에서 이미지 메타데이터 fetch
3. SHA-256 digest 검증 (변조 감지)
4. ~/.nemoclaw/blueprints/<version>/ 에 캐시
5. Docker 컨테이너로 배포
```

### 3.5 10개 Policy Connector (Tool 확장)

네트워크 정책 프리셋이 사실상 "허가된 외부 서비스 목록"으로 기능하며, 이것이 NemoClaw의 Tool 확장 메커니즘이다:

| Connector | 허용 egress | 인증 방식 |
|-----------|-----------|---------|
| Slack | api.slack.com | Bearer Token |
| Discord | discord.com/api | Bot Token |
| Telegram | api.telegram.org | Bot Token |
| Jira | *.atlassian.net | API Key |
| Outlook | graph.microsoft.com | OAuth 2.0 |
| HuggingFace | huggingface.co | API Token |
| PyPI | pypi.org, files.pythonhosted.org | 없음 (공개) |
| npm | registry.npmjs.org | 없음 (공개) |
| Docker | registry-1.docker.io | Docker credentials |
| GitHub | api.github.com, github.com | PAT / GitHub App |

---

## 4. 보안 아키텍처 (4-Layer)

### 4.1 4-Layer 보안 모델 개요

NemoClaw의 보안은 **샌드박스 컨테이너를 경계로** 4개 독립 레이어가 중첩된다. 기존 프레임워크들이 프로세스 수준 격리(WASM, seccomp)에 집중한 것과 달리, NemoClaw는 컨테이너 수준 격리를 기본으로 설정하고 그 위에 추가 레이어를 쌓는다.

```
┌─────────────────────────────────────────────┐
│  Layer 4: Inference Policy                  │
│  (모든 모델 호출 → OpenShell 게이트웨이 강제) │
├─────────────────────────────────────────────┤
│  Layer 3: Process Policy                    │
│  (non-root, seccomp, 네트워크 네임스페이스)   │
├─────────────────────────────────────────────┤
│  Layer 2: Filesystem Policy                 │
│  (Landlock MAC, 읽기전용 시스템 경로)         │
├─────────────────────────────────────────────┤
│  Layer 1: Network Policy                    │
│  (deny-by-default egress, 10 프리셋)         │
└─────────────────────────────────────────────┘
```

### 4.2 Layer 1: Network Policy (네트워크 정책)

**핵심 원칙: deny-by-default egress**

- 모든 outbound 트래픽은 기본 차단
- 10개 사전 정의 프리셋 중 선택적 허용
- Binary 수준 규칙: 특정 실행 파일만 특정 도메인 접근 가능
- Method 수준 인증: HTTP 메서드(GET/POST/PUT/DELETE)별 별도 권한
- TLS termination: 모든 허용 트래픽에 TLS 강제
- **Hot-reload**: 컨테이너 재시작 없이 정책 변경 반영

```
# 정책 적용 예시 (개념적)
egress_policy:
  default: deny
  rules:
    - binary: /usr/bin/node
      destination: api.slack.com
      port: 443
      methods: [POST, GET]
      auth: bearer_token
    - binary: python3
      destination: huggingface.co
      port: 443
      methods: [GET]
      auth: api_token
```

**Operator Approval Flow**: 정책에 없는 egress 요청 발생 시 Operator TUI에 승인 요청을 전송하고 응답을 대기한다. 승인되면 임시 예외 규칙이 추가된다.

### 4.3 Layer 2: Filesystem Policy (파일시스템 정책)

**Landlock MAC (Mandatory Access Control)** 기반:

| 경로 | 권한 | 대상 |
|------|------|------|
| `/sandbox` | read-write | 에이전트 작업 디렉토리 |
| `/tmp` | read-write | 임시 파일 |
| `/usr`, `/lib`, `/bin` | read-only | 시스템 바이너리 |
| `/proc` | read-only (제한적) | 프로세스 정보 |
| `~/.nemoclaw` | 접근 불가 | 호스트 자격증명 보호 |
| 기타 호스트 경로 | 접근 불가 | 마운트 없음 |

**Landlock의 의미**: Linux 5.13+에서 제공되는 커널 수준 sandboxing으로, 프로세스 자체가 자신의 파일시스템 접근을 제한할 수 있다. root 권한 없이도 강력한 격리를 제공하며 컨테이너 탈출(escape) 공격면을 줄인다.

### 4.4 Layer 3: Process Policy (프로세스 정책)

4가지 프로세스 수준 제약이 결합된다:

| 제약 | 구현 | 효과 |
|------|------|------|
| **Non-root 실행** | `USER sandbox:sandbox` | 컨테이너 탈출 시 호스트 권한 없음 |
| **Seccomp 필터** | 허용 syscall 화이트리스트 | 커널 취약점 악용 공격면 감소 |
| **No Privilege Escalation** | `no-new-privileges` 플래그 | setuid/setgid 바이너리 실행 불가 |
| **Network Namespace Isolation** | 별도 네트워크 네임스페이스 | 호스트 네트워크 인터페이스 불가시 |

### 4.5 Layer 4: Inference Policy (추론 정책)

**구조적 자격증명 격리**: 모든 모델 호출이 OpenShell 게이트웨이를 통과해야 한다.

```
[샌드박스 내부 에이전트]
    │
    │  추론 요청 (API 키 없음)
    ▼
[OpenShell Gateway]
    │
    │  자격증명 주입 (샌드박스 외부에서 관리)
    ▼
[NVIDIA Cloud: integrate.api.nvidia.com/v1]
```

**기존 프레임워크와의 구조적 차이:**

| 프레임워크 | API 키 위치 | 유출 위험 |
|-----------|-----------|---------|
| NemoClaw | OpenShell 게이트웨이 (샌드박스 외부) | 샌드박스 침해 시에도 API 키 보호 |
| IronClaw | AES-256-GCM 암호화 볼트 (프로세스 내) | 프로세스 메모리 덤프 시 위험 |
| OpenClaw | `~/.config/openclaw/` 파일 | 프로세스 접근 가능 |
| 기타 | 환경 변수 (프로세스 내) | 가장 취약 |

### 4.6 기존 프레임워크 보안 비교

| 보안 기준 | NemoClaw | OpenFang | IronClaw | ZeroClaw |
|---------|---------|---------|---------|---------|
| 컨테이너 격리 | [O] Docker Layer 1~4 | [X] | [X] WASM만 | [X] Docker (단순) |
| API 키 격리 | [O] 게이트웨이 외부 | [X] env var | [O] AES-256-GCM | [O] ChaCha20 |
| Landlock MAC | [O] | [X] | [X] | [X] |
| 네트워크 deny-by-default | [O] | [X] | [~] capability | [X] |
| Operator 승인 플로우 | [O] TUI | [O] LLM 기반 | [O] 도구별 | [O] E-Stop |
| Hot-reload 정책 | [O] | [X] | [X] | [X] |

---

## 5. 메모리/상태 관리

### 5.1 런타임 상태 구조

NemoClaw는 **자체 대화 메모리나 지식 그래프를 구현하지 않는다.** 대화 메모리는 OpenClaw에 완전히 위임하고, NemoClaw가 직접 관리하는 것은 샌드박스 런타임 상태다.

```
~/.nemoclaw/
├── state/
│   └── runs/
│       └── nc-20260317-143022-a3f7b9c1/   <- Run ID 디렉토리
│           ├── sandbox.json               <- 샌드박스 런타임 상태
│           ├── policy.json                <- 적용된 정책 스냅샷
│           └── logs/                      <- 실행 로그
├── blueprints/
│   └── <version>/                         <- Blueprint OCI 캐시
├── credentials.json                       <- 자격증명 저장소 (mode 600)
└── registry.json                          <- 샌드박스 인스턴스 레지스트리
```

### 5.2 Run ID 체계

```
nc-YYYYMMDD-HHMMSS-<uuid8>

예: nc-20260317-143022-a3f7b9c1

nc-       = NemoClaw 식별자 prefix
20260317  = 날짜 (YYYYMMDD)
143022    = 시간 (HHMMSS)
a3f7b9c1  = UUID v4 앞 8자리 (충돌 방지)
```

이 Run ID가 모든 상태 디렉토리, 로그, 레지스트리 엔트리의 기본 키로 사용된다.

### 5.3 자격증명 관리

```json
// ~/.nemoclaw/credentials.json (mode 600)
{
  "nvidia_api_key": "nvapi-...",
  "connectors": {
    "slack": { "token": "xoxb-..." },
    "github": { "pat": "ghp_..." }
  }
}
```

**모드 600 강제**: 파일 시스템 권한으로 소유자만 읽기/쓰기 가능. 그룹·기타 사용자 접근 차단.

**한계**: HashiCorp Vault나 AWS Secrets Manager 같은 외부 시크릿 관리 시스템과 통합되지 않는다. JSON 파일 기반 관리는 단순하지만 비밀 로테이션, 접근 감사, 중앙화된 시크릿 관리가 불가능하다.

### 5.4 마이그레이션 스냅샷

Blueprint rollback을 위한 상태 스냅샷 메커니즘:

- **포맷**: tar 아카이브 (symlink 보존 포함)
- **내용**: 샌드박스 /sandbox 디렉토리 전체 상태
- **트리거**: apply 실행 전 자동 생성
- **복구**: `rollback` 액션이 가장 최근 스냅샷으로 복원

```
스냅샷 파일명 규칙:
snapshot-<run-id>-<timestamp>.tar.gz
```

### 5.5 메모리 아키텍처 비교

| 항목 | NemoClaw | OpenFang | OpenClaw | IronClaw |
|------|---------|---------|---------|---------|
| 대화 메모리 | OpenClaw 위임 | SQLite (Phase 1) | LanceDB+decay | pgvector+RRF |
| Knowledge Graph | 없음 | [O] entity-relation | [X] | [X] |
| 벡터 검색 | 없음 (위임) | 코사인 유사도 | MMR+decay | RRF |
| 런타임 상태 | JSON 파일 | SQLite WAL | SQLite | PostgreSQL |
| 스냅샷/복구 | tar 아카이브 | [X] | [X] | [X] |

---

## 6. 채널/통신 아키텍처

### 6.1 5개 통신 채널

NemoClaw는 5개의 독립적인 통신 경로를 가진다:

```
┌─────────────────────────────────────────────────────────────┐
│                    채널 1: OpenClaw Chat                     │
│  /nemoclaw slash command → Plugin API → 샌드박스             │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│                    채널 2: Host CLI                          │
│  nemoclaw.js → 대화형 TUI (openshell term)                  │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│                    채널 3: Telegram Bridge                   │
│  scripts/telegram-bridge.js → Bot API → 샌드박스            │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│            채널 4: 추론 (단방향, 샌드박스 → NVIDIA)           │
│  OpenClaw → OpenShell Gateway → integrate.api.nvidia.com/v1 │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│                채널 5: Operator 승인 플로우                    │
│  미허가 egress 요청 → Operator TUI → 승인/거부               │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Telegram Bridge 구조

`scripts/telegram-bridge.js`가 Telegram Bot API와 NemoClaw 샌드박스 사이의 브릿지로 동작한다:

- Telegram Bot API polling/webhook 방식 지원
- 메시지를 샌드박스로 전달, 응답을 Telegram으로 반환
- 정책상 허용된 경우 Telegram connector 프리셋 자동 적용

**주목할 점**: 다른 메신저 브릿지(Slack, Discord 등)는 정책 프리셋으로 허용되지만, 별도 브릿지 스크립트는 Telegram만 존재한다. 다른 메신저는 사용자가 직접 구현해야 한다.

### 6.3 채널 비교 (기존 프레임워크 대비)

| 프레임워크 | 채널 수 | 방식 |
|-----------|---------|------|
| OpenFang | 40 어댑터 | 자체 채널 레이어 |
| NemoClaw | 5 (OpenClaw 채널 + 4개 추가) | Plugin 위임 + 보조 채널 |
| OpenClaw | OpenClaw 내장 채널 | 자체 채널 레이어 |
| IronClaw | 0 | 없음 |

---

## 7. GPU 최적화 및 추론 시스템

### 7.1 GPU 감지 플로우

`nemoclaw onboard` 실행 시 3단계 GPU 감지가 순차적으로 시도된다:

```
Step 1: nvidia-smi 실행
  → 성공: NVIDIA GPU 감지
  → VRAM 용량 파싱 (GB)
  → GPU 모델명, 드라이버 버전 확인

Step 2: system_profiler SPDisplaysDataType (Apple 전용)
  → 성공: Apple Silicon / AMD GPU 감지
  → Metal 지원 확인
  → Unified Memory 용량 파싱

Step 3: DGX Spark 감지
  → /etc/dgx-release 파일 존재 확인
  → DGX 시스템 특화 설정 적용

감지 실패:
  → CPU-only 모드 (NVIDIA cloud 프로파일만 사용)
```

### 7.2 4개 추론 프로파일

GPU 감지 결과에 따라 자동 선택되는 추론 프로파일:

| 프로파일 | 엔드포인트 | 선택 조건 | 특성 |
|---------|-----------|---------|------|
| **NVIDIA cloud** | build.nvidia.com | GPU 없음 또는 VRAM 부족 | 관리형 클라우드, API 과금 |
| **NCP partner** | NCP 파트너 엔드포인트 | 파트너 계약 보유 | 중간 수준 격리, 전용 리소스 |
| **nim-local** | localhost NIM 컨테이너 | NVIDIA GPU 충분한 VRAM | 완전 로컬, 지연 최소 |
| **vLLM** | localhost vLLM 서버 | NVIDIA/AMD GPU (vLLM 지원) | 오픈소스, 다중 모델 지원 |

### 7.3 NIM 컨테이너 관리

`nim-local` 프로파일 선택 시 NIM(NVIDIA Inference Microservice) 컨테이너가 자동으로 관리된다:

```
nemoclaw onboard (nim-local 선택 시)
  → Docker Hub에서 NIM 이미지 pull
  → GPU 할당 설정 (--gpus 플래그)
  → 포트 바인딩 (localhost:8000)
  → 헬스체크 대기
  → Blueprint에 nim-local 엔드포인트 등록
```

**VRAM별 자동 선택 기준** (GPU 감지 로직에서 파싱):

| VRAM | 추천 프로파일 | Nemotron 3 Super 120B |
|------|------------|----------------------|
| < 24GB | NVIDIA cloud | 클라우드 추론 (지원 불가) |
| 24~80GB | nim-local (단일 GPU) | FP8 양자화 필요 |
| 80GB+ | nim-local | FP16 전체 정밀도 |
| 멀티 GPU | nim-local (분산) | Tensor Parallelism |

### 7.4 기본 모델: Nemotron 3 Super 120B

```
모델: Nemotron 3 Super 120B
API: integrate.api.nvidia.com/v1
설명: NVIDIA 최신 대형 언어 모델 (120B 파라미터)
특성: 추론 집약적 태스크 최적화, 도구 사용 지원
```

**단일 모델 집중**: 기존 pre-launch 예측이 Multi-Model Routing을 예상했던 것과 달리, v0.1.0에서는 Nemotron 3 Super 120B 단일 모델에 집중한다. 모델 선택권은 프로파일 선택을 통해 간접적으로만 제공된다 (vLLM 프로파일 사용 시 다른 모델 지정 가능).

### 7.5 GPU 최적화 관련 기존 프레임워크 비교

| 프레임워크 | GPU 지원 | 로컬 추론 | 자동 GPU 감지 |
|-----------|---------|---------|------------|
| NemoClaw | [O] 4 프로파일 | [O] nim-local / vLLM | [O] 3단계 감지 |
| OpenFang | [X] | [X] | [X] |
| IronClaw | [X] | [X] | [X] |
| OpenClaw | [X] | [X] | [X] |
| 기타 모두 | [X] | [X] | [X] |

**NemoClaw는 비교 대상 프레임워크 중 유일하게 GPU 인식 온보딩과 로컬 추론을 구현한다.**

---

## 8. 엔터프라이즈 커넥터

### 8.1 10개 네트워크 정책 프리셋 전체 목록

NemoClaw의 "엔터프라이즈 커넥터"는 **네트워크 정책 프리셋**으로 구현된다. 50+ 커넥터를 예측했던 pre-launch 분석과 달리, v0.1.0은 10개에 집중한다.

| 커넥터 | 허용 도메인 | 인증 타입 | 용도 |
|--------|-----------|---------|------|
| **Slack** | api.slack.com, slack.com | Bearer Token | 팀 커뮤니케이션 |
| **Discord** | discord.com/api, cdn.discordapp.com | Bot Token | 커뮤니티/팀 |
| **Telegram** | api.telegram.org | Bot Token | 메시징 (브릿지 스크립트 포함) |
| **Jira** | *.atlassian.net | API Key + Basic Auth | 이슈 트래킹 |
| **Outlook** | graph.microsoft.com, login.microsoftonline.com | OAuth 2.0 | 이메일/캘린더 |
| **HuggingFace** | huggingface.co, cdn-lfs.huggingface.co | API Token | 모델/데이터셋 |
| **PyPI** | pypi.org, files.pythonhosted.org | 없음 (공개) | Python 패키지 |
| **npm** | registry.npmjs.org, npmjs.com | 없음/npm token | Node 패키지 |
| **Docker** | registry-1.docker.io, auth.docker.io | Docker credentials | 컨테이너 이미지 |
| **GitHub** | api.github.com, github.com, raw.githubusercontent.com | PAT / GitHub App | 코드 저장소 |

### 8.2 Hot-reload 메커니즘

정책 프리셋의 가장 중요한 운영 특성:

```
nemoclaw policy apply slack  (또는 /nemoclaw status 채팅에서)
  → 정책 파일 변경
  → 컨테이너 재시작 없이 네트워크 규칙 즉시 적용
  → 기존 연결 유지
  → 새 연결부터 새 정책 적용
```

**기존 프레임워크 중 Hot-reload 네트워크 정책을 구현한 곳은 NemoClaw뿐이다.**

### 8.3 커넥터 병합 방식

여러 커넥터를 동시에 활성화할 수 있으며, 정책이 merge된다:

```
기본: deny-all egress
  + slack  프리셋 → api.slack.com 허용
  + github 프리셋 → api.github.com 허용
  = slack + github 동시 허용 (나머지는 deny 유지)
```

머지 충돌(동일 도메인 서로 다른 정책) 발생 시 더 제한적인 규칙이 우선 적용된다.

### 8.4 자격증명 관리

커넥터 자격증명은 환경 변수를 통해 주입된다:

```bash
# nemoclaw deploy 시 환경 변수로 전달
SLACK_BOT_TOKEN=xoxb-... nemoclaw deploy
GITHUB_TOKEN=ghp_...    nemoclaw deploy
```

샌드박스 내부에서는 이 환경 변수에 접근할 수 있지만, `~/.nemoclaw/credentials.json`에는 접근 불가능하다. 이 분리가 Layer 2 (Filesystem Policy)에 의해 강제된다.

### 8.5 Pre-launch 예측 대비 커넥터 격차

| 예측 카테고리 | 예측 수 | 실제 v0.1.0 | 차이 |
|-------------|---------|-----------|------|
| 전체 커넥터 | 50+ | 10 | -40+ |
| 데이터베이스 커넥터 | 20+ | 0 | -20+ |
| 엔터프라이즈 SaaS (Salesforce, SAP 등) | 10+ | 0 | -10+ |
| 메시징 | 다수 | 3 (Slack/Discord/Telegram) | 범위 축소 |
| 패키지 저장소 | 미언급 | 3 (PyPI/npm/Docker) | 신규 |

---

## 9. Pre-launch 분석 vs 실제 소스코드 비교

### 9.1 전체 비교표

| 예측 항목 | Pre-launch 예측 | 실제 소스코드 | 일치 여부 |
|---------|---------------|------------|---------|
| **배포 모델** | 독립형 on-premise (H100/A100) | OpenClaw Plugin 모델 | 불일치 |
| **GPU 지원** | H100, A100, GB200, L40S, RTX Ada | NVIDIA GPU 일반 + Apple + DGX Spark | 부분 일치 |
| **엔터프라이즈 커넥터** | 50+ (DB, SaaS, 스토리지 포함) | 10개 네트워크 정책 프리셋 | 불일치 |
| **자격증명 관리** | HashiCorp Vault / AWS Secrets Manager | credentials.json (mode 600) | 불일치 |
| **보안 모델** | Zero-trust + Enterprise SSO + SAML | 4-Layer 컨테이너 샌드박스 | 부분 일치 |
| **멀티 모델 추론** | 여러 LLM 라우팅 지원 | Nemotron 3 Super 120B 단일 | 불일치 |
| **파인튜닝** | on-premise 파인튜닝 지원 | 없음 | 불일치 |
| **연합 학습** | Federated Learning 로드맵 | 없음 | 불일치 |
| **채널/메신저** | REST API + WebSocket + gRPC 독립 제공 | OpenClaw 채널 위임 + Telegram 브릿지 | 불일치 |
| **샌드박스 격리** | 예측 없음 | Docker 4-Layer 핵심 기능 | 예측 초과 |
| **Blueprint IaC** | 예측 없음 | OCI Registry + plan/apply/rollback | 예측 초과 |
| **Hot-reload 정책** | 예측 없음 | 네트워크 정책 Hot-reload 구현 | 예측 초과 |
| **GPU 자동 감지** | 예측 없음 | 3단계 nvidia-smi/system_profiler/DGX | 예측 초과 |
| **Operator 승인 TUI** | "Operator approval 플로우" 언급 | TUI 기반 실제 구현 | 일치 |
| **24/7 자율 운영** | 고객 인프라 의존 | 샌드박스 Always-On 설계 | 부분 일치 |

### 9.2 예측이 크게 빗나간 이유 분석

**가장 큰 오인**: Pre-launch 분석은 NemoClaw를 독립 실행형 엔터프라이즈 플랫폼으로 예측했다. 실제 v0.1.0은 OpenClaw의 Plugin으로, 훨씬 집중된 범위를 가진다.

| 오인 원인 | 설명 |
|---------|------|
| **NVIDIA 브랜드 이미지** | NVIDIA = 엔터프라이즈 H100/A100이라는 선입견 |
| **NeMo 프레임워크 연상** | NVIDIA NeMo의 엔터프라이즈 기능을 NemoClaw에 투영 |
| **Guardrails 오해** | NVIDIA NeMo Guardrails (별개 프로젝트)를 NemoClaw 기능으로 혼동 |
| **버전 단계 무시** | v0.1.0 Alpha의 의미를 충분히 고려하지 않음 |

### 9.3 예측이 정확했던 영역

| 예측 | 실제 | 정확도 |
|------|------|--------|
| GPU 최적화가 핵심 차별점 | GPU 감지 + 4 추론 프로파일 | [O] 핵심 방향 정확 |
| 엔터프라이즈 보안 중시 | 4-Layer 샌드박스 보안 | [O] 방향 정확, 구현은 다름 |
| 격리된 컨테이너 실행 | Docker 컨테이너 기반 | [O] 일치 |
| Operator 승인 플로우 | TUI 승인 인터페이스 | [O] 일치 |
| 자격증명 별도 관리 | OpenShell 자격증명 주입 | [O] 방향 정확 |

---

## 10. 종합 평가 및 결론

### 10.1 핵심 강점

**1. 샌드박스 우선 보안이 구조적으로 우수하다**

4-Layer 보안 모델, 특히 Inference Policy의 OpenShell 게이트웨이 패턴은 API 키를 에이전트 프로세스에서 완전히 분리한다. 이 설계는 다른 프레임워크들이 자격증명 볼트(IronClaw의 AES-256-GCM)나 환경 변수로 해결하는 것을 아키텍처 수준에서 처리한다.

**2. Blueprint IaC가 재현 가능한 에이전트 환경을 가능하게 한다**

OCI 레지스트리 기반 Blueprint는 에이전트 환경을 버전 관리하고 plan/apply/rollback이 가능한 선언적 객체로 만든다. 이는 기존 프레임워크들이 에이전트 설정을 TOML/JSON 파일이나 코드로 관리하는 것과 근본적으로 다른 접근이다.

**3. GPU 온보딩 자동화가 진입 장벽을 낮춘다**

nvidia-smi / system_profiler / DGX Spark 3단계 감지와 VRAM 기반 자동 프로파일 선택은 GPU 환경 설정의 복잡성을 숨긴다. 사용자는 GPU 종류를 신경 쓰지 않고 `nemoclaw onboard` 하나로 최적 추론 환경을 얻는다.

### 10.2 핵심 약점

**1. OpenClaw 의존성이 독립 사용을 불가능하게 한다**

OpenClaw 없이 NemoClaw는 채널도 메모리도 없다. 이 의존성은 OpenClaw 사용자에게는 장점(즉시 사용 가능한 인프라)이지만, 독립적인 에이전트 플랫폼으로서는 불완전하다.

**2. 자격증명 관리가 프로덕션 수준에 미치지 못한다**

credentials.json (mode 600)은 개발 단계에 적합하지만 엔터프라이즈 환경에서는 비밀 로테이션, 중앙화된 감사, 팀 단위 접근 제어가 불가능하다. Pre-launch에서 예측했던 Vault 통합이 v0.1.0에는 없다.

**3. 10개 커넥터는 엔터프라이즈 요구에 부족하다**

Salesforce, SAP, Workday, Snowflake 같은 핵심 엔터프라이즈 시스템 커넥터가 없다. 데이터베이스 커넥터도 전무하다. "엔터프라이즈" 포지셔닝과 실제 커넥터 생태계 사이의 간격이 크다.

**4. 대화 메모리를 직접 제어하지 않는다**

OpenClaw의 메모리 시스템에 완전히 위임하므로, NemoClaw 자체의 컨텍스트 관리 전략이 없다. OpenClaw 메모리 시스템의 제약이 NemoClaw 에이전트에도 그대로 적용된다.

### 10.3 포지셔닝 평가

```
NemoClaw v0.1.0의 실제 포지션:
  "GPU 최적화 샌드박스 에이전트를 위한 OpenClaw Plugin"

Pre-launch 예측 포지션:
  "엔터프라이즈 on-premise AI 에이전트 플랫폼"

차이:
  범위: 플랫폼 → 플러그인 (대폭 축소)
  대상: 독립 엔터프라이즈 고객 → OpenClaw 사용자
  강점: GPU 추론 최적화 + 샌드박스 보안 (예측보다 집중적)
  약점: 엔터프라이즈 커넥터/통합 (예측보다 빈약)
```

### 10.4 비교 프레임워크 내 위치

| 평가 기준 | 순위 | 설명 |
|---------|------|------|
| **보안 아키텍처** | 상위권 | 컨테이너 격리 + API 키 분리 조합은 독보적 |
| **GPU/추론 최적화** | 1위 | 유일한 GPU 인식 온보딩 및 로컬 추론 |
| **메모리 아키텍처** | 하위권 | 자체 구현 없이 OpenClaw 위임 |
| **엔터프라이즈 커넥터** | 하위권 | 10개 프리셋, DB/SaaS 커넥터 없음 |
| **채널 다양성** | 하위권 | OpenClaw 채널 + Telegram 브릿지만 |
| **독립 운영 가능성** | 최하위 | OpenClaw 필수 의존 |
| **배포/환경 관리** | 상위권 | Blueprint IaC, plan/apply/rollback |

### 10.5 신규 오픈 퀘스천

**Q1. OpenShell 게이트웨이의 자격증명 주입은 실제로 어떻게 구현되는가?**
API 키가 샌드박스 외부에서 관리된다는 것은 확인됐지만, OpenShell이 요청별로 자격증명을 어떻게 삽입하는지(헤더 주입? 프록시?) 상세 구현이 공개되지 않았다. 이 부분이 전체 보안 모델의 가장 중요한 미검증 부분이다.

**Q2. Blueprint OCI 레지스트리는 어디에 호스팅되는가?**
digest 검증이 구현됐지만, OCI 레지스트리 자체가 NVIDIA가 운영하는 것인지, 사용자가 private 레지스트리를 지정할 수 있는지 명확하지 않다. air-gap 환경 배포 가능 여부가 여기에 달려 있다.

**Q3. Hot-reload 네트워크 정책은 기존 TCP 연결을 어떻게 처리하는가?**
정책 변경 시 이미 established된 연결이 강제 종료되는지, 자연 종료를 기다리는지, 또는 이전 정책으로 계속 허용되는지 문서화되어 있지 않다. 프로덕션 환경에서 정책 변경의 안전성과 직결된 문제다.

**Q4. v0.1.0 Alpha에서 v1.0으로 가는 로드맵에 Vault 통합이 포함되는가?**
pre-launch 예측의 핵심이었던 HashiCorp Vault 통합이 v0.1.0에 없다. 이것이 의도적 범위 축소인지, 향후 버전에 포함될 예정인지, 아니면 OpenClaw의 자격증명 관리에 완전히 위임하는 전략적 결정인지 명확하지 않다.

**Q5. nim-local 프로파일에서 Nemotron 3 Super 120B 이외의 모델을 사용할 수 있는가?**
vLLM 프로파일이 다른 모델을 지원할 가능성이 있지만, 공식 문서화되어 있지 않다. NemoClaw가 Nemotron 계열에 종속되는지, 아니면 범용 로컬 추론 레이어로 발전할 수 있는지가 장기 포지셔닝에 중요하다.

---

*분석 완료 일자: 2026-03-17*
*대상 버전: NVIDIA NemoClaw v0.1.0 Alpha (Apache 2.0)*
*참조 컴포넌트: bin/ (Host CLI) · nemoclaw/ (OpenClaw Plugin) · nemoclaw-blueprint/ (Blueprint)*
*총 분석 LOC: 25,650 (JavaScript 6,155 / TypeScript 3,041 / Python 3,629 / Shell ~8,000+ / YAML ~525)*
