# ClawCharts 상세 분析 보고서

> **작성일**: 2026-04-14
> **계층**: usecases/ -- 커뮤니티 콘텐츠 & 실사용 모음
> **자료 출처**: WebFetch https://clawcharts.com (2026-04-14 시점 렌더링)

---

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **URL** | https://clawcharts.com |
| **유형** | OpenClaw 생태계 특화 **GitHub 스타 성장 리더보드 대시보드** |
| **슬로건** | "Who will molt to the top?" |
| **운영자** | [@chrysb](https://twitter.com/chrysb) |
| **가격 모델** | 무료 (공개 대시보드, 로그인 불필요) |
| **추적 프로젝트 수** | **11개** |
| **생태계 총 스타 (스냅샷)** | **653,912 stars across 11 tracked repos** |
| **생태계 7일 성장률** | **+61,376 Stars · +10.4%** |
| **데이터 소스** | "GitHub API snapshots, synced every six hours and normalized to daily points" |
| **업데이트 주기** | 6시간 GitHub API 스냅샷 → 일별 정규화 포인트 |
| **업데이트 지연** | 페이지 상단에 "1분 전" 표시 (실시간 sync 상태) |
| **로컬 경로** | 없음 (웹사이트, 레포 클론 불가) |

---

## 2. 핵심 특징

ClawCharts는 **OpenClaw 생태계를 GitHub 스타 증가 속도로 가시화**하는 실시간 리더보드다. Awesome List가 "정적 컬렉션"이고 meetup 전사가 "사람 목소리"라면, ClawCharts는 "숫자로 보는 생태계 맥박"이다.

핵심 차별점 3가지:

1. **OpenClaw 생태계 특화 큐레이션**: 범용 AI 에이전트 집계가 아닌, OpenClaw 및 직계 파생 11개만 선별적으로 추적. "클로 생태계"라는 암묵적 경계를 사이트 소유자가 명시적으로 정의.
2. **동적 성장 지표 중심**: 총 스타 수보다 **7D 성장률, 기여자 수, 커밋 빈도**를 전면에 배치. "현재 누가 가장 빠르게 molt 중인가" 시각이 핵심.
3. **시간 범위 탭**: Overview / Market Share / Star Trend / Contrib Trend / Commit Trend × **7D / 14D / 30D / 90D** 축 조합으로 **멱집합적 비교**를 제공.

compare_claws 맥락에서의 위치: 우리 repos/가 "코드 구조를 정적으로 분석"한다면, ClawCharts는 "생태계 모멘텀을 동적으로 관측"한다. **시간축에서의 usecases/ 신호**.

---

## 3. 구조 分析

### 3.1 추적 프로젝트 11개

```
OpenClaw (flagship)
├── Hermes Agent       (Nous Research)
├── Paperclip          (paperclipai, 조직 오케스트레이션)
├── Nanobot            (HKUDS, 4K LOC 미니멀)
├── ZeroClaw           (zeroclaw-labs, 5MB RAM)
├── PicoClaw           (sipeed, Go, 10MB RAM)
├── NanoClaw           (qwibitai, TS, 컨테이너 swarm)
├── OpenFang           (Rust Agent OS, Hands)
├── IronClaw           (nearai, WASM 샌드박스)
├── NullClaw           (Zig 정적 바이너리)
└── TinyClaw           (jlia0, multi-agent TUI)
```

### 3.2 탭 / 메트릭 매트릭스

| 탭 | 설명 | 시간 범위 |
|----|------|----------|
| **Overview** | 순위, 메달(🥇🥈🥉), 총 스타, 7D 증가량, 시장 점유율 | 스냅샷 |
| **Market Share** | 생태계 전체 스타 중 프로젝트별 점유 비율 | 7D/14D/30D/90D |
| **Star Trend** | 시간별 스타 증가 곡선 | 7D/14D/30D/90D |
| **Contrib Trend** | 활성 기여자 수 추이 | 7D/14D/30D/90D |
| **Commit Trend** | 커밋 빈도 추이 | 7D/14D/30D/90D |

### 3.3 기여자 섹션

"활동 중인 상위 기여자" 목록 + 각자의 최근 커밋 수 노출. 프로젝트별 bus factor를 암묵적으로 시사.

### 3.4 데이터 파이프라인 (추정)

```
GitHub API (REST/GraphQL)
        |
        | (6시간마다 snapshot 수집)
        v
정규화 레이어 (일별 포인트로 resample)
        |
        v
정적 JSON/CSV 또는 서버리스 엔드포인트
        |
        v
프론트엔드 차트 렌더링 (React/Recharts/Chart.js 추정)
```

사이트 자체는 GitHub API 래퍼 + 시계열 시각화의 조합. 새로운 프레임워크 추가는 **큐레이터(@chrysb) 수동 등록** 방식으로 추정 (자동 discovery 없음).

---

## 4. 콘텐츠 分析

### 4.1 compare_claws 관점의 커버리지 격차

compare_claws는 repos/에 **14개**를 등록하고 있다. ClawCharts는 **11개**만 추적한다:

| 범주 | compare_claws repos/ | ClawCharts 추적 |
|------|---------------------|----------------|
| OpenClaw | ✅ | ✅ |
| Hermes Agent | ✅ | ✅ |
| Paperclip | (repos_applied/) | ✅ |
| Nanobot | ✅ | ✅ |
| ZeroClaw | ✅ | ✅ |
| PicoClaw | ✅ | ✅ |
| NanoClaw | ✅ | ✅ |
| OpenFang | ✅ | ✅ |
| IronClaw | ✅ | ✅ |
| NullClaw | ✅ | ✅ |
| TinyClaw | ✅ | ✅ |
| **Claude Code** | ✅ (repos/cc_2.1.80) | ❌ |
| **OpenJarvis** | ✅ | ❌ |
| **NemoClaw** | ✅ (OpenClaw 플러그인) | ❌ |
| **CoPaw** | ✅ | ❌ |
| **GoClaw** | ✅ | ❌ |

**ClawCharts에 없는 5개**:
- **Claude Code** (Anthropic 공식): "오픈 생태계가 아님" 해석 가능 — ClawCharts가 오픈소스/커뮤니티에 한정
- **OpenJarvis** (Stanford local-first): 커뮤니티 인지도 대비 ClawCharts 큐레이터 미등록
- **NemoClaw**: OpenClaw의 **플러그인**이라 독립 레포로 추적하지 않음 (합리적)
- **CoPaw** (13.6k★): Xiaohongshu/AgentScope 기반이라 "Claw 파생" 경계 밖으로 판단한 가능성
- **GoClaw**: 최근 등장, 큐레이터 미등록 지연

**compare_claws에 없는 ClawCharts 항목**: 없음. 우리가 슈퍼셋.

### 4.2 암묵적 "Claw 생태계" 경계 정의

ClawCharts의 선별은 **"OpenClaw의 직계 인지도 파생"**이라는 암묵 기준을 드러낸다:
- "Claw" 이름 명명 + OpenClaw 보안/크기/언어 차별화 주장 = **포함**
- OpenClaw와 독립적으로 설계되었거나, 다른 베이스(AgentScope, Anthropic 등)에서 파생 = **제외** 경향

이는 compare_claws의 "Claw 생태계 + 인접" 넓은 범위보다 좁은 큐레이션이다. 우리가 OpenJarvis/CoPaw/GoClaw를 포함하는 것은 "실용적으로 비교 가치가 있는 런타임"이라는 기능적 기준을 사용하기 때문.

### 4.3 시장 점유율 관점 (snapshot 653,912★)

11개 중 상위 4개가 생태계 스타의 ~90% 독점으로 추정 (일반적 long-tail 분포 가정):
- OpenClaw (flagship, 가장 큰 비중 추정)
- Hermes Agent (9.3k★)
- Paperclip (53k★, 최근 급성장)
- Nanobot (Karpathy "noticed")

나머지 7개는 long-tail. 이는 compare_claws가 Tier 1(IronClaw/ZeroClaw/NullClaw) 등 **작지만 아키텍처적으로 중요한** 프레임워크에 과대 관심을 가지는 것과 대비된다. ClawCharts는 "큰 게 흥미롭다", compare_claws는 "다양한 게 흥미롭다".

### 4.4 커뮤니티 평가 루프의 한계

GitHub 스타는 **선행 지표**(관심)지 **성숙 지표**(실사용)가 아니다:
- Hermes Agent 같은 신생이 Tirith/LLM Wiki/Self-Evolution 같은 차별화로 급성장하면 스타 스파이크 → ClawCharts 순위 상승
- 반대로 OpenClaw 같은 성숙 프레임워크는 안정적 증가율이라 일별 델타가 작음 → 순위 체감 하락 가능
- **7D 성장률 중시 설계가 "신상 편향"을 유발**한다. 사용자가 "누가 최상단으로 올라올까"에 집중하면 성숙도·안정성 대신 "현재 트렌드"를 선택하게 됨.

---

## 5. 신규 패턴 R-번호

**없음**. GitHub API + 시계열 시각화 + 수동 큐레이션의 조합은 **공지의 구현 패턴** (shields.io, Star-History, GitHub Trending과 동등). OpenClaw 특화 큐레이션 자체는 주제 선정의 문제지 아키텍처 패턴이 아님.

---

## 6. 비교 테이블

| 축 | ClawCharts | awesome-openclaw-usecases (hesamsheikh) | awesome-openclaw-agents (mergisi) | agency-agents (msitarzewski) | Star-History (공지 집계) |
|----|-----------|---------------------------------------|----------------------------------|-----------------------------|------------------------|
| **유형** | 실시간 대시보드 | 정적 마크다운 리스트 | 정적 마크다운 + SOUL.md 템플릿 | 정적 멀티-툴 페르소나 | 범용 GitHub 스타 추적 |
| **지표** | 스타/기여자/커밋 시계열 | use-case 40개 (도메인 분류) | SOUL.md 174개 + use-case 132개 | 60+ 에이전트 × 10 도구 포맷 | 스타만 |
| **업데이트** | 6시간마다 자동 | 커뮤니티 PR 수동 | 커뮤니티 PR 수동 | 커뮤니티 PR 수동 | 실시간 |
| **큐레이션 범위** | OpenClaw 직계 11개 | 범용 (use-case 도메인) | 범용 (직군) | 범용 (AI 도구) | 임의 지정 |
| **시간축** | 7D/14D/30D/90D | 없음 | 없음 | 없음 | 임의 |
| **R-번호** | 없음 | R31 | (없음, agency 파생) | R34 | — |
| **실무 유용성** | 프레임워크 모멘텀 파악 | 구체 활용 사례 탐색 | 직군별 에이전트 템플릿 선택 | 10개 도구 공통 에이전트 배포 | 개별 레포 역사 |

**ClawCharts 고유 포지션**: 타 usecases 항목이 "내용(무엇을 만드는가)"이라면 ClawCharts는 "속도(생태계 어디가 뜨거운가)"다. 상호 보완적.

---

## 7. 한계

1. **스타 = 관심 ≠ 품질/성숙도**: 앞서 4.4에서 언급. 의사결정 시 ClawCharts 순위만 믿으면 "트렌드 편향" 위험.
2. **큐레이터 단일 실패점**: @chrysb 개인이 운영. 중단 시 대체재 없음. 오픈소스 레포가 아니라 **재현성/포크 불가**.
3. **데이터 주권 불투명**: "GitHub API snapshots"만 명시, 원본 스냅샷 공개 여부 불명. 독립 재분석 불가.
4. **메트릭 정의 부재**: "Contributors" 활성 기준, "Commits" 스팸 필터링 등 **메트릭 정의 문서 없음**. 해석이 사용자 상식에 의존.
5. **API 키 없음 / 검색 없음**: 외부 자동화 연동 불가 (수동 방문만). 우리가 정기적으로 이 사이트 스냅샷을 받으려면 웹 스크래핑 필요.
6. **5개 프레임워크 누락**: 4.1에서 지적. 특히 CoPaw(13.6k★), GoClaw(1.4k★)는 크기 기준으로 당연히 포함 대상인데 빠져 있음 — 큐레이션 지연/편향.
7. **모바일 친화성 불명**: 시간 범위 탭 × 트렌드 탭 × 11 프로젝트 UI가 작은 화면에서 어떻게 접히는지 미확인.

---

## 8. 참고 링크

- **사이트**: https://clawcharts.com
- **운영자 Twitter**: https://twitter.com/chrysb
- **관련 usecases 항목**:
  - `reports/usecases/details/awesome_openclaw_usecases_report.md` (정적 활용 사례)
  - `reports/usecases/details/awesome_openclaw_agents_report.md` (정적 에이전트 템플릿)
  - `reports/usecases/details/agency_agents_report.md` (R34 멀티-툴 트랜스파일)
  - `reports/usecases/details/openclaw_seoul_meetup_0315_report.md` (밋업 전사)
- **교차 참조**:
  - `reports/usecases/usecases_index.md` (4계층 전체 인덱스)
  - `reports/repos/framework_catalog.md` (14개 런타임 — ClawCharts보다 넓은 커버리지)
- **유사 집계 서비스**: [star-history.com](https://star-history.com), [GitHub Trending](https://github.com/trending), [shields.io](https://shields.io)

---

**최종 수정**: 2026-04-14
