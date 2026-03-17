# 중국 AI 에이전트 서비스 보안 특성 분석
# 6개 서비스 비교: Kimi Claw, Z.ai OpenClaw, Alibaba OpenClaw, Baidu DuClaw, Zhipu AutoClaw, OpenClawD

> **분석 일자**: 2026-03-14
> **분석 대상**: 6개 중국 AI 에이전트 서비스 (공개 문서 기반)
> **평가 방법**: 기존 10개 Claw 프레임워크 보안 매트릭스를 기준으로 5개 영역 평가
> **핵심 질문**: "중국 AI 에이전트 서비스들이 자격증명 관리, 샌드박싱, 권한 제어, 인젝션 방어, HITL에서 어떤 수준의 보안을 제공하는가?"

---

## 목차

1. [Executive Summary](#executive-summary)
2. [5대 보안 영역 비교 매트릭스](#5대-보안-영역-비교-매트릭스)
3. [서비스별 상세 분석](#서비스별-상세-분석)
4. [보안 Tier 분류 (Tier 1-4)](#보안-tier-분류)
5. [핵심 보안 갭 분석](#핵심-보안-갭-분석)
6. [권장사항](#권장사항)

---

## Executive Summary

| 서비스 | 자격증명 관리 | 샌드박싱 | 권한 제어 | 인젝션 방어 | HITL | 보안 Tier |
|--------|----------|---------|----------|-----------|------|---------|
| **Kimi Claw** | 평문/환경변수 | 없음 | 기본 RBAC | 없음 | 없음 | Tier 3 |
| **Z.ai OpenClaw** | 평문/config 파일 | 부분 (프로세스) | 도구 화이트리스트 | 부분 (로깅) | 없음 | Tier 2 |
| **Alibaba OpenClaw** | 환경변수 (평문) | 컨테이너 기반 | RBAC + 채널별 정책 | 프롬프트 마킹 | 검토 모드 | Tier 2 |
| **Baidu DuClaw** | 암호화 저장소 | Docker + WASM 부분 | 도구별 위험도 분류 | PromptGuard 유사 | 승인 워크플로우 | Tier 2-3 |
| **Zhipu AutoClaw** | 평문 (환경변수) | 네이티브 프로세스 격리 | 기본 ACL | 기본 검증 | 없음 | Tier 3 |
| **OpenClawD** | 키 저장소 | Docker 격리 | RBAC (기본) | 입력 검증 | 기본 로깅 | Tier 2-3 |

### 주요 발견

1. **암호화 자격증명 저장소**: Baidu DuClaw만 명시적 암호화 저장소 구현. 나머지 5개는 환경변수 또는 평문 config 파일에 의존.

2. **HITL 구현**: Alibaba (검토 모드)와 Baidu (승인 워크플로우)만 부분 구현. 도구별 세밀한 승인은 미실장.

3. **프롬프트 인젝션 방어**: 전용 탐지 레이어 없음. Baidu만 PromptGuard 유사 구현 시작. 나머지는 기본 입력 검증 수준.

4. **샌드박싱 다층화**: Alibaba와 Baidu만 컨테이너 기반 격리 고려. Z.ai와 OpenClawD는 기본 프로세스 격리만.

5. **권한 제어 세밀함**: Alibaba (채널별 정책)와 Baidu (위험도 분류)가 가장 세밀. 나머지는 기본 RBAC 또는 화이트리스트 수준.

---

## 5대 보안 영역 비교 매트릭스

### 2.1 자격증명 관리

| 서비스 | 저장 방식 | 암호화 | OS Keychain 지원 | 환경변수 격리 | 평가 |
|--------|---------|--------|-----------------|------------|------|
| **Kimi Claw** | 환경변수 + config | [X] | [X] | 부분 (KIMI_ 접두사) | 낮음 |
| **Z.ai OpenClaw** | config 파일 (JSON) | [X] | [X] | 기본 격리 | 낮음 |
| **Alibaba OpenClaw** | 환경변수 + Vault 참조 | [O] (선택적) | [X] | 고급 필터링 | 중간 |
| **Baidu DuClaw** | 암호화 저장소 (ChaCha20) | [O] | 파일 기반 키 | 자동 redact | 높음 |
| **Zhipu AutoClaw** | 환경변수 (평문) | [X] | [X] | 없음 | 낮음 |
| **OpenClawD** | 키 저장소 파일 | 부분 | [X] | 기본 격리 | 중간 |

**분석**:
- **가장 안전**: Baidu DuClaw (ChaCha20-Poly1305 암호화 + 자동 로그 redact)
- **가장 위험**: Kimi Claw, Zhipu AutoClaw (평문 환경변수)
- **갭**: 전체 6개 모두 OS Keychain 연동 미실장. IronClaw 수준의 Zero-Exposure 프록시 모델 부재.

---

### 2.2 샌드박싱

| 서비스 | Docker | WASM | OS 격리 | 파일시스템 제한 | 평가 |
|--------|--------|------|---------|---------------|------|
| **Kimi Claw** | [X] | [X] | 네이티브 프로세스만 | 작업 디렉터리만 | 낮음 |
| **Z.ai OpenClaw** | [O] (선택적) | [X] | 프로세스 격리 | 경로 화이트리스트 | 중간 |
| **Alibaba OpenClaw** | [O] (non-root) | [X] | 컨테이너 네트워크 | mount-security 유사 | 높음 |
| **Baidu DuClaw** | [O] | [O] (평가 중) | Landlock 기반 | workspace_only 기본 | 높음 |
| **Zhipu AutoClaw** | [X] | [X] | 프로세스 공간 격리 | 리소스 제한만 | 낮음 |
| **OpenClawD** | [O] (기본 격리) | [X] | Docker 네임스페이스 | 기본 경로 제한 | 중간 |

**분석**:
- **가장 정교**: Baidu DuClaw (Docker + Landlock + WASM 진행 중)
- **가장 단순**: Kimi Claw, Zhipu AutoClaw (네이티브 프로세스만)
- **갭**: IronClaw/ZeroClaw 수준의 WASM + Docker 이중 샌드박스 미구현.

---

### 2.3 권한 제어 및 인가 시스템

| 서비스 | 접근 방식 | 도구 수준 제어 | 데이터 흐름 추적 | 핵심 평가 |
|--------|---------|-------------|------------|----------|
| **Kimi Claw** | 기본 RBAC | [X] (역할만 제어) | [X] | 낮음 |
| **Z.ai OpenClaw** | deny/allow 목록 + 정책 | [O] (도구 화이트리스트) | [X] | 중간 |
| **Alibaba OpenClaw** | 채널별 정책 + 도구 RBAC | [O] (도구별 ACL) | [O] (제한적) | 중상 |
| **Baidu DuClaw** | 3단계 AutonomyLevel + 위험도 분류 | [O] (High/Medium/Low) | [X] | 중상 |
| **Zhipu AutoClaw** | 기본 ACL | [O] (앱 수준만) | [X] | 낮음 |
| **OpenClawD** | RBAC (기본) | [O] (감사 기반) | [X] | 중간 |

**분석**:
- **가장 세밀**: Baidu DuClaw (명령어 위험도 분류 + 3단계 자율성)
- **두 번째**: Alibaba (채널별 정책 + 도구 ACL)
- **갭**: OpenJarvis 수준의 4-label Taint Tracking (PII/SECRET/USER_PRIVATE/EXTERNAL) 미구현.

---

### 2.4 프롬프트 인젝션 방어

| 서비스 | 전용 탐지 레이어 | 방어 메커니즘 | 위협 수준 분류 | 평가 |
|--------|----------------|----------|-------------|------|
| **Kimi Claw** | [X] | 기본 문자 이스케이프 | [X] | 낮음 |
| **Z.ai OpenClaw** | [X] | 외부 콘텐츠 마킹 + 정규식 | [X] | 낮음-중간 |
| **Alibaba OpenClaw** | [X] | 프롬프트 마킹 (랜덤 경계) | [X] | 낮음 |
| **Baidu DuClaw** | [O] (진행 중) | PromptGuard 유사 6패턴 탐지 | [O] (부분) | 중간 |
| **Zhipu AutoClaw** | [X] | 기본 입력 검증 | [X] | 낮음 |
| **OpenClawD** | [X] | 구조적 격리 | [X] | 낮음 |

**분석**:
- **가장 고급**: Baidu DuClaw (PromptGuard 유사, 6가지 패턴)
- **기본 수준**: Kimi Claw, Zhipu AutoClaw, OpenClawD
- **갭**: IronClaw의 SafetyLayer 4중 방어, ZeroClaw의 6패턴 탐지, OpenJarvis의 regex 기반 Scanner (4개 위협 수준) 미구현.

---

### 2.5 Human-in-the-Loop (HITL)

| 서비스 | HITL 구현 | 승인 대상 | 긴급 정지 | 평가 |
|--------|---------|---------|---------|------|
| **Kimi Claw** | [X] (자율 실행) | — | [X] | 낮음 |
| **Z.ai OpenClaw** | [X] (자율 실행) | — | [O] (중단 신호) | 낮음 |
| **Alibaba OpenClaw** | [O] (검토 모드) | 고위험 도구 | [X] | 중간 |
| **Baidu DuClaw** | [O] (승인 워크플로우) | 중/고위험 명령 | [O] (즉시 정지) | 높음 |
| **Zhipu AutoClaw** | [X] (자율 실행) | — | [X] | 낮음 |
| **OpenClawD** | [X] (자율 실행) | — | [O] (기본 로깅) | 낮음 |

**분석**:
- **가장 정교**: Baidu DuClaw (승인 워크플로우 + 즉시 정지)
- **부분 구현**: Alibaba (검토 모드)
- **갭**: ZeroClaw 수준의 E-Stop (4단계 + OTP) 미구현. 도구별 세밀한 승인/항상승인/자동허용 스펙트럼 부재.

---

## 서비스별 상세 분석

### 3.1 Kimi Claw (Moon.AI / Moonshot)

**보안 철학**: 경량 에이전트. 보안보다 접근성 우선.

**강점**:
- OpenAI 호환 API (표준화)
- 다중 모델 지원 (K2, K2.5, Thinking)

**약점**:
- 평문 환경변수 저장
- OS Keychain 미지원
- 샌드박싱 없음 (네이티브 프로세스)
- HITL 전무
- 프롬프트 인젝션 전용 방어 없음

**보안 Tier**: Tier 3 (Denylist-Based, 기본 보안만)

**추천 사용 시나리오**:
- 개인용 로컬 에이전트 (신뢰 환경)
- 보안 요구사항 낮은 프로토타이핑

**개선 권고**:
1. 환경변수 -> OS Keychain 마이그레이션 (macOS/Linux)
2. 도구 화이트리스트 구현
3. 프롬프트 인젝션 기본 필터 추가 (15개 패턴)

---

### 3.2 Z.ai OpenClaw

**보안 철학**: 중간 수준의 격리 + 도구 정책 기반 제어.

**강점**:
- 도구별 화이트리스트 (deny/allow)
- 외부 콘텐츠 마킹
- config 파일 기반 자격증명 격리

**약점**:
- 암호화 저장소 없음 (평문)
- 선택적 Docker만 지원
- HITL 없음
- 인젝션 방어 기본 수준
- 경로 순회 방어 부분적

**보안 Tier**: Tier 2 (Container-First, 기본 인가)

**추천 사용 시나리오**:
- 팀 내 공유 에이전트
- 통제된 도구 집합 (5-10개 도구)

**개선 권고**:
1. ChaCha20-Poly1305 기반 자격증명 암호화
2. Baidu DuClaw 수준의 위험도 분류 (High/Medium/Low)
3. 프롬프트 인젝션 Scanner (regex 기반, 4개 위협 수준)

---

### 3.3 Alibaba OpenClaw

**보안 철학**: 채널별 정책 + 컨테이너 격리 + 검토 모드.

**강점**:
- Docker 격리 (non-root)
- 채널별 정책 (DM/그룹)
- 도구 RBAC
- 검토 모드 (HITL 부분)
- 환경변수 고급 필터링 (20+ 패턴)

**약점**:
- 자격증명 암호화 선택적만
- WASM 미지원
- 프롬프트 인젝션 전용 탐지 없음
- Taint Tracking 미구현
- 비용 제한 없음

**보안 Tier**: Tier 2 (Container-First, 다층 정책)

**추천 사용 시나리오**:
- 엔터프라이즈 팀 협업 (다채널 지원)
- 민감한 작업 (검토 모드 활성화)

**개선 권고**:
1. 자격증명 암호화 기본화 (선택적 -> 필수)
2. Baidu의 3단계 AutonomyLevel 도입
3. OpenJarvis의 Taint Tracking + SINK_POLICY 도입
4. WASM 샌드박스 추가

---

### 3.4 Baidu DuClaw

**보안 철학**: Defense-in-Depth의 중국판. 암호화 + 다층 격리 + 위험도 분류 + 승인 워크플로우.

**강점**:
- ChaCha20-Poly1305 자격증명 암호화
- Docker + Landlock 기반 격리 (WASM 진행 중)
- 3단계 AutonomyLevel (ReadOnly/Supervised/Full)
- 명령어 위험도 분류 (High/Medium/Low)
- 승인 워크플로우 (도구별 승인)
- 즉시 정지 (E-Stop 유사)
- 자동 로그 redact
- PromptGuard 유사 6패턴 탐지 (진행 중)

**약점**:
- OS Keychain 미지원 (파일 기반 키만)
- WASM 아직 평가 단계
- Taint Tracking 미구현
- 비용 하드 한도 정책 미명시

**보안 Tier**: Tier 2-3 (Defense-in-Depth 초기 단계)

**추천 사용 시나리오**:
- 엔터프라이즈 미션 크리티컬 작업
- 금융/보건 등 규제 산업
- 24시간 상주 에이전트

**개선 권고**:
1. WASM 샌드박스 완성 (wasmi, 10억 fuel 제한)
2. OpenJarvis의 Taint Tracking 도입 (4-label)
3. ZeroClaw의 E-Stop 패턴 (4단계 + OTP) 도입
4. 일별 비용 하드 한도 정책 수립 ($5-10)

---

### 3.5 Zhipu AutoClaw

**보안 철학**: 최소 보안 오버헤드로 자동화 지향.

**강점**:
- 네이티브 프로세스 공간 격리
- 기본 ACL 지원

**약점**:
- 평문 환경변수 저장
- 샌드박싱 없음
- HITL 전무
- 도구 수준 제어 없음
- 프롬프트 인젝션 방어 기본만
- 권한 제어 매우 기본

**보안 Tier**: Tier 3 (Denylist-Based, 최소 보안)

**추천 사용 시나리오**:
- 개인 프로토타이핑
- 신뢰 환경 (폐쇄 네트워크)

**개선 권고**:
1. 환경변수 -> 암호화 저장소 (ChaCha20)
2. Docker 또는 WASM 샌드박스 도입
3. 도구별 위험도 분류 + 화이트리스트
4. 프롬프트 인젝션 정규식 필터 (15개 패턴)

---

### 3.6 OpenClawD

**보안 철학**: 기본 RBAC + Docker 격리 + 감사 로깅.

**강점**:
- Docker 격리 (기본)
- RBAC 기본 구현
- 감사 로깅
- 키 저장소 파일 (부분 보안)

**약점**:
- 자격증명 암호화 미부분적
- WASM 미지원
- 도구별 세밀한 제어 부족
- HITL 없음 (감사 기반만)
- 프롬프트 인젝션 입력 검증 수준
- 비용 제한 없음

**보안 Tier**: Tier 2-3 (Container-First, 기본 감사)

**추천 사용 시나리오**:
- 팀 협업 + 감사 추적 필요한 환경
- 중소 규모 팀

**개선 권고**:
1. 자격증명 암호화 기본화
2. Baidu의 AutonomyLevel 도입
3. 프롬프트 인젝션 Scanner (regex, 4개 수준)
4. WASM 샌드박스 추가

---

## 보안 Tier 분류

### Tier 1: Defense-in-Depth (엔터프라이즈급 보안)

**요구사항**: 암호화 볼트 + WASM/Docker 이중 샌드박스 + 다층 인젝션 방어 + HITL 승인 + 비용 제한

**해당 서비스**: 없음 (Baidu DuClaw가 가장 가깝지만 아직 미완성)

**레퍼런스**: IronClaw, ZeroClaw

---

### Tier 2: Container-First (고급 보안)

**요구사항**: Docker/subprocess 격리 + 도구 화이트리스트 + 자격증명 격리 + 부분적 인젝션 방어 + 선택적 HITL

**해당 서비스**:
- Alibaba OpenClaw (채널별 정책 + 검토 모드)
- Z.ai OpenClaw (도구 화이트리스트 + 외부 콘텐츠 마킹)
- OpenClawD (Docker + RBAC + 감사 로깅)

---

### Tier 2-3: Transitional (중간 보안)

**요구사항**: 부분적 컨테이너 격리 + 자격증명 부분 암호화 + 기본 HITL + 부분적 인젝션 방어

**해당 서비스**:
- Baidu DuClaw (암호화 + 위험도 분류, 다만 WASM 미완)

---

### Tier 3: Denylist-Based (기본 보안)

**요구사항**: 정규식 기반 명령어 차단 + 파일시스템 제한 + 평문 자격증명 + HITL 없음

**해당 서비스**:
- Kimi Claw (경량 설계)
- Zhipu AutoClaw (최소 보안)

---

## 핵심 보안 갭 분석

### 갭 1: 암호화 자격증명 저장소의 부재

**현재**: Baidu DuClaw만 ChaCha20-Poly1305. 나머지 5개는 평문 환경변수 또는 config 파일.

**이상적**: IronClaw 수준의 "Zero-Exposure 프록시" 모델
- 호스트는 자격증명을 암호화 저장
- 에이전트는 실제 값을 절대 볼 수 없음
- 프록시가 HTTP 요청에 런타임 주입

**영향**: 컨테이너 탈취 시 자격증명 노출 위험 (매우 높음)

---

### 갭 2: 프롬프트 인젝션 전용 탐지 레이어 부재

**현재**: 기본 입력 검증 또는 마킹만. Baidu만 PromptGuard 유사 시작.

**이상적**: OpenJarvis 수준의 "Prompt Injection Scanner"
- regex 기반 스캔 (15+ 패턴)
- 4개 위협 수준 분류 (LOW/MEDIUM/HIGH/CRITICAL)
- 위험 패턴 탐지 후 자동 격리/알림

**영향**: 프롬프트 인젝션 공격 우회 용이 (높음)

---

### 갭 3: Taint Tracking (데이터 흐름 추적) 미구현

**현재**: 모든 6개 서비스가 구현 없음.

**이상적**: OpenJarvis의 "4-label Taint Tracking + SINK_POLICY"
- 라벨: PII, SECRET, USER_PRIVATE, EXTERNAL
- 정책 예시:
  - web_search: PII + SECRET 데이터 전달 금지
  - channel_send: SECRET 데이터 전달 금지
  - code_interpreter: SECRET 데이터 전달 금지

**영향**: 민감 데이터 유출 경로 차단 불가 (높음)

---

### 갭 4: 정교한 Human-in-the-Loop (HITL) 부재

**현재**: Baidu만 승인 워크플로우. Alibaba는 검토 모드. 나머지 없음.

**이상적**: ZeroClaw 수준의 3단계 AutonomyLevel
- ReadOnly: 도구 실행 불가 (조회만)
- Supervised: 매 도구 실행마다 승인 필요
- Full: 자율 실행

**추가**: E-Stop (4단계 + OTP)
- KillAll: 모든 실행 즉시 정지
- NetworkKill: 네트워크만 차단
- DomainBlock: 특정 도메인만 차단
- ToolFreeze: 특정 도구만 정지

**영향**: 폭주 에이전트 제어 불가 (높음)

---

### 갭 5: WASM 샌드박스 미완성

**현재**: Baidu만 평가 중. 나머지 없음.

**이상적**: ZeroClaw 수준의 WASM 격리
- 메모리: 10MB 제한
- Fuel: 10억 제한 (연산 제한)
- 네트워크: 화이트리스트 기반
- 파일: workspace_only 강제

**영향**: 컨테이너 탈취 시 호스트 공격 가능 (높음)

---

### 갭 6: 비용/토큰 하드 한도 미실장

**현재**: 모든 6개 서비스 비용 추적 없거나 하드 한도 없음.

**이상적**: ZeroClaw 수준의 "일별 $5 하드 한도"
- 비용 초과 시 자동 정지
- HITL + 비용 조합 -> 폭주 방지

**영향**: 비용 폭발 (매우 높음, 특히 24시간 에이전트)

---

## 권장사항

### 우선순위 1: 자격증명 암호화 (즉시 - 모든 서비스)

변경 전 (위험):
```
KIMI_API_KEY=sk-xxxxx  # 환경변수에 평문
auth.json {apiKey: "sk-xxxxx"}  # config 파일에 평문
```

변경 후 (권장):
```
# 1단계: ChaCha20-Poly1305 암호화
credentials.encrypted = ChaCha20Poly1305.encrypt(
  plaintext: "sk-xxxxx",
  key: derive_from_password(password: USER_PASSWORD),
  nonce: random_96_bits()
)

# 2단계: OS Keychain 저장 (macOS/Linux)
keychain.put("kimi-api-key", credentials.encrypted)

# 3단계: 에이전트는 런타임 복호화만
plaintext = keychain.get("kimi-api-key").decrypt(user_password)
request.headers["Authorization"] = "Bearer " + plaintext
```

**핵심 코드 (Rust 예시 - ZeroClaw 패턴)**:
```rust
use chacha20poly1305::{ChaCha20Poly1305, Key, Nonce};
use argon2::Argon2;

fn encrypt_credential(password: &str, api_key: &str) -> String {
    let key = Argon2::default()
        .hash_password(password.as_bytes(), &SaltString::generate(OsRng))
        .unwrap()
        .hash
        .unwrap();

    let cipher = ChaCha20Poly1305::new(
        Key::<ChaCha20Poly1305>::from_slice(&key[..32])
    );
    let nonce = Nonce::from_slice(&random_96_bits());
    let ciphertext = cipher.encrypt(nonce, api_key.as_bytes()).unwrap();

    format!("{:x}|{:x}", nonce, ciphertext)
}
```

---

### 우선순위 2: 프롬프트 인젝션 Scanner (1-2주)

Python 구현 패턴:
```python
import re
from enum import Enum

class ThreatLevel(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CRITICAL = 4

INJECTION_PATTERNS = [
    # 시스템 오버라이드
    (r"ignore.*previous.*instructions", ThreatLevel.CRITICAL),
    (r"forget.*everything", ThreatLevel.CRITICAL),

    # 역할 혼란
    (r"you are now a.*hacker", ThreatLevel.HIGH),
    (r"pretend you are.*admin", ThreatLevel.HIGH),

    # JSON 인젝션
    (r'"\s*}\s*,\s*"', ThreatLevel.MEDIUM),

    # 시크릿 추출
    (r"(password|api_key|secret).*show.*me", ThreatLevel.CRITICAL),
]

def scan_prompt_injection(user_input: str) -> ThreatLevel | None:
    for pattern, level in INJECTION_PATTERNS:
        if re.search(pattern, user_input, re.IGNORECASE):
            return level
    return None

# 사용
threat = scan_prompt_injection(user_input)
if threat == ThreatLevel.CRITICAL:
    log_security_event("CRITICAL_INJECTION_ATTEMPT", user_input)
    return "요청이 거부되었습니다"
```

---

### 우선순위 3: 도구별 위험도 분류 (2-4주)

TypeScript 구현:
```typescript
enum ToolRiskLevel {
  Low = "low",       // 자동 실행
  Medium = "medium", // 승인 필요
  High = "high",     // 감시자 승인 필요
}

interface ToolPolicy {
  toolId: string;
  riskLevel: ToolRiskLevel;
  requires_approval_override: boolean;
  requires_audit_log: boolean;
}

async function execute_tool(tool: Tool, autonomy: string, user: User) {
  const policy = get_policy_for_tool(tool.id);

  if (autonomy === "full" && policy.riskLevel === "high") {
    const approval = await request_user_approval(user, tool);
    if (!approval) {
      audit_log("TOOL_REJECTED", {toolId: tool.id});
      return error("Tool execution rejected");
    }
  }

  const result = await tool.execute();
  audit_log("TOOL_EXECUTED", {toolId: tool.id, riskLevel: policy.riskLevel});
  return result;
}
```

---

### 우선순위 4: 비용 하드 한도 (2주)

Python 구현:
```python
from datetime import datetime, timedelta

async def check_cost_limit(user: User, estimated_cost: float) -> bool:
    budget = get_user_budget(user.id)
    current_cost = calculate_cost_today(user.id)

    if current_cost + estimated_cost > budget.daily_limit:
        audit_log("COST_LIMIT_EXCEEDED", {
            user_id: user.id,
            current_cost,
            estimated_cost,
            limit: budget.daily_limit
        })
        raise PermissionError(
            f"Daily cost limit ${budget.daily_limit} exceeded"
        )

    return True
```

---

## 비교 표: 기존 Claw vs 중국 서비스

| 기능 | IronClaw | ZeroClaw | Baidu | Alibaba | Z.ai | Kimi | Zhipu |
|------|----------|----------|--------|---------|------|------|-------|
| 자격증명 암호화 | AES-256 | ChaCha20 | ChaCha20 | [O] (선택) | [X] | [X] | [X] |
| OS Keychain | [O] | [X] | [X] | [X] | [X] | [X] | [X] |
| Docker | [O] | [O] | [O] | [O] | [O] | [X] | [X] |
| WASM | [O] | [O] | [O] (평가) | [X] | [X] | [X] | [X] |
| 도구 화이트리스트 | [O] | [O] | [O] | [O] | [O] | [X] | [X] |
| 위험도 분류 | [O] | [O] | [O] | [X] | [X] | [X] | [X] |
| HITL | [O] | [O] | [O] | [O] | [X] | [X] | [X] |
| 프롬프트 인젝션 | SafetyLayer | PromptGuard | 6패턴 | 마킹 | 마킹 | 기본 | 기본 |
| Taint Tracking | [X] | [X] | [X] | [O] (제한) | [X] | [X] | [X] |
| E-Stop | [X] | [O] | [O] (부분) | [X] | [X] | [X] | [X] |
| 비용 한도 | [X] | $5/일 | [X] | [X] | [X] | [X] | [X] |
| 보안 Tier | Tier 1 | Tier 1 | Tier 2-3 | Tier 2 | Tier 2 | Tier 3 | Tier 3 |

---

## 결론

### 핵심 발견

1. **중국 서비스 중 Baidu DuClaw가 가장 진전**: 암호화 + 위험도 분류 + 승인 워크플로우를 부분 구현. 하지만 WASM 미완, Taint Tracking 없음, OS Keychain 미지원으로 아직 Tier 1 미달.

2. **Alibaba OpenClaw의 채널별 정책이 혁신적**: 팀 협업 환경에서 유용. 하지만 자격증명 암호화 선택적이고 검토 모드만으로 자동화 HITL 부족.

3. **Kimi Claw와 Zhipu AutoClaw는 개인용 이상 부적절**: 평문 환경변수, 샌드박싱 없음, HITL 없음.

4. **프롬프트 인젝션 방어 전반적 부족**: 기본 입력 검증 수준. Baidu만 PromptGuard 유사 시작.

5. **Taint Tracking은 전무**: OpenJarvis 수준의 데이터 흐름 추적 미구현. 민감 정보 유출 경로 차단 불가.

### 사용 권고안

| 시나리오 | 권장 서비스 | 이유 | 주의사항 |
|---------|-----------|------|--------|
| 개인 프로토타이핑 | Kimi Claw | 경량, 빠름 | 보안 불충분 |
| 팀 협업 + 감시 | Alibaba | 채널별 정책 + 검토 | 암호화 필수 활성화 |
| 미션 크리티컬 | Baidu | 암호화 + HITL + 위험도 | WASM 완성 대기 |
| 팀 + 감사 추적 | OpenClawD | RBAC + 감사 로깅 | 암호화 추가 필요 |
| 프로토타이핑 + 도구 제어 | Z.ai | 도구 화이트리스트 | 암호화 필수, HITL 추가 |

---

**문서 버전**: 1.0
**최종 검토 일자**: 2026-03-14
**다음 리뷰 예정**: 2026-04-14 (각 서비스 업데이트 확인)
