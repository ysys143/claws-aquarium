# compare_claws — 프로젝트 규칙 & 컨벤션

이 문서는 Claude Code가 compare_claws 프로젝트에서 작업할 때 따라야 할 규칙과 컨벤션을 정의합니다.

---

## 프로젝트 개요

**compare_claws**는 Claw 생태계(OpenClaw 및 파생 AI 에이전트 프레임워크)를 종합 분석하는 연구 저장소입니다.

- **구성**: 코드 기반 비교(소스 레포 클론) + 마크다운 보고서 + 기억 파일
- **핵심 산출물**: 12개+ 상세 보고서, 교차 분석, 보안 Tier 분류, 신규 패턴(R-번호) 추출
- **주요 관심**: 아키텍처, 보안, 기억 관리, 브라우저 자동화, 배포 전략, 상용화

---

## 디렉토리 구조 & 역할

```
compare_claws/
├── repos/              # 12개 에이전트 런타임 프레임워크 (git clone)
├── repos_applied/      # Claw 기반 응용 프로젝트 (스킬, 도구, 서비스)
├── repos_research/     # 연구 자동화 도구 (RL, 실험 루프, 논문 분석)
├── reports/            # 분석 보고서
│   ├── repos/          # 프레임워크별 분석
│   │   ├── details/    # 상세 보고서 (<name>_report.md)
│   │   ├── framework_catalog.md       # 전체 카탈로그 + 목차
│   │   ├── framework_cross_analysis.md
│   │   ├── security_report.md
│   │   ├── memory_architecture_report.md
│   │   ├── browser_actions_report.md
│   │   └── session_context_report.md
│   ├── repos_applied/
│   ├── repos_research/
│   └── deployment/, commercial/
├── ideas/              # 메모, 아이디어 문서
├── meetup/             # 발표 자료
├── MEMORY.md           # 프로젝트 메모리 (프레임워크 요약, 패턴 목록, 보안 Tier)
└── README.md           # 프로젝트 소개
```

---

## 레포 등록 워크플로우 (3계층)

### 1. `repos/` — 프레임워크 런타임

**조건**: Claw 생태계의 에이전트 런타임 자체

**등록 절차**:
1. `git clone <url> repos/<framework-name>`
2. `reports/repos/details/<framework-name>_report.md` 작성 (상세 분석)
3. `reports/repos/framework_catalog.md` 업데이트
   - 목차 섹션에 링크 추가
   - 본문에 새 섹션 추가
   - 종합 비교표에 행 추가
4. `reports/repos/security_report.md` 업데이트 (보안 Tier 분류)
5. `MEMORY.md` 업데이트
   - repos 카운트 증가
   - 프레임워크별 항목 추가
   - 신규 패턴 발견 시 R-번호 부여

### 2. `repos_applied/` — 응용 계층

**조건**: Claw 프레임워크 위에 구축된 응용 프로젝트, 스킬, 도구, 서비스

**등록 절차**:
1. `git clone <url> repos_applied/<name>`
2. `reports/repos_applied/details/<name>_report.md` 작성
3. `reports/repos_applied/repos_applied_report.md` 섹션 추가
4. `MEMORY.md` 업데이트

### 3. `repos_research/` — 연구 도구

**조건**: AI 에이전트 연구 자동화 도구 (RL 훈련, 실험 루프, 논문 분석 등)

**등록 절차**:
1. `git clone <url> repos_research/<name>`
2. `reports/repos_research/<name>_report.md` 작성
3. `MEMORY.md` 업데이트

### 4. `usecases/` — 커뮤니티 콘텐츠 & 실사용 모음

**조건**: OpenClaw 생태계 실사용 사례 모음, 어썸 리스트(awesome list), 밋업/컨퍼런스 콘텐츠

**등록 절차**:
1. GitHub 레포: `git clone <url> usecases/<name>` / 로컬: `cp -r <source> usecases/<name>`
2. `reports/usecases/details/<name>_report.md` 작성 (아래 적용 양식 사용)
3. `reports/usecases/usecases_index.md` 업데이트
4. `MEMORY.md` 업데이트

**usecases/ 보고서 양식** (코드 레포가 아닌 커뮤니티 콘텐츠에 맞게 조정된 8섹션):

| 섹션 | 내용 |
|------|------|
| 1. 기본 정보 | 소스 URL/경로, Stars, 유형, 큐레이터, 날짜, 항목 수 |
| 2. 핵심 특징 | 커뮤니티 필요 해결 방식, 생태계 내 위치 |
| 3. 구조 분析 | 콘텐츠 정보 아키텍처 (카테고리 체계, 분류 방식) |
| 4. 콘텐츠 분析 | 주요 항목, 커버리지, 공백, 큐레이션 품질 |
| 5. 신규 패턴 R-번호 | R1–현재와 비교, 진짜 신규만 부여 |
| 6. 비교 테이블 | 유사 커뮤니티 아티팩트 2–3개와 비교 |
| 7. 한계 | 커버리지 공백, 큐레이션 편향, 진부화 위험 |
| 8. 참고 링크 | 소스, 이 저장소 관련 보고서 교차 링크 |

---

## 보고서 작성 규칙

### 상세 보고서 (details/*.md) 구조

모든 프레임워크/도구 상세 보고서는 이 순서로 구성:

1. **기본 정보 테이블**
   - GitHub URL, Stars, 언어, LOC, 라이선스, 생성일, 개발 팀

2. **핵심 개념 / 핵심 특징**
   - "이것이 뭐고 왜 중요한가"를 1문단 요약

3. **아키텍처**
   - 디렉토리 구조 (트리 형식)
   - 주요 파일 설명 (5-10개, 라인 수 포함)
   - 실행 흐름 (진입점 -> 주요 모듈 -> 출력)

4. **주제별 섹션** (프레임워크 특성에 맞게)
   - 메모리 / 도구 / 보안 / 플랫폼 / 채널 / 배포 등
   - 각 섹션: 개념 설명(줄글) + 구체적 구현(코드 블록/표)

5. **신규 패턴 (R-번호)**
   - 기존 다른 프레임워크에 없는 고유 아키텍처만 부여
   - 형식: "**R23: 패턴명** — 간단한 설명 (이 프레임워크에서만 구현)"
   - MEMORY.md의 "New Patterns from..." 섹션에 등록

6. **비교 테이블**
   - 유사 프레임워크(2-3개)와 대조
   - 열: 기능, 보안, 성능, 철학

7. **한계 (Limitations)**
   - 명시적으로 설계 목표 밖의 것들
   - "이 프레임워크는 X를 지원하지 않으므로 Y 용도에 부적합"

8. **참고 링크**
   - GitHub, 문서, 관련 논문, 참고 보고서

### 글쓰기 스타일

- **언어**: 한국어로 작성
- **구조**: 표와 코드블록 적극 활용, 개념 설명은 충분한 줄글로
- **구체성**: 단순 나열이 아닌 "왜 이것이 중요한가"를 설명
- **근거**: 프레임워크 간 비교 시 수치, 파일명, 라인 수 포함
- **열린 질문**: 분석 결과에 대한 미해결 질문을 마지막에 포함

### 예시

**좋은 예시**:
```markdown
## 메모리 아키텍처

OpenJarvis는 3개 저장소로 계층화한다:
- Layer 0: 현재 세션(SQLite, 24시간)
- Layer 1: 중기 기억(Qdrant 벡터 DB, 30일)
- Layer 2: 장기 기억(파일 시스템, 무제한)

이는 기존 9개 Claw 프레임워크(전부 JSON 또는 SQLite 단일 계층)와 다르다.
```

**나쁜 예시**:
```markdown
## 메모리
OpenJarvis는 메모리를 지원한다. (왜? 어떻게?)
```

---

## MEMORY.md 관리

위치: `/Users/jaesolshin/.claude/projects/-Users-jaesolshin-Documents-GitHub-compare-claws/memory/MEMORY.md`

**프레임워크/도구 등록 시 반드시 업데이트할 항목**:

1. **Memory Files** — 새 프레임워크 분석 파일 있으면 링크
2. **Directory Structure** — repos/, reports/, ideas/ 카운트 갱신
3. **Reports** — 새 보고서 추가 (번호, 제목, 핵심 발견)
4. **프레임워크별 항목** — 요약 (별도 섹션)
5. **New Patterns (R-번호)** — 신규 아키텍처 패턴 추가
6. **Security Tiers** — 보안 분류 업데이트
7. **Open Questions** — 미해결 질문 번호 계속 증가

---

## 신규 패턴 (R-번호) 부여 기준

**부여 조건**:
- 기존 12개 Claw 프레임워크 어디에도 없는 고유 아키텍처 패턴
- 충분히 일반화 가능한 개념 (단순 구현 차이는 제외)
- 다른 프로젝트에서 재사용 가능할 수준

**현재 상태**: R1~R22 부여됨. 다음은 R23부터.

**형식**:
```markdown
**R23: 패턴명** — 간단한 설명.
구현한 프레임워크: FrameworkName
원리: 2-3문장으로 핵심 원리 설명
시사점: 다른 프레임워크에 어떻게 응용 가능한가
```

---

## 보안 Tier 분류

`reports/repos/security_report.md`에 명시된 기준 준수:

| Tier | 기준 | 예시 |
|------|------|------|
| **Tier 1** | WASM/Docker + 암호화 + HITL 승인 + 다층 방어 | IronClaw, ZeroClaw, OpenFang |
| **Tier 2** | Docker + RBAC + 자격증명 격리 (암호화 없음) | OpenClaw, OpenJarvis, NanoClaw |
| **Tier 3** | 정규식 차단 + 파일시스템 제한 | Nanobot, PicoClaw |
| **Tier 4** | 최소 메커니즘 | TinyClaw |

새 프레임워크 추가 시 이 기준에 따라 분류.

---

## 코드 리뷰 규칙

이 프로젝트는 **분석 중심 저장소**입니다:

- [OK] **클론한 레포 분석**: 소스코드 읽기, 구조 파악, 문제점 지적 — 자유
- [OK] **보고서 작성**: 마크다운 문서 작성, 비교표, 패턴 추출 — 자유
- [NO] **레포 수정**: 클론한 레포(repos/, repos_applied/, repos_research/)의 코드 변경 — 금지
  - 예외: 하위 프로젝트 중 로컬 구현 예제(`ideas/` 등)는 수정 가능

---

## 검증 & 테스트

마크다운 보고서 검증 체크리스트:

- [CHECK] 모든 코드 예시는 실제 소스에서 추출했는가? (발명/추측 금지)
- [CHECK] 비교표 수치(LOC, Stars, 파일 수 등)는 최신 상태인가?
- [CHECK] 링크(GitHub, 문서, 참고 보고서)가 유효한가?
- [CHECK] 신규 패턴(R-번호)은 정말 새로운가? (MEMORY.md와 기존 보고서 확인)
- [CHECK] 보안 Tier는 security_report.md와 일치하는가?

---

## 작업 시작 체크리스트

새 프레임워크/도구 분석을 시작할 때:

1. [TASK] `repos/` (또는 `repos_applied/` 또는 `repos_research/`)에 클론
2. [TASK] `reports/repos/details/<name>_report.md` 파일 생성
3. [TASK] README.md와 핵심 소스 읽기 (5-10개 파일)
4. [TASK] 상세 보고서 8개 섹션 작성
5. [TASK] 신규 패턴 있으면 R-번호 부여 & MEMORY.md에 등록
6. [TASK] `framework_catalog.md`와 `security_report.md` 업데이트
7. [TASK] 링크 유효성 검증
8. [TASK] MEMORY.md 최종 업데이트

---

## 참고 문서

- `/Users/jaesolshin/Documents/GitHub/compare_claws/README.md` — 프로젝트 개요
- `/Users/jaesolshin/.claude/projects/-Users-jaesolshin-Documents-GitHub-compare-claws/memory/MEMORY.md` — 누적된 프레임워크/패턴 지식
- `/Users/jaesolshin/Documents/GitHub/compare_claws/reports/repos/security_report.md` — 보안 Tier 정의
- `/Users/jaesolshin/.claude/CLAUDE.md` — 전역 Claude Code 규칙

---

**최종 수정**: 2026-03-20
