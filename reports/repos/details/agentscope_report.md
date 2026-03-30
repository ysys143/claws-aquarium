# AgentScope 상세 분석 보고서

---

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub URL** | https://github.com/agentscope-ai/agentscope |
| **Stars** | 22k |
| **언어** | Python 3.10+ |
| **LOC** | 약 42,442줄 (src/ Python 파일 기준) |
| **라이선스** | Apache-2.0 |
| **arXiv 논문** | cs.MA-2402.14034 |
| **개발 팀** | Alibaba Tongyi Lab (SysML team) |
| **카탈로그 위치** | repos/ (15번째 프레임워크) |

---

## 2. 핵심 특징

AgentScope는 "A Flexible yet Robust Multi-Agent Platform"을 표방하는 Alibaba Tongyi Lab의 연구 프레임워크다. 기존 Claw 생태계의 프레임워크들이 채널·도구·메모리를 각자 재구현한 것과 달리, AgentScope는 OpenTelemetry를 코어 의존성으로 내장해 모든 LLM 호출·에이전트 응답·임베딩을 분산 트레이싱으로 자동 기록한다. 이에 더해 프롬프트 자동 튜닝(tuner), 모델 자동 선택(model selection), A2A(Agent-to-Agent) 프로토콜, MCP(>=1.13) 통합을 단일 패키지에 결합함으로써 "에이전트 프레임워크"보다 "에이전트 플랫폼 인프라"에 가까운 위상을 차지한다. CoPaw, ReMe 등 파생 프레임워크들의 기반 런타임이기도 하다.

---

## 3. 아키텍처

### 3.1 디렉토리 구조

```
agentscope/src/agentscope/
├── agent/            # 에이전트 기본 클래스 + ReAct 에이전트
├── a2a/              # Agent-to-Agent 프로토콜 (a2a-sdk + nacos)
├── embedding/        # 임베딩 모델 추상화
├── evaluate/         # 평가 모듈
├── formatter/        # 메시지 포맷 변환
├── hooks/            # 라이프사이클 훅
├── mcp/              # MCP 클라이언트/서버 (>=1.13)
├── memory/           # SQLAlchemy + mem0 메모리 (31줄 인터페이스)
├── message/          # 메시지 타입 정의
├── model/            # LLM 제공자 추상화
├── module/           # 모듈 시스템
├── pipeline/         # MsgHub + chat_room 파이프라인
├── plan/             # 실행 계획 모듈
├── rag/              # RAG 파이프라인
├── realtime/         # WebSocket 실시간 통신
├── session/          # 세션 관리
├── token/            # 토큰 계산
├── tool/             # 도구 실행
├── tracing/          # OTel 분산 트레이싱 (1,917줄)
│   ├── _trace.py     # 트레이싱 데코레이터 (646줄)
│   ├── _extractor.py # 트레이스 추출기 (892줄)
│   ├── _attributes.py # OTel 속성 정의 (183줄)
│   └── _converter.py # 포맷 변환 (125줄)
├── tune/             # 파인튜닝 지원
├── tuner/            # 프롬프트/모델 자동 튜닝 (737줄)
│   ├── _config.py    # 튜닝 설정 (267줄)
│   ├── _model.py     # 모델 선택기 (148줄)
│   ├── _tune.py      # 튜닝 실행 (96줄)
│   └── _workflow.py  # 튜닝 워크플로우 (54줄)
├── tts/              # 텍스트-음성 변환
└── types/            # 공통 타입
```

### 3.2 핵심 의존성 (pyproject.toml)

| 패키지 | 버전 | 역할 |
|--------|------|------|
| opentelemetry-api | >=1.39.0 | 분산 트레이싱 API |
| opentelemetry-sdk | >=1.39.0 | 트레이싱 SDK |
| opentelemetry-exporter-otlp | >=1.39.0 | 트레이스 내보내기 |
| mcp | >=1.13 | MCP 프로토콜 클라이언트 |
| sqlalchemy | - | 메모리 영속성 |
| a2a-sdk | (optional) | Agent-to-Agent 프로토콜 |
| nacos-sdk-python | >=3.0.0 (optional) | A2A 서비스 디스커버리 |
| anthropic | - | Claude API |
| dashscope | - | Alibaba 모델 API |
| openai | - | OpenAI 호환 API |

### 3.3 실행 흐름

```
agentscope 초기화
  --> tracing/_setup.py (OTel 트레이서 등록)
  --> agent/ (에이전트 인스턴스 생성)
        --> model/ (LLM 제공자 선택)
        --> hooks/ (라이프사이클 훅 등록)
        --> mcp/ (MCP 서버 연결)
  --> pipeline/MsgHub (메시지 라우팅)
        --> agent.reply() 호출
              --> tracing/_trace.py 자동 계측
              --> tool/ 실행
              --> memory/ 조회/업데이트
        --> tracing/_extractor.py (트레이스 수집)
```

---

## 4. 분산 트레이싱 (OTel-first 설계)

AgentScope의 가장 두드러진 특징은 OpenTelemetry를 선택적 플러그인이 아닌 **코어 의존성**으로 채택한 것이다. `tracing/` 모듈은 1,917줄로 상당한 비중을 차지하며, `@trace`, `@trace_llm`, `@trace_reply`, `@trace_toolkit`, `@trace_embedding` 데코레이터를 통해 에이전트의 모든 핵심 동작을 자동 계측한다.

```python
# tracing/__init__.py에서 노출하는 퍼블릭 API
from ._setup import setup_tracing
from ._trace import (
    trace,         # 일반 함수 트레이싱
    trace_llm,     # LLM 호출 트레이싱
    trace_reply,   # 에이전트 응답 트레이싱
    trace_format,  # 포맷 변환 트레이싱
    trace_toolkit, # 도구 실행 트레이싱
    trace_embedding, # 임베딩 트레이싱
)
```

`_extractor.py` (892줄)는 OTel 스팬에서 구조화된 에이전트 실행 데이터를 추출해 평가 파이프라인에 제공한다. 이는 단순 로깅이 아닌 "분산 에이전트 실행의 완전한 인과 추적"을 의미한다.

---

## 5. 프롬프트 튜너 + 모델 선택기

`tuner/` 모듈(737줄)은 기존 Claw 프레임워크에 없는 **자동 최적화 계층**을 제공한다.

```python
# tuner/_config.py - 튜닝 설정 (267줄)
# 자동으로 최적 프롬프트와 모델을 선택하는 워크플로우

# tuner/_model.py - 모델 선택기 (148줄)
# 작업 유형, 비용, 성능 지표를 기반으로 LLM 자동 선택

# tuner/_tune.py - 튜닝 실행 (96줄)
# 프롬프트 변형 생성 및 평가

# tuner/_workflow.py - 튜닝 워크플로우 (54줄)
# 튜닝 파이프라인 오케스트레이션
```

`tune/` 모듈은 파인튜닝(fine-tuning)을 지원한다. MetaClaw의 R37(MAML 기반 스킬 진화)이 런타임 적응을 다룬다면, AgentScope의 tuner는 배포 전 프롬프트/모델 최적화를 자동화한다.

---

## 6. A2A (Agent-to-Agent) 프로토콜

`a2a/` 모듈은 Google의 A2A 프로토콜(a2a-sdk)을 구현하고, `nacos-sdk-python`을 통해 마이크로서비스 방식의 에이전트 서비스 디스커버리를 지원한다. 에이전트가 서로를 HTTP 엔드포인트로 노출하고 등록/발견할 수 있어, 분산 멀티에이전트 시스템 구성이 가능하다.

```toml
# pyproject.toml optional dependencies
[project.optional-dependencies]
a2a = [
    "a2a-sdk",
    "httpx",
    "nacos-sdk-python>=3.0.0",   # 서비스 디스커버리
]
```

이는 같은 프로세스 내 에이전트 협업에 집중하는 OpenJarvis, OpenClaw와 달리 진정한 분산 에이전트 아키텍처를 지원한다.

---

## 7. 메모리 아키텍처

AgentScope의 메모리는 SQLAlchemy 기반 관계형 영속성과 Redis 워킹 메모리, mem0 장기 기억을 조합한다.

| 계층 | 구현 | 범위 |
|------|------|------|
| 워킹 메모리 | Redis (fast) | 현재 세션 |
| 관계형 저장 | SQLAlchemy | 중장기 |
| 장기 기억 | mem0 통합 | 영속 |
| 임베딩 | embedding/ 모듈 | 시맨틱 검색 |

---

## 8. MCP 통합 (>= 1.13)

`mcp/` 모듈은 MCP 1.13 이상을 필수 의존성으로 요구하는 가장 최신 MCP 통합을 보여준다. 기존 Claw 프레임워크들이 MCP를 optional로 취급하는 것과 달리 AgentScope는 MCP를 에이전트 도구 연결의 기본 메커니즘으로 채택한다.

---

## 9. 신규 패턴 (R-번호)

**R40: OTel-first 관찰 가능성 (코어 의존성 내장)**
구현: AgentScope `tracing/` 모듈, opentelemetry-api/sdk/exporter 코어 의존성
원리: OpenTelemetry를 선택적 플러그인이 아닌 필수 의존성으로 채택하고, `@trace_llm`, `@trace_reply`, `@trace_toolkit` 데코레이터를 프레임워크 레벨에서 자동 적용한다. 에이전트의 모든 동작(LLM 호출, 응답, 도구 실행, 임베딩)이 자동으로 분산 트레이싱 스팬에 기록된다.
시사점: 프레임워크 자체가 "관찰 가능성 기반 설계"를 강제함으로써 디버깅, 비용 추적, 에이전트 행동 감사가 추가 설정 없이 가능해진다.

**R41: 내장 프롬프트 튜너 + 모델 자동 선택기**
구현: AgentScope `tuner/` 모듈 (_config.py 267줄, _model.py 148줄)
원리: 에이전트 배포 전 단계에서 주어진 작업에 대한 최적 프롬프트 변형과 LLM을 자동으로 탐색·평가한다. 비용, 지연시간, 품질 지표를 종합해 모델을 선택하는 _model.py는 단순 프롬프트 엔지니어링을 넘어 모델 선택까지 자동화한다.
시사점: "어떤 모델을 써야 하나"를 사람이 결정하지 않아도 되는 self-optimizing 에이전트 배포 파이프라인 구현 가능.

---

## 10. 비교 테이블

| 항목 | AgentScope | OpenJarvis | MetaClaw |
|------|-----------|-----------|---------|
| OTel 내장 | 코어 의존성 | 없음 | 없음 |
| 프롬프트 튜닝 | 자동화 (tuner/) | 수동 | MAML 기반 런타임 |
| A2A 프로토콜 | a2a-sdk + nacos | 없음 | 없음 |
| MCP 버전 | >=1.13 (필수) | 없음 | 없음 |
| 파인튜닝 | tune/ 내장 | 없음 | 없음 |
| 파생 프레임워크 | CoPaw, ReMe | 없음 | 없음 |
| Stars | 22k | ~2k | 낮음 |
| 연구 논문 | arXiv 2402.14034 | 없음 | 없음 |

---

## 11. 한계

- **Alibaba 종속성**: `dashscope` (Alibaba 모델 API)가 코어 의존성에 포함되어 있어 Alibaba 생태계 외 사용자에게 불필요한 의존성이 발생
- **채널 부재**: AgentScope 자체에는 메시징 채널(Telegram, Discord 등)이 없음 — CoPaw 같은 파생 프레임워크에 위임
- **복잡한 의존성 트리**: opentelemetry + mcp + sqlalchemy + a2a-sdk가 모두 코어 의존성 — 경량 배포에 부적합
- **self-hosted 전용**: 관리형 클라우드 서비스 없음

---

## 12. 참고 링크

- GitHub: https://github.com/agentscope-ai/agentscope
- 문서: https://doc.agentscope.io/
- arXiv: https://arxiv.org/abs/2402.14034
- 로드맵: https://github.com/agentscope-ai/agentscope/blob/main/docs/roadmap.md
- 파생 프레임워크: `reports/repos/details/copaw_report.md`
