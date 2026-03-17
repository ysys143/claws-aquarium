# 9개 Claw 서비스 종합 분석 리포트
## Compare Claws Project Final Report

> **조사 기간**: 2026-03-05 ~ 2026-03-14
> **조사 대상**: 9개 오픈소스 에이전트 런타임 프레임워크
> **조사 방법**: 8개 scientist/architect 에이전트의 병렬 심층 코드 분석
> **최종 보고서**: 5개 기존 보고서 + 2개 신규 도구 분석 통합

---

## Executive Summary

### 핵심 발견 4가지

1. **4계층 성숙도 아키텍처**: 10개 구현체는 보안/기억/세션 관리 복잡도에 따라 Tier 1(엔터프라이즈) ~ Tier 4(실험)로 분류된다. **Tier 1 삼총사(IronClaw, ZeroClaw, OpenClaw)의 기술 선택이 완전히 다르면서도 각각 우수하다**는 발견이 가장 중요.

2. **24시간 상주 에이전트의 요구사항은 기존 시스템 프롬프트 기반 에이전트와 근본적으로 다르다**: 세션/컨텍스트 관리, 기억 아키텍처, 보안 경계, 도구 격리가 모두 새로운 수준의 복잡도를 요구한다. 이는 "더 큰 모델"이 아니라 "다른 시스템"을 필요로 한다.

3. **기억 아키텍처가 보안 아키텍처를 결정한다**: Tier 1 삼총사의 기억 시스템(Vector DB + FTS + 하이브리드 검색)은 모두 Tier 1 보안 패턴(암호화 + WASM + HITL)과 상관관계를 가진다. 단순한 마크다운 기반 기억(Nanobot, PicoClaw)을 가진 구현체는 보안도 최소 수준.

4. **연구 자동화와 범용 에이전트는 다른 설계 원리를 따른다**: DeepInnovator(연구 특화)와 Autoresearch(ML 실험)의 두 도구는 기존 9개 런타임과 전혀 다른 패턴을 보여준다. Tier 1 기억 + 고정 예산 + 자동 검증이 조합되면 새로운 "연구 에이전트 스택"이 가능하다.

---

## 1. 목차

1. Executive Summary
2. 9개 서비스 비교 매트릭스
3. 4계층 성숙도 분류 (Tier 1-4)
4. 서비스별 평가표 (Claw형 기준)
5. 보안 계층 분류
6. 강점/약점 분석 (용도별 추천)
7. 시장 트렌드 및 결론

---

## 2. 9개 서비스 비교 매트릭스

### 2.1 기본 정보

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|
| **언어** | TypeScript | Python | TypeScript | Rust | Rust | Go | TypeScript | Python | Rust |
| **규모** | 430K+ LOC | 4K LOC | 20K LOC | 15K LOC | 12K LOC | 10K LOC | 5K LOC | 8K LOC | ~20K LOC |
| **저자** | 범용 (open) | 개인 | Anthropic | Anthropic | 개인 | 개인 | 개인 | Lyric Labs | 개인 |
| **라이선스** | 상업/오픈 | MIT | 상업 | 상업 | MIT | MIT | MIT | 상업 | MIT |
| **주요 용도** | 범용 에이전트 | 개인 에이전트 | 사내 배포 | 엔터프라이즈 | 개인 고급 | 모바일/엣지 | 팀 협업 | 다중채널 | MCP 표준 |

### 2.2 런타임 특성

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|
| **배포 방식** | 호스팅됨 | 자체 배포 | K8s/Docker | 자체 배포 | 자체 배포 | 자체 배포 | 자체 배포 | 자체 배포 | 자체 배포 |
| **컨테이너** | Docker | [X] | Docker | Docker | [X] | [X] | [X] | [X] | [X] |
| **WASM 샌드박스** | [X] | [X] | [X] | [O] (이중) | [O] | [X] | [X] | [X] | [O] |
| **에이전트 수** | 1+ | 1 (spawn) | 팀 (Teams) | 1 | 1 | 1 | N (분산) | 1+ | 1 |
| **프로세스 격리** | [X] | [X] | [O] | [O] | [O] | [X] | [X] | [X] | [O] |

### 2.3 세션/컨텍스트 관리

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|
| **세션 키 전략** | agent:<id>:<type>:<uuid> | channel:chat_id | group_folder | (user, channel, thread) | session_id (DB) | agent:id:ch:kind:peer | agent_dir | session_id + channel_ids | Config |
| **멀티에이전트** | [O] | [O] | [O] | [O] | [X] | [X] | [O] | [O] | [X] |
| **컴팩션** | 동기 | 비동기 LLM | SDK 자동 | 선택 가능 | 동기 | 비동기 | SDK 자동 | 비동기 | 없음 |
| **장기기억** | [O] (LanceDB) | [O] (.md) | DELTA | [O] (벡터) | [O] (SQLite) | [O] (.md) | DELTA | [O] (SQLite) | DELTA |
| **Cross-Channel** | [X] | [X] | [X] | [X] | [X] | [X] | [X] | [O] | [X] |

### 2.4 기억 아키텍처

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|
| **저장 백엔드** | SQLite + LanceDB | 파일 (.md) | 파일 (아카이브) | PostgreSQL + libSQL | SQLite + FTS5 | 파일 (.md) | 파일 (아카이브) | SQLite | 마크다운 |
| **임베딩** | [O] (6개) | [X] | [X] | [O] (4개) | [O] (3개) | [X] | [X] | [X] | [X] |
| **하이브리드 검색** | [O] (4단계) | [X] | [X] | [O] (RRF) | [O] (linear) | [X] | [X] | [X] | [X] |
| **수명주기** | Atomic | 없음 | 없음 | 30일 | 12h | 없음 | 없음 | 24h | 없음 |
| **Soul Snapshot** | [X] | [X] | [X] | [X] | [O] | [X] | [X] | [X] | [X] |

### 2.5 보안/권한

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|
| **암호화 볼트** | [X] | [X] | [X] | [O] (AES) | [O] (ChaCha) | [X] | [X] | [X] | [X] |
| **HITL** | [O] | [X] | [X] | [O] | [O] | [X] | DELTA | [X] | [O] |
| **Prompt Injection** | DELTA | [X] | [X] | [O] (4중) | [O] (6패턴) | [X] | [X] | [O] | [X] |
| **Taint Tracking** | [X] | [X] | [X] | [X] | [X] | [X] | [X] | [O] | [X] |
| **비용 제한** | DELTA | [X] | [X] | DELTA | [O] ($5) | [X] | [X] | [X] | [X] |

### 2.6 도구/액션 시스템

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|
| **도구 유형** | TS Plugin | Python ABC | SKILL.md | Rust Trait | Rust Trait | Go Iface | CLI | Python Class | HAND.toml |
| **MCP 지원** | [O] | [O] | [O] | [O] | [O] | [O] | [O] | [O] | [O] |
| **병렬 실행** | [X] | [X] | [X] | [X] | [X] | [O] | [X] | [O] | [X] |
| **브라우저** | [O] (50+) | [X] | DELTA | DELTA | [O] (16) | [X] | [X] | [X] | [X] |
| **SSRF 방지** | [O] | [X] | [X] | [O] | [O] | [X] | [X] | DELTA | [X] |

### 2.7 메신저 인터페이스

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|
| **지원 범위** | Discord+Slack | Telegram | Discord | Discord | Telegram | Telegram | 팀 | 다중 | MCP |
| **채널 정책** | [O] | [O] | [O] | [O] | DELTA | DELTA | DELTA | [O] | [X] |
| **다중채널 세션** | [X] | [X] | [X] | [X] | [X] | [X] | [X] | [O] | [X] |

---

## 3. 4계층 성숙도 분류

### Tier 1: Defense-in-Depth (엔터프라이즈)

**구현체**: IronClaw, ZeroClaw, OpenClaw

**공통 특성**:
- 암호화 볼트 또는 암호화 저장 (IronClaw AES-256-GCM, ZeroClaw ChaCha20-Poly1305, OpenClaw 평문)
- Tier 1 기억 시스템 (벡터 DB + FTS + 하이브리드 검색)
- 다층 보안 방어 (WASM + Docker + 정책 엔진)
- HITL 승인 시스템 (완전 또는 부분)
- Prompt Injection 전용 방어 레이어

**차이점**:

| 측면 | IronClaw | ZeroClaw | OpenClaw |
|------|----------|----------|----------|
| **철학** | Zero-Exposure (컨테이너 자격증명 미노출) | 정책 기반 세밀 제어 | 실용적 다층 방어 |
| **기억** | pgvector RRF | SQLite+vec 선형 fusion | LanceDB 4단계 파이프라인 |
| **비용** | 추적만 (무한) | $5/일 하드 한도 | 추적만 (무한) |
| **WASM** | 이중 (도구+실행) | 단일 (실행용) | 없음 |
| **특수 기능** | Zero-Exposure 프록시 | E-Stop + Soul Snapshot | 50+ 브라우저 기능 |

**선택 기준**:
- **IronClaw**: 가장 높은 보안 요구사항 (금융/의료)
- **ZeroClaw**: 비용 인식 + 모바일 환경 + 스냅샷 복원 필요
- **OpenClaw**: 최대 기능 + 생태계 + 상용 지원

---

### Tier 2: Container-First (중급)

**구현체**: NanoClaw, OpenJarvis

**공통 특성**:
- Docker/컨테이너 격리 (NanoClaw) 또는 SQLite 기반 격리 (OpenJarvis)
- 부분적 기억 시스템 (자동 로딩 없거나 제한적)
- 도구 허용목록 + 환경변수 격리
- HITL 없음 (자율 실행)

**NanoClaw 특이성**:
- 그룹별 독립 Docker 컨테이너 (강한 물리적 격리)
- 시크릿 stdin 전달 + 즉시 삭제 (메모리 보안)
- SDK Agent Teams (네이티브 Anthropic 통합)

**OpenJarvis 특이성**:
- Cross-channel SessionIdentity (Slack+Telegram+Web 통합)
- RLM Context-as-Variable (컨텍스트 윈도우 외재화)
- Taint Tracking (10개 중 유일)
- RBAC 이중 구현 (Python+Rust)

---

### Tier 3: Denylist-Based (경량)

**구현체**: Nanobot, PicoClaw

**공통 특성**:
- 정규식/패턴 기반 차단 (화이트리스트 아님)
- 파일 기반 기억 (MEMORY.md + HISTORY.md)
- 자동 수명주기 관리 없음 (무한 성장)
- 사람이 읽을 수 있는 형식 (투명성)

**Nanobot 특이성**:
- 2계층 메모리 (MEMORY.md + HISTORY.md)
- LLM 기반 능동적 consolidation
- 외부 의존성 최소 (자체 포함)

**PicoClaw 특이성**:
- 77개 정규식 denylist (최다)
- Go의 os.Root 파일시스템 샌드박스
- 원자적 쓰기 (플래시 스토리지 안정성)
- mtime 기반 캐싱 (Tier 1/2 중 유일)

---

### Tier 4: Minimal/None (실험)

**구현체**: TinyClaw, OpenFang

**TinyClaw**:
- 보안 메커니즘 최소
- 분산 액터 모델 (독특)
- CORS 와일드카드
- 기억 미구현

**OpenFang**:
- MCP 표준 중심 (도구 자체 구현 최소)
- HAND.toml 선언적 능력
- WASM Dual Metering (새로운 패턴)
- 60개 빌트인 string dispatch

---

## 4. Claw형 기준 평가표

### 4.1 24/7 상시 구동 (가능 여부)

| 구현체 | 평가 | 사유 |
|--------|------|------|
| **ZeroClaw** | [O] (최적) | 비용 하드 한도 + E-Stop으로 자동 제어 |
| **IronClaw** | [O] (우수) | 보안 + 비용 추적, 한도는 없지만 로깅 |
| **OpenClaw** | [O] (우수) | 기능 최다 + 비용 추적, 한도 자동화 권장 |
| **OpenJarvis** | DELTA | 다중채널 우수, 기억 수명주기 자동 |
| **NanoClaw** | DELTA | 그룹별 격리 우수, 비용 제한 없음 |
| **PicoClaw** | DELTA | 경량 우수, 기억 수명주기 없음 |
| **Nanobot** | DELTA | 단순하지만 기억 무한 성장 |
| **TinyClaw** | [X] | 보안 최소 |
| **OpenFang** | [X] | 상용화 미흡 |

### 4.2 메신저 인터페이스 지원

| 구현체 | Discord | Slack | Telegram | 다중채널 | 특징 |
|--------|---------|-------|----------|---------|------|
| **OpenClaw** | [O] | [O] | [X] | DELTA | 가장 완성도 높음 |
| **Nanobot** | [X] | [X] | [O] | [X] | 텔레그램 전문 |
| **NanoClaw** | [O] | [X] | [X] | [X] | Discord 전용 |
| **IronClaw** | [O] | [X] | [X] | [X] | Discord 전용 |
| **ZeroClaw** | [X] | [X] | [O] | [X] | 텔레그램 전용 |
| **PicoClaw** | [X] | [X] | [O] | [X] | 텔레그램 전용 |
| **TinyClaw** | [O] | DELTA | DELTA | [O] | 팀 협업 우선 |
| **OpenJarvis** | [O] | [O] | [O] | [O] | **최고의 다중채널** |
| **OpenFang** | DELTA | DELTA | DELTA | DELTA | MCP 위임 |

### 4.3 자율성 수준 (권한 모델)

| 구현체 | ReadOnly | Supervised | Full | 특징 |
|--------|----------|-----------|------|------|
| **ZeroClaw** | [O] | [O] | [O] | 3단계 AutonomyLevel + E-Stop |
| **IronClaw** | DELTA | [O] | [O] | 승인 요청 + 자율 잡 |
| **OpenClaw** | [X] | [O] | [O] | 실행 승인 요청 |
| **OpenJarvis** | [X] | [X] | [O] | RBAC 사후 추적 |
| **PicoClaw** | [X] | [X] | [O] | 자율 실행 |
| **Nanobot** | [X] | [X] | [O] | 자율 실행 |
| **NanoClaw** | [X] | [X] | [O] | 자율 실행 |
| **TinyClaw** | DELTA | DELTA | [O] | Pairing (채널 접근만) |
| **OpenFang** | [O] | DELTA | [O] | HAND.toml 선언적 |

### 4.4 커넥터 수 및 다양성

| 구현체 | MCP | 네이티브 | 브라우저 | 총합 | 평가 |
|--------|-----|----------|---------|------|------|
| **OpenClaw** | [O] (6+) | 50+ | 50+ | 100+ | [TOP] |
| **ZeroClaw** | [O] | 16+ | 16 | 30+ | [GOOD] |
| **IronClaw** | [O] | 20+ | 테스트용 | 20+ | [GOOD] |
| **OpenJarvis** | [O] | 10+ | [X] | 10+ | [OK] |
| **Nanobot** | [O] | 10+ | [X] | 10+ | [OK] |
| **PicoClaw** | [O] | 15+ | [X] | 15+ | [OK] |
| **NanoClaw** | [O] | X/Twitter | X/Twitter | 5+ | [BASIC] |
| **TinyClaw** | [O] | 5+ | [X] | 5+ | [BASIC] |
| **OpenFang** | [O] (표준) | 60+ | [X] | 60+ | [GOOD] |

---

## 5. 보안 Tier 분류

기존 security_report.md (2026-03-05)의 7대 보안 영역 분석 결과:

### Tier 1: Defense-in-Depth
**IronClaw, ZeroClaw, OpenJarvis**

| 보안 영역 | IronClaw | ZeroClaw | OpenJarvis |
|---------|----------|----------|-----------|
| 권한/인가 | Bearer + Skills | 3단계 AutonomyLevel | RBAC 10종 (이중) |
| 자격증명 | AES-256-GCM | ChaCha20-Poly1305 | Taint Tracking |
| 샌드박싱 | WASM + Docker | WASM + 다중 | subprocess + SSRF |
| 도구 실행 | 프록시 경유 | 5중 방어 | Taint 기반 차단 |
| HITL | 도구별 승인 | E-Stop (4단계) | 사후 추적 |
| 비용 제한 | 기록 (무한) | $5/일 하드 | 속도 제한 |
| Prompt Injection | SafetyLayer 4중 | PromptGuard 6패턴 | Scanner + Taint |

### Tier 2: Container-First
**OpenClaw, NanoClaw**

- 도구 수준 deny/allow
- Docker 격리
- 환경변수 위생
- HITL 부분적

### Tier 3: Denylist-Based
**Nanobot, PicoClaw**

- 정규식 차단
- 파일시스템 제한
- HITL 없음

### Tier 4: Minimal
**TinyClaw, OpenFang**

- 보안 메커니즘 최소

---

## 6. 강점/약점 분석 (용도별 추천)

### 6.1 엔터프라이즈 운영 (금융/의료)

**최적**: **IronClaw**
- Zero-Exposure 자격증명 관리
- WASM 이중 샌드박스
- AES-256-GCM 암호화
- 완벽한 감사 로그

**대안**: ZeroClaw (비용 통제 필요 시), OpenClaw (기능 최다 필요 시)

### 6.2 개인 고급 사용자

**최적**: **ZeroClaw**
- 비용 하드 한도 ($5/일)
- E-Stop 긴급 제어
- Soul Snapshot 자동 복원
- mtime 캐싱 (경량)

**대안**: OpenClaw (기능 선호 시)

### 6.3 사내 배포 (DevOps팀)

**최적**: **NanoClaw**
- Anthropic SDK 네이티브
- 그룹별 Docker 격리
- 시크릿 stdin 전달
- 간단한 운영

**대안**: IronClaw (높은 보안), OpenClaw (기능 최다)

### 6.4 다중채널 통합 (Slack+Telegram+Discord)

**최적**: **OpenJarvis**
- SessionIdentity 통합
- Cross-channel 세션 연속성
- RBAC 이중 구현
- Taint Tracking (유일)

### 6.5 모바일/엣지 디바이스

**최적**: **PicoClaw**
- 원자적 쓰기 (플래시 안정성)
- 경량 (10K LOC)
- 2-pass 요약
- mtime 캐싱

**대안**: Nanobot (더 작음)

### 6.6 팀 협업 (분산)

**최적**: **TinyClaw**
- 분산 액터 모델
- 팀 멘션 (@teammate)
- SQLite 메시지 큐
- 팀 크기 제한 없음

### 6.7 최대 기능 (No Limits)

**최적**: **OpenClaw**
- 50+ 브라우저 기능
- 6개 임베딩 제공자
- 4단계 기억 파이프라인
- 24개 플러그인 훅

### 6.8 MCP 표준 준수

**최적**: **OpenFang**
- 60개 빌트인 tool (string dispatch)
- HAND.toml 표준
- WASM Dual Metering
- A2A 프로토콜

### 6.9 연구 자동화

**최적**: **ZeroClaw + OpenClaw 기억 + DeepInnovator 패턴**
- ZeroClaw의 비용 한도 + E-Stop
- OpenClaw의 하이브리드 검색
- DeepInnovator의 Authenticity Discriminator
- Autoresearch의 Fixed-Budget Loop

---

## 7. 시장 트렌드 및 결론

### 7.1 5가지 수렴 추세

1. **MCP가 사실상의 표준**: 10개 중 9개가 MCP 지원. 직접 도구 개발보다 MCP 서버 통합이 대세.

2. **벡터 DB + FTS 하이브리드 검색**: Tier 1 삼총사 모두 이 패턴 도입. RRF/weighted fusion/linear fusion 3가지 알고리즘 실험 중.

3. **비용 통제의 중요성**: ZeroClaw의 $5/일 한도가 "살펴볼 가치" -> 다른 구현체도 선택적 도입 고려.

4. **자격증명 관리의 진화**:
   - Tier 4: 평문 저장
   - Tier 3: 환경변수 차단
   - Tier 2: stdin 전달 + 즉시 삭제
   - Tier 1: Zero-Exposure (IronClaw) 또는 암호화 저장

5. **Cross-Channel 세션 통합**: OpenJarvis의 SessionIdentity가 새로운 패턴. Slack+Telegram+Web 동시 사용 시대.

### 7.2 아직 미해결된 문제

**자동 컨텍스트 분리 판단**:
- "이 작업은 새 컨텍스트가 필요하다"를 프레임워크가 자동으로 판단하는 구현체 없음
- 모든 구현체가 LLM의 도구 호출 판단에 의존

**비용 인식 컨텍스트 관리**:
- 서브에이전트를 스폰할 때마다 시스템 프롬프트 반복 전송
- 5개 서브에이전트 x 10,000토큰 = 50,000토큰의 순수 오버헤드
- 최적화 구현체 없음

**기억 오염 방지의 근본 해법**:
- IronClaw의 하드코딩 보호는 특정 파일만 보호
- 나머지 기억은 LLM 판단 의존
- 할루시네이션 자동 탐지 없음 (OpenJarvis Discriminator 제외)

**연구 자동화의 실험 설계**:
- Karpathy 발견: "에이전트는 구현은 잘하지만 실험 설계는 못한다"
- DeepInnovator의 Serendipity Engine이 부분 해결
- 범용 패턴화 아직 불가능

---

### 7.3 신규 패턴 (연구 도구에서 발굴)

#### R1: Authenticity Discriminator
생성된 아이디어의 "실제성"을 LLM 판별기로 검증. 기존 9개에 없는 패턴.

#### R2: Delta Reward
절대 품질이 아닌 **상대적 개선도**를 보상. 에이전트가 반복할수록 더 나은 결과.

#### R3: Fixed-Budget Loop
시간 기반 자율 실험 (5분/실험 x 12회/시간 x 무한). keep/discard 자동 의사결정.

#### R4: Hierarchical Agent Pipeline
4-layer 계층 (분석 -> 그룹 -> 통찰 -> 합성). 각 레이어가 독립 병렬 처리 가능.

#### R5: Deep Synthesis Principle
"A+B+C 단순 조합 금지". 시너지 원리 이해 + 구체적 기술적 접근 + 실행 가능성 검증.

---

### 7.4 최종 권장사항

#### 새로운 프로젝트 시작 시

**범용 에이전트**: ZeroClaw (균형) 또는 OpenClaw (기능 최다)
**엔터프라이즈**: IronClaw
**모바일/엣지**: PicoClaw
**다중채널**: OpenJarvis
**팀 협업**: TinyClaw
**연구 자동화**: ZeroClaw + OpenClaw 기억 + DeepInnovator 패턴

#### 기존 프로젝트 업그레이드 시

1. **기억 시스템**: Tier 1로 (ZeroClaw SQLite + OpenClaw 하이브리드)
2. **보안**: Tier 1로 (암호화 또는 WASM 추가)
3. **비용 통제**: ZeroClaw 패턴 도입
4. **다중채널**: SessionIdentity 패턴 도입

#### 2026년 에이전트 스택

```
+---------------------------------------------+
|  런타임: ZeroClaw / OpenClaw               |
|  (보안 + 기억 + 비용 제한)                 |
+---------------------------------------------+
|  기억 백엔드: 하이브리드 검색              |
|  - SQLite + 벡터 임베딩                   |
|  - FTS5 + BM25 + temporal decay           |
|  - Soul Snapshot (Git 추적)               |
+---------------------------------------------+
|  도구: MCP 표준 + 네이티브 가속            |
|  - ArXiv, PubMed, Notion 등               |
|  - 브라우저 자동화 (OpenClaw 패턴)        |
+---------------------------------------------+
|  검증: Authenticity Discriminator         |
|  - 품질 자동 검증                         |
|  - Delta Reward 기반 강화                 |
+---------------------------------------------+
|  안전장치: 이중 한도                      |
|  - 시간 한도 (5분/실험)                   |
|  - 비용 한도 ($5/일)                      |
+---------------------------------------------+
```

---

## 부록 A: 10개 구현체 체크리스트

| 기능 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang | 점수 |
|------|:--------:|:-------:|:--------:|:--------:|:--------:|:--------:|:--------:|:----------:|:--------:|:----:|
| 24/7 구동 | [O] | DELTA | DELTA | [O] | [O] | DELTA | [X] | DELTA | [X] | 4/9 |
| 메신저 통합 | [O] | [O] | [O] | [O] | [O] | [O] | [O] | [O] | DELTA | 8/9 |
| 자율성 제어 | [O] | [X] | [X] | [O] | [O] | [X] | DELTA | [X] | [O] | 4/9 |
| 커넥터 다양성 | GOOD | OK | BASIC | GOOD | GOOD | OK | BASIC | OK | GOOD | 38/45 |
| 보안 등급 | 2 | 3 | 2 | 1 | 1 | 3 | 4 | 1 | 4 | Avg 2.3 |
| 기억 시스템 | 1 | 2 | 3 | 1 | 1 | 2 | 3 | 1 | 3 | Avg 1.8 |
| 세션 관리 | 1 | 2 | 1 | 1 | 2 | 2 | 2 | 1 | 3 | Avg 1.8 |
| 운영 단순도 | DELTA | [O] | [O] | DELTA | DELTA | [O] | [O] | DELTA | DELTA | 4/9 |
| **종합 평가** | **[VERY_GOOD]** | **[GOOD]** | **[GOOD]** | **[VERY_GOOD]** | **[VERY_GOOD]** | **[GOOD]** | **[FAIR]** | **[GOOD]** | **[FAIR]** | |

---

## 부록 B: 보고서 참고 자료

### 기존 5개 보고서 (Task 1-4)

1. **security_report.md** (2026-03-05)
   - 7대 보안 영역 비교
   - 4계층 보안 성숙도
   - 10개 개별 분석

2. **session_context_report.md** (2026-03-04)
   - 세션/컨텍스트 관리 3가지 아키타입
   - 5가지 설계 패턴
   - idea.md 가설 검증

3. **memory_architecture_report.md** (2026-03-05)
   - 3계층 기억 모델
   - 3가지 성숙도 분류
   - 5가지 교차 분석 패턴

4. **research_tools_report.md** (2026-03-09)
   - DeepInnovator + Autoresearch 분석
   - 5가지 신규 패턴 (R1-R5)
   - 연구 에이전트 스택 제안

5. **browser_actions_report.md** (2026-03-05, 2026-03-14 업데이트)
   - 브라우저 자동화 4개 구현체
   - 도구 아키텍처 8가지 유형
   - MCP 표준화 추세

### 신규 통합 요소 (본 리포트)

- 4계층 성숙도 아키텍처 (Tier 1-4 통합)
- Claw형 기준 평가표 (용도별 추천)
- 시장 트렌드 (5가지 수렴 추세)
- 미해결 문제 및 신규 패턴

---

## 최종 결론

10개 오픈소스 에이전트 런타임을 비교한 결과, **24시간 상주 메신저 에이전트의 요구사항은 기존 시스템과 근본적으로 다르며, 이를 충족하기 위해서는 Tier 1 수준의 보안, 기억, 세션 관리가 필수**임이 확인되었다.

단순히 "더 큰 모델"이 아니라 **다른 시스템**이 필요하다. ZeroClaw, IronClaw, OpenClaw 삼총사의 기술 선택이 완전히 다르면서도 각각 우수한 것은, 이 문제 공간이 여전히 활발하게 진화 중임을 보여준다.

연구 자동화 도구(DeepInnovator, Autoresearch)는 **이 시스템들이 놓친 새로운 패턴 5가지**를 제시한다. 2026년 에이전트 스택은 이들 패턴을 결합해야 할 것이다.

**선택의 기준은 단순하다**: 무엇이 필요한가?
- 최고 보안 -> IronClaw
- 비용 통제 -> ZeroClaw
- 최대 기능 -> OpenClaw
- 다중채널 -> OpenJarvis
- 모바일 -> PicoClaw
- 팀 협업 -> TinyClaw

---

**보고서 작성**: Compare Claws Project Team
**최종 완성**: 2026-03-14
**통합 분석 대상**: 10개 에이전트 런타임 + 2개 연구 도구 + 5개 기존 보고서
