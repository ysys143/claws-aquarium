# idea3: AI Research Agent -- 학술 연구 자동화 에이전트

## 한 줄 요약

학술 논문 검색-수집-분석-정리-집필을 에이전트 파이프라인으로 자동화하는 시스템.
Claw 프레임워크 분석에서 얻은 기억 아키텍처 + 도구 설계 + 세션 관리 인사이트를 활용.


## 배경

### 문제 인식

연구자의 워크플로우는 본질적으로 다단계 에이전트 작업이다:

1. 논문 검색 (arxiv, pubmed, google scholar, nature, PNAS, bioRxiv, OUP...)
2. 서지 수집 및 관리 (Zotero, Mendeley)
3. 논문 읽기 및 요약
4. 지식 정리 및 연결 (Obsidian, Notion)
5. 분석/시뮬레이션 (코드 작성, 데이터 분석)
6. 집필 (논문, 슬라이드, 포스터)

현재 각 단계가 분절되어 있고, 도구 간 연결이 수동이다.

### 기존 시도들

| 프로젝트 | 접근 방식 | 한계 |
|----------|-----------|------|
| [magi-researchers](https://github.com/Axect/magi-researchers) | Claude+Gemini+Codex 3-AI 교차검증 연구 | Claude Code 플러그인, 범용 리서치용 |
| [arXiv_explorer](https://github.com/Axect/arXiv_explorer/) | TF-IDF 기반 arXiv 논문 추천 + AI 요약 | arXiv 단일 소스, 터미널 UI |
| [satang](https://github.com/revfactory/satang) | NotebookLM 영감, 업로드 기반 대화+인포그래픽+슬라이드 | 논문 검색 없음, 수동 업로드 |
| [qbio-skills](https://github.com/hyeshik/qbio-skills) | Claude Code 스킬로 연구장비 구매 문서 자동화 | 특정 업무(구매) 한정 |
| AI Scientist (Sakana AI) | 논문 아이디어 생성부터 집필까지 완전 자동화 | 연구 "보조"가 아니라 연구 "대체" 지향 |
| Zotero + Obsidian | Zotero 서지관리 + Obsidian 노트 연동, AI 토큰 없이 자동 태깅 | 검색/분석 자동화 없음 |

### 관련 인프라/생태계

| 프로젝트 | 역할 |
|----------|------|
| [serverless-openclaw](https://github.com/serithemage/serverless-openclaw) | AWS 서버리스 OpenClaw 배포 (월 ~$1) |
| [OpenSCV](https://github.com/revfactory/OpenSCV) | Slack에서 Claude Code 원격 실행 |
| [summon](https://github.com/TheMagicTower/summon) | Rust 리버스 프록시, 모델별 LLM 라우팅 |
| [awesome-openclaw-usecases](https://github.com/hesamsheikh/awesome-openclaw-usecases) | OpenClaw 실사용 사례 36건+ |
| [yonsei-thesis-typst](https://github.com/Axect/yonsei-thesis-typst) | Typst 학위논문 템플릿 |
| Telegram Bot API 9.3+ | AI 챗봇 실시간 스트리밍 지원 ([참고](https://news.hada.io/topic?id=27163)) |

### 관련 키워드/개념

- **검색 기술**: BM25, embedding, RAG, hybrid search, graph RAG
- **에이전트 아키텍처**: ReAct, tool calling, MCP, subagent, skill, plan/todos/tasks
- **기존 도구**: Claude Code, DeepAgent, CC Workflow Studio
- **학회/도메인**: CoSyne (Computational & Systems Neuroscience)
- **강화학습**: InternRL, research agent MARS


## 핵심 아이디어

### Claw 분석에서 가져올 인사이트

idea2.md의 4개 분석 보고서에서 직접 활용 가능한 설계 패턴:

**1. 기억 아키텍처 (memory_architecture_report.md)**
- Tier 1 하이브리드 검색 (BM25 + vector + temporal decay) -> 논문 DB 검색에 직접 적용
- Dual Injection 패턴 (항상 로드 + 온디맨드 검색) -> 연구 맥락 유지
- ZeroClaw Soul Snapshot -> 장기 연구 프로젝트의 "연구 상태" 저장/복원
- 임베딩 기반 유사 논문 검색 + BM25 키워드 매칭 = 최적 논문 검색

**2. 세션/컨텍스트 관리 (session_context_report.md)**
- 멀티 에이전트 컨텍스트 격리 -> 검색/분석/집필을 별도 에이전트로 분리
- 프로젝트 세션 추상화 (아직 아무도 안 만든 계층) -> 연구 프로젝트 단위 세션
- Auto-compaction + 기억 연속성 -> 장기 연구 진행 상황 추적

**3. 보안 (security_report.md)**
- API 키 관리 (arxiv, pubmed, semantic scholar API 등)
- Credential proxy 패턴 -> 에이전트에게 안전하게 API 접근 위임

**4. 도구 아키텍처 (browser_actions_report.md)**
- MCP 표준화 -> 논문 검색/다운로드/파싱을 MCP 도구로 구현
- 스킬 시스템 -> 연구 워크플로우를 재사용 가능한 스킬로 정의

### 구성 요소 (안)

```
[사용자]
   |
   | (메신저: Telegram / Slack / 터미널)
   |
[Orchestrator Agent]  -- 연구 맥락 유지, 작업 계획, 에이전트 배분
   |
   +-- [Paper Search Agent]     -- 다중 소스 검색 (arxiv, pubmed, scholar, nature...)
   |     +-- arxiv MCP tool
   |     +-- pubmed MCP tool
   |     +-- semantic scholar MCP tool
   |     +-- google scholar scraper
   |
   +-- [Paper Analysis Agent]   -- 논문 읽기, 요약, 비교 분석
   |     +-- PDF 파서
   |     +-- 구조화된 요약 생성
   |     +-- 인용 네트워크 분석
   |
   +-- [Knowledge Manager]      -- 서지 관리 + 지식 그래프
   |     +-- Zotero 연동 (자동 태깅)
   |     +-- Obsidian 노트 생성/업데이트
   |     +-- 임베딩 DB (논문 벡터 검색)
   |
   +-- [Writer Agent]           -- 집필 보조
   |     +-- 논문 초안 (LaTeX/Typst)
   |     +-- 슬라이드 생성 (satang 방식)
   |     +-- 연구 노트 정리
   |
   +-- [Data/Code Agent]        -- 분석 실행
         +-- 통계 분석, 시뮬레이션
         +-- 그래프/시각화
```

### 차별화 포인트

1. **"보조"에 집중**: AI Scientist처럼 연구를 대체하는 게 아니라, 반복적 검색/정리/서식 작업을 자동화
2. **기억 연속성**: Claw Tier 1 기억 스택으로 장기 연구 맥락 유지 (세션 넘어서)
3. **다중 소스 통합**: arxiv만이 아니라 pubmed, nature, google scholar 등 학제간 검색
4. **Zotero/Obsidian 연동**: 기존 연구자 워크플로우에 끼워넣기, 새 도구 강요 안 함
5. **메신저 인터페이스**: 24시간 대기, "이 주제로 최신 논문 찾아줘"에 즉시 응답
6. **프로젝트 세션**: idea2.md에서 발견한 미구현 계층 -- 연구 프로젝트 단위의 세션 관리


## 기술 스택 후보

### 논문 검색 API
- arxiv API (open, 무료)
- PubMed E-utilities / NCBI API
- Semantic Scholar API (citation graph 포함)
- CrossRef API (DOI 메타데이터)
- OpenAlex API (open scholarly metadata)
- Google Scholar (scraping 필요, 공식 API 없음)
- bioRxiv/medRxiv API

### 서지관리 연동
- Zotero Web API (공식, REST)
- Zotero Translation Server (메타데이터 추출)
- Obsidian local vault (파일시스템 직접 접근)

### 임베딩/검색
- sqlite-vec 또는 LanceDB (OpenClaw 패턴)
- BM25 (SQLite FTS5)
- 하이브리드 검색 (Claw Tier 1에서 입증된 패턴)

### 에이전트 런타임
- Claude Code SDK (subagent, skill, MCP)
- 또는 Claw 프레임워크 중 하나를 베이스로 (OpenClaw? NanoClaw?)


## 구현 접근 방식 후보

### A. Claude Code 스킬/플러그인으로 구현
- 가장 빠른 시작. qbio-skills, magi-researchers와 같은 접근.
- paper-search, paper-analyze, paper-summarize 스킬 이미 설치되어 있음.
- 한계: 장기 기억, 프로젝트 세션 관리는 Claude Code 인프라에 의존.

### B. OpenClaw 확장으로 구현
- 가장 풍부한 기억 스택 (sqlite-vec + LanceDB + hybrid search).
- 24시간 메신저 에이전트 이미 구현됨.
- MCP 도구 + 플러그인 시스템 활용.
- serverless-openclaw로 저렴한 배포 가능.

### C. 독립 에이전트로 처음부터 구현
- Claw 분석에서 "최적 조합"을 체리피킹.
- ZeroClaw Soul Snapshot + OpenClaw hybrid search + IronClaw identity protection.
- 가장 유연하지만 가장 많은 작업량.


## 참고 자료

### GitHub
- https://github.com/Axect/magi-researchers -- Multi-AI 교차검증 연구 플러그인
- https://github.com/Axect/arXiv_explorer/ -- TF-IDF arXiv 추천 + AI 요약
- https://github.com/revfactory/OpenSCV -- Slack 원격 Claude Code 실행
- https://github.com/revfactory/satang -- AI 지식 노트북 (인포그래픽+슬라이드)
- https://github.com/revfactory/satang/blob/main/docs/slide-generation-analysis.md -- 슬라이드 생성 분석
- https://github.com/hyeshik/qbio-skills -- 연구장비 구매 자동화 스킬
- https://github.com/hesamsheikh/awesome-openclaw-usecases -- OpenClaw 사용 사례
- https://github.com/serithemage/serverless-openclaw -- 서버리스 OpenClaw 배포
- https://github.com/TheMagicTower/summon -- LLM 라우팅 프록시
- https://github.com/Axect/yonsei-thesis-typst -- Typst 학위논문 템플릿

### 외부 링크
- https://x.com/ArtemXTech/status/2028330693659332615
- https://x.com/i/status/2022682231677022408
- https://drive.google.com/file/d/1eAuw8Sg1XKdIMJUr7PylOS3N6f00QxHC/view?usp=sharing
- https://news.hada.io/topic?id=27163 -- Telegram Bot API 9.3+ AI 스트리밍

### 키워드
llm, bm25, embedding, rag, agent, ReAct, tool, mcp, graph, claude code, plan, todos, tasks, subagent, skill, deepagent, AI scientist, CC Workflow Studio, research agent MARS, InternRL, CoSyne


## 열린 질문

Q1. 스킬/플러그인 vs 독립 에이전트 vs OpenClaw 확장 -- 어느 접근이 현실적인가?
Q2. Google Scholar 공식 API 부재 -- scraping vs Semantic Scholar/OpenAlex로 대체 가능한가?
Q3. Zotero + Obsidian 자동 태깅에서 "AI 토큰 없이"라는 건 정확히 어떤 구조인가?
Q4. 논문 전문(full-text) 접근 -- PDF 파싱 vs publisher API vs Sci-Hub 대안?
Q5. 기억 스택의 적정 규모 -- 논문 1000편 수준에서 vector DB 성능은?
Q6. 연구 도메인 특화 vs 범용 -- CoSyne/neuroscience에 먼저 집중할 것인가?
Q7. 상용 서비스화 가능성 -- Elicit, Consensus, Semantic Scholar 등과의 차별화?
Q8. 기존 paper-* 스킬 (paper-search, paper-download, paper-summarize)과의 관계 -- 확장? 대체?
