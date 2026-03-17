# ClawWork 상세 분석 보고서

> **소스**: `reports/repos_applied/repos_applied_report.md` §3.1에서 추출
> **조사 일자**: 2026-03-07
> **조사 방법**: scientist 에이전트 소스코드 심층 분석
> **기반 프레임워크**: Nanobot

---

## 목차

1. [Executive Summary](#1-executive-summary)
2. [이중 실행 경로](#2-이중-실행-경로)
3. [Nanobot 비침습적 확장 — 7가지 기법](#3-nanobot-비침습적-확장--7가지-기법)
4. [경제 엔진의 세부 설계](#4-경제-엔진의-세부-설계)
5. [평가 시스템](#5-평가-시스템)
6. [미완성 부분](#6-미완성-부분)

---

## 1. Executive Summary

**핵심 철학**: "에이전트가 실제로 돈을 벌 수 있는가?" — 기술 지표 대신 경제적 생존을 측정

ClawWork는 Claw 생태계 응용 계층에서 **에이전트 경제 벤치마크**를 담당한다. Nanobot 소스를 수정하지 않고 7가지 기법으로 경제 추적 레이어를 삽입한다.

---

## 2. 이중 실행 경로

```
독립 시뮬레이션 (livebench/)
  GDPVal parquet → TaskManager → LiveAgent (LangChain + MCP) → WorkEvaluator → EconomicTracker
                                                               ↑
ClawMode 통합 (clawmode_integration/)               Nanobot AgentLoop 서브클래스
  Telegram/Discord → Nanobot → ClawWorkAgentLoop → /clawwork 명령 → 동일 EconomicTracker
```

두 경로가 `EconomicTracker`, `WorkEvaluator`, `TaskManager`를 공유한다. `LiveAgent`는 `langchain_openai.ChatOpenAI`와 MCP를 사용하며, ClawMode는 기존 Nanobot `AgentLoop`를 서브클래싱하여 메신저 게이트웨이에 통합된다 (`livebench/agent/live_agent.py:38`, `clawmode_integration/agent_loop.py:46`).

---

## 3. Nanobot 비침습적 확장 — 7가지 기법

Nanobot 소스 파일을 단 한 줄도 수정하지 않고 경제 추적 레이어를 삽입하는 방법:

| 기법 | 구체적 내용 | 파일:라인 |
|------|-----------|-----------|
| **서브클래싱** | `ClawWorkAgentLoop(AgentLoop)` | `agent_loop.py:46` |
| **도구 추가** | `_register_default_tools()` 오버라이드, `super()` 호출 유지 | `agent_loop.py:76` |
| **메서드 오버라이드** | `_process_message()`로 start/end_task 자동화 | `agent_loop.py:91` |
| **런타임 클래스 교체** | `self.provider.__class__ = CostCapturingLiteLLMProvider` | `agent_loop.py:63` |
| **투명 래퍼** | `TrackedProvider`로 `chat()` 가로채기 | `provider_wrapper.py:37` |
| **스킬 주입** | `SKILL.md` + `always: true`로 경제 프로토콜 상시 로딩 | `skill/SKILL.md:4` |
| **설정 분리** | `~/.nanobot/config.json`의 `agents.clawwork` 섹션 | `config.py:59` |

`CostCapturingLiteLLMProvider`는 `_parse_response()`를 오버라이드하여 OpenRouter의 `response.usage.cost`와 `response._hidden_params["response_cost"]`를 포착한다 (`provider_wrapper.py:18-34`). TrackedProvider는 `__getattr__`로 나머지 모든 호출을 원본 프로바이더에 위임한다 (`provider_wrapper.py:71`).

---

## 4. 경제 엔진의 세부 설계

### 비용 추적 우선순위 (`economic_tracker.py:158-173`)

```
OpenRouter 직접 보고 비용 > litellm 계산 > 로컬 공식 (input_price × tokens)
```

### 지급 이중 구조 — 평가 점수와 지급액의 비선형 관계

```
evaluation_score (GPT, 0-10) → 정규화 (÷10) → 0.0-1.0
  score >= 0.6:  payment = score × max_payment  (선형 비례)
  score < 0.6:   payment = $0.00               (하드 클리프)
```

0.59와 0.60 사이의 차이가 max_payment 전액이 된다 (`economic_tracker.py:380-395`, `livebench/llm_evaluator.py:166`). 최소 임계치 0.6은 인스턴스 생성 시 파라미터로 설정 가능하다.

### $10 시작 잔액의 설계 근거

Tavily 검색 1회 $0.0008, LLM 수십 회 호출이 잔액의 수%를 소진한다. 품질 미달 2-3회 반복 시 실제로 파산 위기가 발생하도록 설계됐다.

### 생존 상태 분류 (`economic_tracker.py:524-538`)

| 잔액 | 상태 |
|------|------|
| ≤ $0 | bankrupt |
| $0 ~ $100 | struggling |
| $100 ~ $500 | stable |
| > $500 | thriving |

---

## 5. 평가 시스템

44개 직업 카테고리별 GPT 루브릭 (`eval/meta_prompts/{Occupation}.json`). Fallback이 완전히 제거됐다 — 루브릭 파일이 없으면 `FileNotFoundError`, LLM 평가 실패 시 `raise ValueError`로 명시적 차단 (`evaluator.py:43`). 평가 전용 API 키(`EVALUATION_API_KEY`) 분리 지원.

**지급 공식** (GDPVal 기반):
```
Payment = quality_score × (estimated_hours × BLS_hourly_wage)
범위: $82.78 ~ $5,004.00, 평균: $259.45
```

---

## 6. 미완성 부분

| 항목 | 상태 | 코드 근거 |
|------|------|-----------|
| Trading 시스템 | 코드 존재, 완전 비활성화 | `live_agent.py:189` |
| 포트폴리오 가치 계산 | TODO | `economic_tracker.py:496` |
| 이미지/PDF 아티팩트 분석 | 메타데이터만 전달 | `llm_evaluator.py:264` |
| `get_cost_analytics()` | `record["type"]` KeyError 잠재 버그 | `economic_tracker.py:641` |
