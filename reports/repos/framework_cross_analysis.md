# 13개 Claw 서비스 종합 분석 리포트
## Compare Claws Project Final Report

> **조사 기간**: 2026-03-05 ~ 2026-03-17
> **조사 대상**: 13개 오픈소스 에이전트 런타임 프레임워크
> **조사 방법**: 8개 scientist/architect 에이전트의 병렬 심층 코드 분석
> **최종 보고서**: 5개 기존 보고서 + 2개 신규 도구 분석 + 4개 신규 프레임워크 통합

---

## Executive Summary

### 핵심 발견 4가지

1. **4계층 성숙도 아키텍처**: 13개 구현체는 보안/기억/세션 관리 복잡도에 따라 Tier 1(엔터프라이즈) ~ Tier 4(실험)로 분류된다. **Tier 1 삼총사(IronClaw, ZeroClaw, OpenClaw)의 기술 선택이 완전히 다르면서도 각각 우수하다**는 발견이 가장 중요. 신규 편입된 **OpenFang(Agent OS)**은 S등급으로 Tier 1을 압도하는 독립 카테고리를 형성한다. **NullClaw**는 Zig 정적 컴파일 + Landlock OS sandbox으로 극한 최소화 Tier 1 니치를 개척한다.

2. **24시간 상주 에이전트의 요구사항은 기존 시스템 프롬프트 기반 에이전트와 근본적으로 다르다**: 세션/컨텍스트 관리, 기억 아키텍처, 보안 경계, 도구 격리가 모두 새로운 수준의 복잡도를 요구한다. 이는 "더 큰 모델"이 아니라 "다른 시스템"을 필요로 한다. OpenFang과 OpenJarvis가 이 공간에서 새로운 기준을 제시한다.

3. **기억 아키텍처가 보안 아키텍처를 결정한다**: Tier 1 삼총사의 기억 시스템(Vector DB + FTS + 하이브리드 검색)은 모두 Tier 1 보안 패턴(암호화 + WASM + HITL)과 상관관계를 가진다. 단순한 마크다운 기반 기억(Nanobot, PicoClaw)을 가진 구현체는 보안도 최소 수준. OpenJarvis는 5중 검색 백엔드로 기억 A등급을 달성.

4. **연구 자동화와 범용 에이전트는 다른 설계 원리를 따른다**: DeepInnovator(연구 특화)와 Autoresearch(ML 실험)의 두 도구는 기존 런타임과 전혀 다른 패턴을 보여준다. Tier 1 기억 + 고정 예산 + 자동 검증이 조합되면 새로운 "연구 에이전트 스택"이 가능하다. NemoClaw는 GPU 최적화 + 4계층 샌드박스로 보안 특화 니치를 선점.

---

## 1. 목차

1. Executive Summary
2. 13개 서비스 비교 매트릭스
3. 4계층 성숙도 분류 (Tier S/1-4)
4. 서비스별 평가표 (Claw형 기준)
5. 보안 계층 분류
6. 강점/약점 분석 (용도별 추천)
7. 시장 트렌드 및 결론

---

## 2. 13개 서비스 비교 매트릭스

### 2.1 기본 정보

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang | NemoClaw | **NullClaw** |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|----------|----------|
| **언어** | TypeScript | Python | TypeScript | Rust | Rust | Go | TypeScript | Python+Rust | Rust | JS/TS/Py/Shell | Zig |
| **규모** | 430K+ LOC | 4K LOC | 20K LOC | 15K LOC | 12K LOC | 10K LOC | 5K LOC | ~50K+ LOC | 137K LOC | 25,650 LOC | 249K LOC |
| **저자** | 범용 (open) | 개인 | Anthropic | Anthropic | 개인 | 개인 | 개인 | Stanford | 개인 | NVIDIA | 개인 |
| **라이선스** | 상업/오픈 | MIT | 상업 | 상업 | MIT | MIT | MIT | 상업 | MIT | 상업 | MIT |
| **주요 용도** | 범용 에이전트 | 개인 에이전트 | 사내 배포 | 엔터프라이즈 | 개인 고급 | 모바일/엣지 | 팀 협업 | 다중채널 | Agent OS | GPU 샌드박스 | 극한 최소화/엣지 |

### 2.2 런타임 특성

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang | NemoClaw | **NullClaw** |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|----------|----------|
| **배포 방식** | 호스팅됨 | 자체 배포 | K8s/Docker | 자체 배포 | 자체 배포 | 자체 배포 | 자체 배포 | 자체 배포 | 자체 배포 | OpenClaw 플러그인 | 자체 배포 |
| **컨테이너** | Docker | [X] | Docker | Docker | [X] | [X] | [X] | [X] | [X] | NIM 컨테이너 | Docker (옵션) |
| **WASM 샌드박스** | [X] | [X] | [X] | [O] (이중) | [O] | [X] | [X] | [X] | [O] (이중 미터링) | [X] | [X] (Landlock OS) |
| **에이전트 수** | 1+ | 1 (spawn) | 팀 (Teams) | 1 | 1 | 1 | N (분산) | 1+ (9종) | 1 (멀티모델) | 1 (정책 제어) | 1+ (서브에이전트) |
| **프로세스 격리** | [X] | [X] | [O] | [O] | [O] | [X] | [X] | [X] | [O] | [O] (Landlock) | [O] (Landlock) |

### 2.3 세션/컨텍스트 관리

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang | NemoClaw | **NullClaw** |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|----------|----------|
| **세션 키 전략** | agent:<id>:<type>:<uuid> | channel:chat_id | group_folder | (user, channel, thread) | session_id (DB) | agent:id:ch:kind:peer | agent_dir | session_id + channel_ids | 3-layer context window | OpenClaw 위임 | session.zig |
| **멀티에이전트** | [O] | [O] | [O] | [O] | [X] | [X] | [O] | [O] | [O] (멀티모델) | DELTA | [O] |
| **컴팩션** | 동기 | 비동기 LLM | SDK 자동 | 선택 가능 | 동기 | 비동기 | SDK 자동 | 비동기 | 12-section 빌더 | OpenClaw 위임 | DELTA |
| **장기기억** | [O] (LanceDB) | [O] (.md) | DELTA | [O] (벡터) | [O] (SQLite) | [O] (.md) | DELTA | [O] (SQLite+5종) | [O] (KG+SQLite) | [X] | [O] (SQLite+10종) |
| **Cross-Channel** | [X] | [X] | [X] | [X] | [X] | [X] | [X] | [O] | [O] (40채널) | [X] | [X] |

### 2.4 기억 아키텍처

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang | NemoClaw | **NullClaw** |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|----------|----------|
| **저장 백엔드** | SQLite + LanceDB | 파일 (.md) | 파일 (아카이브) | PostgreSQL + libSQL | SQLite + FTS5 | 파일 (.md) | 파일 (아카이브) | SQLite/FAISS/ColBERT/BM25/RRF | SQLite BLOB (11테이블) + KG | OpenClaw 호스트 의존 | SQLite/PostgreSQL/Redis/ClickHouse |
| **임베딩** | [O] (6개) | [X] | [X] | [O] (4개) | [O] (3개) | [X] | [X] | [O] (FAISS+ColBERTv2) | [O] (중요도 스코어링) | [X] | [O] |
| **하이브리드 검색** | [O] (4단계) | [X] | [X] | [O] (RRF) | [O] (linear) | [X] | [X] | [O] (BM25+Hybrid RRF) | [O] (KG+SQLite) | [X] | [O] (가중치 설정) |
| **수명주기** | Atomic | 없음 | 없음 | 30일 | 12h | 없음 | 없음 | 24h | Importance scoring | 없음 | 설정 가능 |
| **Soul Snapshot** | [X] | [X] | [X] | [X] | [O] | [X] | [X] | [X] | [O] (영속 스냅샷) | [X] | [X] |

### 2.5 보안/권한

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang | NemoClaw | **NullClaw** |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|----------|----------|
| **암호화 볼트** | [X] | [X] | [X] | [O] (AES) | [O] (ChaCha) | [X] | [X] | [X] | [O] (18 capability types) | [X] | [O] (ChaCha20) |
| **HITL** | [O] | [X] | [X] | [O] | [O] | [X] | DELTA | [X] | [O] | [O] (operator 승인) | [X] |
| **Prompt Injection** | DELTA | [X] | [X] | [O] (4중) | [O] (6패턴) | [X] | [X] | [O] (Scanner+Taint) | [O] (Taint tracking) | DELTA | [X] |
| **Taint Tracking** | [X] | [X] | [X] | [X] | [X] | [X] | [X] | [O] | [O] (16-layer) | [X] | [X] |
| **비용 제한** | DELTA | [X] | [X] | DELTA | [O] ($5) | [X] | [X] | [X] | [X] | [X] | [X] |

### 2.6 도구/액션 시스템

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang | NemoClaw | **NullClaw** |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|----------|----------|
| **도구 유형** | TS Plugin | Python ABC | SKILL.md | Rust Trait | Rust Trait | Go Iface | CLI | Python Class | WASM sandbox | OpenClaw 플러그인 명령 | Zig vtable |
| **빌트인 도구 수** | 50+ | 10+ | 15+ | 20+ | 16+ | 15+ | 5+ | 24+ | 60 | 10 정책 프리셋 | 35+ |
| **MCP 지원** | [O] | [O] | [O] | [O] | [O] | [O] | [O] | [O] (어댑터) | [O] (양방향) | [X] (OpenClaw 위임) | [O] |
| **브라우저** | [O] (50+) | [X] | DELTA | DELTA | [O] (16) | [X] | [X] | [O] (axtree) | [O] (CDP 네이티브, 50+) | [X] | [X] |
| **SSRF 방지** | [O] | [X] | [X] | [O] | [O] | [X] | [X] | DELTA | [O] | [O] (network deny-by-default) | [X] |

### 2.7 메신저 인터페이스

| 항목 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang | NemoClaw | **NullClaw** |
|------|----------|---------|----------|----------|----------|----------|----------|-----------|----------|----------|----------|
| **지원 범위** | Discord+Slack | Telegram | Discord | Discord | Telegram | Telegram | 팀 | 24채널+ | 40채널 어댑터 | Telegram 브릿지 | 19채널 (Signal+Nostr+Matrix) |
| **채널 정책** | [O] | [O] | [O] | [O] | DELTA | DELTA | DELTA | [O] | [O] (채널별 모델 오버라이드) | DELTA | [O] (deny-all 기본) |
| **다중채널 세션** | [X] | [X] | [X] | [X] | [X] | [X] | [X] | [O] | [O] | [X] | [X] |

---

## 3. 4계층 성숙도 분류

### Tier S: Agent OS (신규 최상위)

**구현체**: OpenFang

**공통 특성**:
- 16-layer 보안 아키텍처 (Taint tracking, WASM dual metering, 18 capability types)
- SQLite BLOB + Knowledge Graph + Importance scoring (v7 스키마, 11 테이블)
- 60개 빌트인 도구 + WASM 샌드박스 실행 + MCP 양방향 + A2A 프로토콜
- Native CDP 브라우저 (50+ 기능)
- 40채널 어댑터 (채널별 모델 오버라이드)
- 24/7 자율 실행 + Soul snapshot 영속성
- 137K LOC, 14 crates, tokio async

**평가**: 기존 Tier 1 삼총사를 능가하는 독립 카테고리. 단, 개인 개발자 운영 기반이라는 생태계 리스크 존재.

---

### Tier 1: Defense-in-Depth (엔터프라이즈)

**구현체**: IronClaw, ZeroClaw, OpenClaw, OpenJarvis, NullClaw

**공통 특성**:
- 암호화 볼트 또는 암호화 저장 (IronClaw AES-256-GCM, ZeroClaw ChaCha20-Poly1305, OpenClaw 평문, NullClaw ChaCha20-Poly1305)
- Tier 1 기억 시스템 (벡터 DB + FTS + 하이브리드 검색)
- 다층 보안 방어 (WASM + Docker + 정책 엔진 또는 OS 레벨 격리)
- HITL 승인 시스템 (완전 또는 부분)
- Prompt Injection 전용 방어 레이어

**차이점**:

| 측면 | IronClaw | ZeroClaw | OpenClaw | OpenJarvis | NullClaw |
|------|----------|----------|----------|-----------|----------|
| **철학** | Zero-Exposure (컨테이너 자격증명 미노출) | 정책 기반 세밀 제어 | 실용적 다층 방어 | Trace-driven 학습 루프 | 정적 컴파일 + OS 샌드박스 극한 최소화 |
| **기억** | pgvector RRF | SQLite+vec 선형 fusion | LanceDB 4단계 파이프라인 | SQLite/FAISS/ColBERT/BM25/RRF 5중 | SQLite/PostgreSQL/Redis/ClickHouse 10종 |
| **비용** | 추적만 (무한) | $5/일 하드 한도 | 추적만 (무한) | 속도 제한 | 없음 |
| **WASM** | 이중 (도구+실행) | 단일 (실행용) | 없음 | 없음 | 없음 (Landlock 대체) |
| **특수 기능** | Zero-Exposure 프록시 | E-Stop + Soul Snapshot | 50+ 브라우저 기능 | Taint Tracking + 다중채널 SessionIdentity | 678 KB binary + <2ms 기동 + WASI 타겟 |

> **NullClaw**: ChaCha20-Poly1305 + Landlock OS sandbox. WASM 없이 커널 레벨 격리로 Tier 1 달성.

**선택 기준**:
- **IronClaw**: 가장 높은 보안 요구사항 (금융/의료)
- **ZeroClaw**: 비용 인식 + 모바일 환경 + 스냅샷 복원 필요
- **OpenClaw**: 최대 기능 + 생태계 + 상용 지원
- **OpenJarvis**: 다중채널 통합 + Trace-driven 학습 + 오프라인 우선
- **NullClaw**: 극한 경량 + 엣지/IoT + WASI 이식성 필요

---

### Tier 1.5: Security-Specialized (보안 특화)

**구현체**: NemoClaw

**특성**:
- 4계층 샌드박스 보안 (network deny-by-default + filesystem Landlock + process seccomp + inference gateway)
- Operator 승인 플로우 (알 수 없는 액션에 대한 HITL)
- GPU 최적화 NIM 컨테이너 + blueprint 버전 관리
- 10 정책 프리셋 커넥터

**한계**:
- 독립 실행 불가 (OpenClaw 플러그인으로만 동작)
- 기억 시스템 없음 (D등급)
- Telegram 브릿지만 지원 (네이티브 메신저 제한)

**평가**: 보안/GPU 컴퓨팅에서 탁월하나 단독 Agent 프레임워크로는 미흡. OpenClaw와 함께 사용 시 Tier 1급 보안 달성.

---

### Tier 2: Container-First (중급)

**구현체**: NanoClaw

**공통 특성**:
- Docker/컨테이너 격리
- 부분적 기억 시스템 (자동 로딩 없거나 제한적)
- 도구 허용목록 + 환경변수 격리
- HITL 없음 (자율 실행)

**NanoClaw 특이성**:
- 그룹별 독립 Docker 컨테이너 (강한 물리적 격리)
- 시크릿 stdin 전달 + 즉시 삭제 (메모리 보안)
- SDK Agent Teams (네이티브 Anthropic 통합)

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

**구현체**: TinyClaw

**TinyClaw**:
- 보안 메커니즘 최소
- 분산 액터 모델 (독특)
- CORS 와일드카드
- 기억 미구현

---

## 4. Claw형 기준 평가표

### 4.1 24/7 상시 구동 (가능 여부)

| 구현체 | 평가 | 사유 |
|--------|------|------|
| **OpenFang** | [O] (최적) | 24/7 자율 실행 + Soul snapshot 영속 + 40채널 상시 대기 |
| **ZeroClaw** | [O] (우수) | 비용 하드 한도 + E-Stop으로 자동 제어 |
| **IronClaw** | [O] (우수) | 보안 + 비용 추적, 한도는 없지만 로깅 |
| **OpenClaw** | [O] (우수) | 기능 최다 + 비용 추적, 한도 자동화 권장 |
| **NemoClaw** | [O] (특화) | 상시 샌드박스 에이전트 설계, OpenClaw 필요 |
| **OpenJarvis** | DELTA | 다중채널 우수, OperativeAgent + 스케줄러 지원 |
| **NanoClaw** | DELTA | 그룹별 격리 우수, 비용 제한 없음 |
| **PicoClaw** | DELTA | 경량 우수, 기억 수명주기 없음 |
| **Nanobot** | DELTA | 단순하지만 기억 무한 성장 |
| **TinyClaw** | [X] | 보안 최소 |
| **NullClaw** | [O] (우수) | heartbeat.zig + Landlock 격리 + ChaCha20 암호화 |

### 4.2 메신저 인터페이스 지원

| 구현체 | Discord | Slack | Telegram | 다중채널 | 특징 |
|--------|---------|-------|----------|---------|------|
| **OpenFang** | [O] | [O] | [O] | [O] (40채널) | **최고의 채널 다양성** + 채널별 모델 오버라이드 |
| **OpenJarvis** | [O] | [O] | [O] | [O] (24채널+) | **최고의 세션 통합** (SessionIdentity) |
| **OpenClaw** | [O] | [O] | [X] | DELTA | 가장 완성도 높음 |
| **TinyClaw** | [O] | DELTA | DELTA | [O] | 팀 협업 우선 |
| **Nanobot** | [X] | [X] | [O] | [X] | 텔레그램 전문 |
| **NanoClaw** | [O] | [X] | [X] | [X] | Discord 전용 |
| **IronClaw** | [O] | [X] | [X] | [X] | Discord 전용 |
| **ZeroClaw** | [X] | [X] | [O] | [X] | 텔레그램 전용 |
| **PicoClaw** | [X] | [X] | [O] | [X] | 텔레그램 전용 |
| **NemoClaw** | [X] | [X] | [O] | [X] | Telegram 브릿지 전용 |
| **NullClaw** | DELTA | DELTA | [O] | DELTA | Signal+Nostr+Matrix 포함 19채널 |

### 4.3 자율성 수준 (권한 모델)

| 구현체 | ReadOnly | Supervised | Full | 특징 |
|--------|----------|-----------|------|------|
| **ZeroClaw** | [O] | [O] | [O] | 3단계 AutonomyLevel + E-Stop |
| **OpenFang** | [O] | [O] | [O] | 18 capability types + WASM 이중 미터링 |
| **IronClaw** | DELTA | [O] | [O] | 승인 요청 + 자율 잡 |
| **NemoClaw** | [O] | [O] | DELTA | operator 승인 플로우 + 10 정책 프리셋 |
| **OpenClaw** | [X] | [O] | [O] | 실행 승인 요청 |
| **OpenJarvis** | [X] | [X] | [O] | 9 agent types, RBAC 추적 |
| **PicoClaw** | [X] | [X] | [O] | 자율 실행 |
| **Nanobot** | [X] | [X] | [O] | 자율 실행 |
| **NanoClaw** | [X] | [X] | [O] | 자율 실행 |
| **TinyClaw** | DELTA | DELTA | [O] | Pairing (채널 접근만) |
| **NullClaw** | [X] | [X] | [O] | vtable sandbox 설정 가능 |

### 4.4 커넥터 수 및 다양성

| 구현체 | MCP | 네이티브 | 브라우저 | 총합 | 평가 |
|--------|-----|----------|---------|------|------|
| **OpenClaw** | [O] (6+) | 50+ | 50+ | 100+ | [TOP] |
| **OpenFang** | [O] (양방향+A2A) | 60+ | 50+ (CDP) | 110+ | [TOP] |
| **ZeroClaw** | [O] | 16+ | 16 | 30+ | [GOOD] |
| **IronClaw** | [O] | 20+ | 테스트용 | 20+ | [GOOD] |
| **OpenJarvis** | [O] (어댑터) | 24+ | [O] (axtree) | 25+ | [GOOD] |
| **Nanobot** | [O] | 10+ | [X] | 10+ | [OK] |
| **PicoClaw** | [O] | 15+ | [X] | 15+ | [OK] |
| **NanoClaw** | [O] | X/Twitter | X/Twitter | 5+ | [BASIC] |
| **TinyClaw** | [O] | 5+ | [X] | 5+ | [BASIC] |
| **NemoClaw** | [X] | 10 프리셋 | [X] | 10+ | [BASIC] |
| **NullClaw** | [O] | 35+ | [X] | 35+ | [GOOD] |

---

## 5. 보안 Tier 분류

기존 security_report.md (2026-03-05)의 7대 보안 영역 분석 + 신규 3개 프레임워크 통합:

### Tier S: Layered Agent OS Security
**OpenFang**

| 보안 영역 | OpenFang |
|---------|---------|
| 권한/인가 | 18 capability types, 16-layer 아키텍처 |
| 자격증명 | Taint tracking + capability scope |
| 샌드박싱 | WASM dual metering (도구+실행) |
| 도구 실행 | WASM sandbox isolation |
| HITL | 도구별 승인 내장 |
| 비용 제한 | 없음 (자체 한도 설정 필요) |
| Prompt Injection | Taint tracking 전방위 |

### Tier 1: Defense-in-Depth
**IronClaw, ZeroClaw, OpenJarvis, NullClaw**

| 보안 영역 | IronClaw | ZeroClaw | OpenJarvis | NullClaw |
|---------|----------|----------|-----------|----------|
| 권한/인가 | Bearer + Skills | 3단계 AutonomyLevel | RBAC 10종 (이중) | deny-all + workspace scoping |
| 자격증명 | AES-256-GCM | ChaCha20-Poly1305 | Taint Tracking | ChaCha20-Poly1305 |
| 샌드박싱 | WASM + Docker | WASM + 다중 | subprocess + SSRF | Landlock/Firejail/Bubblewrap/Docker |
| 도구 실행 | 프록시 경유 | 5중 방어 | Taint 기반 차단 | vtable sandbox |
| HITL | 도구별 승인 | E-Stop (4단계) | 사후 추적 | [X] |
| 비용 제한 | 기록 (무한) | $5/일 하드 | 속도 제한 | [X] |
| Prompt Injection | SafetyLayer 4중 | PromptGuard 6패턴 | Scanner + Taint | [X] |

### Tier 1.5: Security-Specialized
**NemoClaw**

| 보안 영역 | NemoClaw |
|---------|---------|
| 권한/인가 | binary-scoped rules + 10 정책 프리셋 |
| 자격증명 | OpenClaw 위임 |
| 샌드박싱 | 4계층 (network Landlock + filesystem Landlock + seccomp + inference gateway) |
| 도구 실행 | operator 승인 플로우 |
| HITL | [O] (unknown action 승인) |
| 비용 제한 | [X] |
| Prompt Injection | DELTA |

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
**TinyClaw**

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

**신규 옵션**: NemoClaw + OpenClaw 조합 (GPU 추론 + 4계층 샌드박스 필요 시)

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

**최적 (세션 통합)**: **OpenJarvis**
- SessionIdentity 통합 (Slack+Telegram+Web)
- Cross-channel 세션 연속성
- RBAC 이중 구현
- Taint Tracking (유일)
- Trace-driven 학습 루프

**최적 (채널 다양성)**: **OpenFang**
- 40채널 어댑터 (최다)
- 채널별 모델 오버라이드
- 24/7 자율 구동

### 6.5 모바일/엣지 디바이스

**신규 최적 (극한 최소화)**: **NullClaw** — 678 KB / <2ms / $5 보드

**최적 (기능 균형)**: **PicoClaw**
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

**최적**: **OpenFang**
- 60+ 빌트인 도구 + WASM sandbox
- CDP 네이티브 브라우저 50+ 기능
- 40채널 어댑터
- A2A + MCP 양방향
- Soul snapshot 영속성

**차선**: **OpenClaw**
- 50+ 브라우저 기능
- 6개 임베딩 제공자
- 4단계 기억 파이프라인
- 24개 플러그인 훅

### 6.8 GPU/고성능 추론

**최적**: **NemoClaw + OpenClaw**
- GPU 최적화 NIM 컨테이너
- Blueprint 버전 관리
- 4계층 샌드박스 보안
- migration 스냅샷 (상태 이전)

### 6.9 오프라인/연구 특화

**최적**: **OpenJarvis**
- 오프라인 우선 설계
- Hardware-aware engine selection
- 5중 검색 백엔드 (ColBERTv2 + BM25 + FAISS + SQLite + Hybrid RRF)
- Stanford 연구 기반
- Trace-driven 학습 루프

### 6.10 연구 자동화

**최적**: **ZeroClaw + OpenClaw 기억 + DeepInnovator 패턴**
- ZeroClaw의 비용 한도 + E-Stop
- OpenClaw의 하이브리드 검색
- DeepInnovator의 Authenticity Discriminator
- Autoresearch의 Fixed-Budget Loop

---

## 7. 시장 트렌드 및 결론

### 7.1 6가지 수렴 추세

1. **MCP가 사실상의 표준**: 13개 중 11개가 MCP 지원 (NemoClaw와 TinyClaw만 미지원). 직접 도구 개발보다 MCP 서버 통합이 대세. OpenFang의 MCP 양방향 + A2A 프로토콜이 차세대 패턴.

2. **벡터 DB + FTS 하이브리드 검색**: Tier 1 삼총사 모두 이 패턴 도입. RRF/weighted fusion/linear fusion 3가지 알고리즘 실험 중. OpenJarvis의 5중 백엔드(ColBERT+BM25+FAISS+SQLite+RRF)가 최신 최고 수준.

3. **비용 통제의 중요성**: ZeroClaw의 $5/일 한도가 "살펴볼 가치" -> 다른 구현체도 선택적 도입 고려. OpenFang과 OpenJarvis는 아직 미도입.

4. **자격증명 관리의 진화**:
   - Tier 4: 평문 저장
   - Tier 3: 환경변수 차단
   - Tier 2: stdin 전달 + 즉시 삭제
   - Tier 1: Zero-Exposure (IronClaw) 또는 암호화 저장
   - Tier S: Taint tracking + capability scope (OpenFang)

5. **Cross-Channel 세션 통합**: OpenJarvis의 SessionIdentity가 새로운 패턴. OpenFang의 40채널 + 채널별 모델 오버라이드가 한 단계 위. Slack+Telegram+Web 동시 사용 시대가 본격화.

6. **샌드박스 계층화**: NemoClaw의 4계층 샌드박스(network + filesystem Landlock + seccomp + inference gateway)와 OpenFang의 WASM dual metering이 새로운 보안 패턴. 단일 레이어 샌드박스는 더 이상 Tier 1이 될 수 없음.

### 7.2 아직 미해결된 문제

**자동 컨텍스트 분리 판단**:
- "이 작업은 새 컨텍스트가 필요하다"를 프레임워크가 자동으로 판단하는 구현체 없음
- 모든 구현체가 LLM의 도구 호출 판단에 의존

**비용 인식 컨텍스트 관리**:
- 서브에이전트를 스폰할 때마다 시스템 프롬프트 반복 전송
- 5개 서브에이전트 x 10,000토큰 = 50,000토큰의 순수 오버헤드
- 최적화 구현체 없음 (OpenFang의 12-section 빌더가 가장 근접)

**기억 오염 방지의 근본 해법**:
- IronClaw의 하드코딩 보호는 특정 파일만 보호
- 나머지 기억은 LLM 판단 의존
- 할루시네이션 자동 탐지 없음 (OpenJarvis Discriminator + OpenFang Taint 제외)

**연구 자동화의 실험 설계**:
- Karpathy 발견: "에이전트는 구현은 잘하지만 실험 설계는 못한다"
- DeepInnovator의 Serendipity Engine이 부분 해결
- 범용 패턴화 아직 불가능

---

### 7.3 신규 패턴 (연구 도구에서 발굴)

#### R1: Authenticity Discriminator
생성된 아이디어의 "실제성"을 LLM 판별기로 검증. 기존 13개에 없는 패턴.

#### R2: Delta Reward
절대 품질이 아닌 **상대적 개선도**를 보상. 에이전트가 반복할수록 더 나은 결과.

#### R3: Fixed-Budget Loop
시간 기반 자율 실험 (5분/실험 x 12회/시간 x 무한). keep/discard 자동 의사결정.

#### R4: Hierarchical Agent Pipeline
4-layer 계층 (분석 -> 그룹 -> 통찰 -> 합성). 각 레이어가 독립 병렬 처리 가능.

#### R5: Deep Synthesis Principle
"A+B+C 단순 조합 금지". 시너지 원리 이해 + 구체적 기술적 접근 + 실행 가능성 검증.

#### R6: Trace-Driven Learning Loop (신규 - OpenJarvis)
실행 trace를 기반으로 에이전트가 자신의 행동 패턴을 학습. Stanford 연구 기반.

#### R7: Hardware-Aware Engine Selection (신규 - OpenJarvis)
가용 하드웨어(GPU/CPU/메모리)에 따라 LLM 엔진을 자동 선택. 오프라인 우선 설계와 결합.

#### R8: WASM Dual Metering (신규 - OpenFang)
도구 실행과 추론을 별도로 미터링하는 이중 계층 WASM 샌드박스. 정밀 비용/보안 제어.

#### R9: Blueprint Versioning (신규 - NemoClaw)
에이전트 정책과 상태를 버전 관리된 blueprint로 표현. migration 스냅샷으로 상태 이전.

#### R15: 정적 컴파일 = 공급망 공격 표면 제거 (NullClaw)
libc만 의존하는 Zig 정적 바이너리는 동적 링킹에서 오는 supply chain attack 표면 자체를 차단. WASM sandbox와 다른 접근 — 코드가 실행되기 전부터 보안.

#### R16: WASI = 에이전트 이식성 표준 (NullClaw)
WebAssembly System Interface 타겟으로 동일 바이너리가 브라우저/엣지/서버에서 실행. 에이전트 배포 표준화 가능성 제시.

---

### 7.4 최종 권장사항

#### 새로운 프로젝트 시작 시

**최대 기능 + Agent OS**: OpenFang
**범용 에이전트**: ZeroClaw (균형) 또는 OpenClaw (기능 최다)
**엔터프라이즈**: IronClaw
**모바일/엣지**: PicoClaw
**극한 경량/엣지 (배터리/IoT)**: NullClaw (678 KB / <2ms / Zig)
**다중채널 세션**: OpenJarvis
**팀 협업**: TinyClaw
**GPU/보안 샌드박스**: NemoClaw + OpenClaw
**연구 자동화**: ZeroClaw + OpenClaw 기억 + DeepInnovator 패턴

#### 기존 프로젝트 업그레이드 시

1. **기억 시스템**: Tier 1로 (ZeroClaw SQLite + OpenClaw 하이브리드, 또는 OpenJarvis 5중 백엔드)
2. **보안**: Tier 1로 (암호화 또는 WASM 추가); GPU 환경이면 NemoClaw 플러그인 추가
3. **비용 통제**: ZeroClaw 패턴 도입
4. **다중채널**: SessionIdentity 패턴 도입 (OpenJarvis) 또는 40채널 어댑터 (OpenFang)

#### 2026년 에이전트 스택

```
+---------------------------------------------+
|  런타임: OpenFang / ZeroClaw / OpenClaw     |
|  (보안 + 기억 + 비용 제한)                 |
+---------------------------------------------+
|  기억 백엔드: 하이브리드 검색              |
|  - SQLite + 벡터 임베딩 (OpenJarvis 5중)  |
|  - FTS5 + BM25 + temporal decay           |
|  - Soul Snapshot (Git 추적, ZeroClaw)      |
|  - Knowledge Graph (OpenFang)              |
+---------------------------------------------+
|  도구: MCP 표준 + A2A + 네이티브 가속      |
|  - ArXiv, PubMed, Notion 등               |
|  - 브라우저 자동화 (OpenClaw/OpenFang)    |
|  - WASM Dual Metering (OpenFang)          |
+---------------------------------------------+
|  보안: 다계층 샌드박스                    |
|  - WASM Dual Metering (OpenFang)          |
|  - 4계층 OS 샌드박스 (NemoClaw)           |
|  - Taint Tracking (OpenJarvis/OpenFang)   |
+---------------------------------------------+
|  검증: Authenticity Discriminator         |
|  - 품질 자동 검증                         |
|  - Delta Reward 기반 강화                 |
|  - Trace-driven 학습 (OpenJarvis)         |
+---------------------------------------------+
|  안전장치: 이중 한도                      |
|  - 시간 한도 (5분/실험)                   |
|  - 비용 한도 ($5/일, ZeroClaw 패턴)       |
|  - Operator 승인 (NemoClaw 패턴)          |
+---------------------------------------------+
```

---

## 부록 A: 13개 구현체 체크리스트

| 기능 | OpenClaw | Nanobot | NanoClaw | IronClaw | ZeroClaw | PicoClaw | TinyClaw | OpenJarvis | OpenFang | NemoClaw | **NullClaw** | 점수 |
|------|:--------:|:-------:|:--------:|:--------:|:--------:|:--------:|:--------:|:----------:|:--------:|:--------:|:--------:|:----:|
| 24/7 구동 | [O] | DELTA | DELTA | [O] | [O] | DELTA | [X] | DELTA | [O] | [O] | [O] | 6/11 |
| 메신저 통합 | [O] | [O] | [O] | [O] | [O] | [O] | [O] | [O] | [O] | DELTA | [O] | 10/11 |
| 자율성 제어 | [O] | [X] | [X] | [O] | [O] | [X] | DELTA | [X] | [O] | [O] | [X] | 5/11 |
| 커넥터 다양성 | TOP | OK | BASIC | GOOD | GOOD | OK | BASIC | GOOD | TOP | BASIC | GOOD | - |
| 보안 등급 | 2 | 3 | 2 | 1 | 1 | 3 | 4 | 1 | S | 1.5 | 1 | Avg 1.86 |
| 기억 시스템 | 1 | 3 | 3 | 1 | 1 | 3 | 4 | 1 | S | 4 | 1 | Avg 2.1 |
| 세션 관리 | 1 | 2 | 1 | 1 | 2 | 2 | 2 | 1 | 1 | 2 | 2 | Avg 1.5 |
| 운영 단순도 | DELTA | [O] | [O] | DELTA | DELTA | [O] | [O] | DELTA | DELTA | DELTA | [O] | 5/11 |
| **종합 평가** | **[VERY_GOOD]** | **[GOOD]** | **[GOOD]** | **[VERY_GOOD]** | **[VERY_GOOD]** | **[GOOD]** | **[FAIR]** | **[VERY_GOOD]** | **[EXCELLENT]** | **[GOOD+]** | **[VERY_GOOD]** | |

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

- Tier S 신설 (OpenFang Agent OS)
- Tier 1.5 신설 (NemoClaw 보안 특화)
- OpenJarvis Tier 1 상향 (5중 검색 백엔드, Trace-driven 학습)
- 4계층 성숙도 아키텍처 → Tier S/1/1.5/2/3/4 (Tier S/1-4 통합)
- Claw형 기준 평가표 확장 (용도 6.8~6.9 신규)
- 시장 트렌드 6번째 항목 추가 (샌드박스 계층화)
- 신규 패턴 R6~R9 추가
- 미해결 문제 업데이트 (OpenJarvis/OpenFang 부분 해결 반영)
- NullClaw 추가 (Zig, Tier 1, §11, R15~R16 패턴 신규)

---

## 최종 결론

13개 오픈소스 에이전트 런타임을 비교한 결과, **24시간 상주 메신저 에이전트의 요구사항은 기존 시스템과 근본적으로 다르며, 이를 충족하기 위해서는 Tier 1 수준의 보안, 기억, 세션 관리가 필수**임이 확인되었다.

신규 분석된 4개 프레임워크는 각기 다른 니치를 선점한다:
- **OpenFang**: Agent OS로서 모든 카테고리를 선도 (S등급). 137K LOC + 16-layer 보안 + 40채널 + Soul snapshot이 조합된 완성형 플랫폼.
- **OpenJarvis**: 다중채널 통합의 최고 수준 (Tier 1 상향). 5중 검색 백엔드 + Trace-driven 학습 + 오프라인 우선 + Stanford 연구 기반.
- **NemoClaw**: GPU 보안 샌드박스 특화 (Tier 1.5). 4계층 OS 샌드박스 + NIM 컨테이너로 엔터프라이즈 GPU 추론의 새 기준.
- **NullClaw**: 극한 최소화 특화 (Tier 1). Zig 정적 바이너리 678 KB + <2ms 기동 + ChaCha20-Poly1305 + Landlock OS sandbox. WASM 없이 커널 레벨 격리로 Tier 1 달성. WASI 타겟으로 에이전트 이식성 표준 가능성 제시.

단순히 "더 큰 모델"이 아니라 **다른 시스템**이 필요하다. ZeroClaw, IronClaw, OpenClaw 삼총사의 기술 선택이 완전히 다르면서도 각각 우수한 것은, 이 문제 공간이 여전히 활발하게 진화 중임을 보여준다. OpenFang의 등장은 이 경쟁 구도에 새로운 차원을 추가한다.

**선택의 기준은 단순하다**: 무엇이 필요한가?
- 최고 보안 -> IronClaw (엔터프라이즈) / NemoClaw+OpenClaw (GPU 환경)
- 비용 통제 -> ZeroClaw
- 최대 기능 -> OpenFang (Agent OS) / OpenClaw (생태계)
- 다중채널 세션 -> OpenJarvis
- 채널 다양성 -> OpenFang
- 모바일 -> PicoClaw
- 극한 경량/엣지 -> NullClaw (678 KB / <2ms / Zig)
- 팀 협업 -> TinyClaw
- 오프라인/연구 -> OpenJarvis

---

**보고서 작성**: Compare Claws Project Team
**최종 완성**: 2026-03-17
**통합 분석 대상**: 13개 에이전트 런타임 + 2개 연구 도구 + 5개 기존 보고서
