# 중국 AI 에이전트 서비스 보안 분석 - 문서 인덱스

**분석 완료 일자**: 2026-03-14

---

## 빠른 시작 (5분)

이 섹션부터 시작하세요.

```
1. 이 문서 (INDEX) 읽기 → 2분
2. SECURITY_ANALYSIS_SUMMARY.md의 "빠른 참조" 섹션 읽기 → 3분
3. 관심 서비스 찾기 및 추천 시나리오 확인 → 1분
```

---

## 문서 선택 가이드

### A. 경영진 / 의사결정자

**읽을 문서**:
1. SECURITY_ANALYSIS_SUMMARY.md (10-15분)
   - "빠른 참조: 보안 Tier"
   - "5개 보안 영역 성적표"
   - "각 서비스별 추천 사용 시나리오"
   - "우선순위별 개선 로드맵"

**걸리는 시간**: 15분
**결과**: Tier 분류 및 사용 권고 파악

---

### B. 보안 담당자 / 리더

**읽을 문서**:
1. SECURITY_ANALYSIS_SUMMARY.md (15분)
   - 전체 읽기

2. README_CHINESE_AI_SECURITY.md (20분)
   - "주요 발견 사항" (6가지 갭)
   - "실행 체크리스트"
   - "FAQ"

**걸리는 시간**: 35분
**결과**: 우선순위, 갭 분석, 구현 로드맵 이해

---

### C. 보안 엔지니어 / 구현팀

**읽을 문서**:
1. README_CHINESE_AI_SECURITY.md (20분)
   - 전체 읽기 (특히 "코드 예제" 섹션)

2. chinese_ai_services_security_analysis.md (60분)
   - "Executive Summary"
   - "5대 보안 영역 비교 매트릭스" (상세)
   - 관심 서비스의 "서비스별 상세 분석"
   - "핵심 보안 갭 분석"
   - "권장사항" (코드 샘플)

**걸리는 시간**: 80분
**결과**: 구현 가능한 상세 기술 권고, 코드 샘플

---

### D. 아키텍처 리뷰어

**읽을 문서**:
1. chinese_ai_services_security_analysis.md (전체 60분)
   - 특히 주목:
     - "보안 Tier 분류"
     - "핵심 보안 갭 분석" (6가지)
     - "기존 Claw와의 비교"

2. 기존 security_report.md (참고)
   - IronClaw/ZeroClaw 수준의 구현 사항 이해

**걸리는 시간**: 70분
**결과**: 장기 기술 로드맵, 아키텍처 개선안

---

## 문서 상세 정보

### 1. README_CHINESE_AI_SECURITY.md

**크기**: 9.5KB
**줄 수**: ~330줄
**섹션**: 37개

**주요 섹션**:
- 개요
- 6개 서비스 빠른 분류 (표)
- 실행 체크리스트 (즉시/단기/중기/장기)
- 주요 발견 사항 6가지 (심각도 포함)
- 코드 예제 (Rust, Python, TypeScript)
- 기존 Claw와의 비교
- FAQ
- 더 읽을 자료
- 시작하기

**추천 대상**: 모든 사용자 (진입점)

**목표**:
- 전체 상황 이해
- 서비스 선택 가이드
- 우선순위 파악
- 첫 구현 단계

---

### 2. SECURITY_ANALYSIS_SUMMARY.md

**크기**: 7.5KB
**줄 수**: ~238줄
**섹션**: 29개

**주요 섹션**:
- Executive Summary (표)
- 빠른 참조: 보안 Tier
- 5대 보안 영역 성적표 (시각화)
- 핵심 보안 갭 (6가지)
- 우선순위별 개선 로드맵
- 각 서비스별 추천 사용 시나리오
- 기존 Claw 프레임워크와의 격차
- 실행 가능한 코드 샘플 (3가지)
- 비교 표

**추천 대상**: 경영진, 의사결정자, 보안 담당자

**목표**:
- 빠른 이해
- Tier 분류
- 의사결정 지원
- 우선순위 수립

---

### 3. chinese_ai_services_security_analysis.md

**크기**: 22KB
**줄 수**: ~661줄
**섹션**: 43개

**주요 섹션**:
- Executive Summary (상세)
- 5대 보안 영역 비교 매트릭스
  - 2.1 자격증명 관리 (6개 서비스 비교)
  - 2.2 샌드박싱 (Docker, WASM, OS 격리)
  - 2.3 권한 제어 (도구 수준, 데이터 흐름)
  - 2.4 프롬프트 인젝션 방어
  - 2.5 Human-in-the-Loop
- 서비스별 상세 분석 (3.1-3.6)
  - Kimi Claw
  - Z.ai OpenClaw
  - Alibaba OpenClaw
  - Baidu DuClaw
  - Zhipu AutoClaw
  - OpenClawD
- 보안 Tier 분류 (Tier 1-4 상세 정의)
- 핵심 보안 갭 분석 (6가지, 심각도 + 영향)
- 권장사항 (우선순위 1-6, 코드 포함)
- 기존 Claw와의 비교 표

**추천 대상**: 보안 엔지니어, 아키텍트, 구현팀

**목표**:
- 상세한 기술 분석
- 구현 가능한 권고
- 코드 샘플 제공
- 장기 아키텍처 지원

---

## 6개 서비스 분류표

| 서비스 | Tier | 추천 용도 | 주요 강점 | 주요 약점 |
|--------|------|---------|---------|---------|
| **Baidu DuClaw** | 2-3 | 엔터프라이즈 | 암호화, HITL, 위험도 분류 | WASM 미완, 비용 한도 |
| **Alibaba OpenClaw** | 2 | 팀 협업 | 채널별 정책, Docker | 암호화 선택적 |
| **Z.ai OpenClaw** | 2 | 도구 제어 | 화이트리스트 | 암호화 없음 |
| **OpenClawD** | 2-3 | 팀 + 감사 | RBAC, 감사 로깅 | 암호화 부분적 |
| **Kimi Claw** | 3 | 개인 프로토 | 경량 | 보안 불충분 |
| **Zhipu AutoClaw** | 3 | 신뢰 환경 | 기본 ACL | 보안 불충분 |

---

## 6가지 핵심 보안 갭

| 갭 | 심각도 | 현재 상태 | 시간 |
|----|--------|---------|------|
| 암호화 자격증명 저장소 | CRITICAL | Baidu만 | 1주일 |
| 프롬프트 인젝션 탐지 | HIGH | Baidu 시작 | 1-2주 |
| Taint Tracking | HIGH | 전무 | 4-8주 |
| HITL 정교함 | HIGH | 부분 구현 | 2-4주 |
| WASM 샌드박스 | HIGH | Baidu 평가 | 8-16주 |
| 비용 하드 한도 | MEDIUM | 전무 | 2주 |

---

## 읽기 순서 권장안

### 시나리오 1: 빠른 의사결정 (15분)

```
1. 이 INDEX 읽기 (2분)
2. SECURITY_ANALYSIS_SUMMARY.md - "빠른 참조" (3분)
3. SECURITY_ANALYSIS_SUMMARY.md - "각 서비스별 추천 사용 시나리오" (5분)
4. SECURITY_ANALYSIS_SUMMARY.md - "우선순위별 개선 로드맵" (5분)
```

**결과**: Tier 분류 및 서비스 선택 가능

---

### 시나리오 2: 상세 이해 (80분)

```
1. 이 INDEX 읽기 (3분)
2. README_CHINESE_AI_SECURITY.md 전체 (20분)
3. SECURITY_ANALYSIS_SUMMARY.md 전체 (15분)
4. chinese_ai_services_security_analysis.md
   - Executive Summary (3분)
   - 5대 보안 영역 매트릭스 (15분)
   - 관심 서비스 상세 분석 (20분)
   - 권장사항 (10분)
```

**결과**: 구현 가능한 기술 권고 및 코드 샘플

---

### 시나리오 3: 완전 분석 (120분)

```
1. README_CHINESE_AI_SECURITY.md 전체 (25분)
2. SECURITY_ANALYSIS_SUMMARY.md 전체 (20분)
3. chinese_ai_services_security_analysis.md 전체 (75분)
```

**결과**: 모든 서비스의 상세 분석 및 장기 로드맵

---

## 자주 묻는 질문

### Q: 지금 당장 어디서 시작할까?

A: README_CHINESE_AI_SECURITY.md의 "실행 체크리스트" 섹션으로 이동.

Phase 1 (즉시)부터 시작:
1. ChaCha20-Poly1305 자격증명 암호화
2. 프롬프트 인젝션 Scanner 추가

---

### Q: 우리 서비스는 어느 Tier인가?

A: SECURITY_ANALYSIS_SUMMARY.md의 "빠른 참조: 보안 Tier" 참고.

Tier 1: 없음
Tier 2: Alibaba, Z.ai, OpenClawD
Tier 2-3: Baidu
Tier 3: Kimi, Zhipu

---

### Q: 코드 샘플은 어디?

A: 두 곳에 있음:
1. README_CHINESE_AI_SECURITY.md - "코드 예제" (간단함)
2. chinese_ai_services_security_analysis.md - "권장사항" (상세함)

---

### Q: 언제쯤 모든 서비스가 Tier 1이 될까?

A: 추정:
- Baidu: 6-9개월
- Alibaba: 8-12개월
- Z.ai: 9-12개월

---

## 파일 위치

```
/Users/jaesolshin/Documents/GitHub/compare_claws/reports/

├── INDEX_CHINESE_SECURITY.md (이 파일)
├── README_CHINESE_AI_SECURITY.md (가이드 + 체크리스트)
├── SECURITY_ANALYSIS_SUMMARY.md (요약 + Tier 분류)
└── chinese_ai_services_security_analysis.md (상세 분석)
```

---

## 다음 단계

### 1단계: 현재 상태 파악 (오늘)
- README_CHINESE_AI_SECURITY.md 읽기
- 서비스 Tier 분류 확인

### 2단계: 우선순위 수립 (내일)
- SECURITY_ANALYSIS_SUMMARY.md 읽기
- Phase 1 작업 리스트 작성

### 3단계: 구현 계획 (1주일)
- chinese_ai_services_security_analysis.md 상세 읽기
- 기술 스펙 작성
- 개발팀과 회의

### 4단계: Phase 1 실행 (1-2주)
- 자격증명 암호화 구현
- 프롬프트 인젝션 Scanner 추가

### 5단계: 재평가 (30일)
- Phase 1 완료 확인
- Phase 2 시작

### 6단계: Tier 재분류 (90일)
- 전체 개선사항 평가
- 보안 Tier 재분류
- 다음 분기 계획

---

## 버전 정보

- **분석 버전**: 1.0
- **분석 일자**: 2026-03-14
- **다음 업데이트**: 2026-04-14
- **작성자**: Security Reviewer Agent
- **모델**: Claude Haiku 4.5

---

## 피드백 & 업데이트

이 분석이 도움이 되셨나요? 다음을 고려하세요:

- 30일 후 각 서비스의 개선 진행 상황 평가
- 90일 후 Tier 재분류
- 새로운 보안 갭 발견 시 업데이트 요청

---

**마지막 업데이트**: 2026-03-14
**다음 리뷰**: 2026-04-14

질문이 있으신가요? README_CHINESE_AI_SECURITY.md의 FAQ 섹션을 참고하세요.
