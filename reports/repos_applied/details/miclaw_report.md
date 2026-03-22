# Xiaomi MiClaw 상세 분석 보고서

> **조사 일자**: 2026-03-20
> **조사 방법**: 공개 정보(TechNode, Gizmochina, Gadgets360, Xiaomi Community, XiaomiTime 등) 기반 분석
> **소스 유형**: 비공개 (클로즈드 베타, GitHub 없음)
> **기반 모델**: Xiaomi MiMo LLM (MiMo-V2 계열)
> **분류**: repos_applied — OpenClaw 생태계의 모바일 OS 응용 사례

---

## 목차

1. [기본 정보](#1-기본-정보)
2. [핵심 개념 — 왜 MiClaw가 중요한가](#2-핵심-개념--왜-miclaw가-중요한가)
3. [아키텍처](#3-아키텍처)
4. [도구 및 기능 계층](#4-도구-및-기능-계층)
5. [Human × Car × Home — 생태계 컨텍스트 통합](#5-human--car--home--생태계-컨텍스트-통합)
6. [보안 및 권한 모델](#6-보안-및-권한-모델)
7. [신규 패턴 (R-번호)](#7-신규-패턴-r-번호)
8. [비교 테이블](#8-비교-테이블)
9. [한계 (Limitations)](#9-한계-limitations)
10. [참고 링크](#10-참고-링크)

---

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **공식 명칭** | Xiaomi micLaw (미클로) |
| **개발사** | Xiaomi (소미, 샤오미) |
| **공개 여부** | 비공개 (Proprietary, 클로즈드 베타) |
| **GitHub** | 없음 |
| **출시 상태** | 2026년 3월 초 제한 클로즈드 베타 |
| **기반 LLM** | Xiaomi MiMo (자체 개발 대형언어모델) |
| **플랫폼** | HyperOS (Xiaomi 스마트폰 전용 OS) |
| **언어** | 비공개 |
| **라이선스** | 독점 |
| **포지셔닝** | "OpenClaw-like mobile AI agent" (자사 표현) |
| **지원 기기** | Xiaomi 14 시리즈 이상 (HyperOS 탑재 기종) |

---

## 2. 핵심 개념 — 왜 MiClaw가 중요한가

MiClaw는 Claw 생태계가 **메신저 채널(Telegram, Discord)과 터미널**을 벗어나 **모바일 OS 자체로 침투**하는 최초의 상용 사례다.

기존 9개 Claw 프레임워크(OpenClaw, Nanobot, NanoClaw 등)는 외부 채널을 통해 에이전트를 호출하는 구조였다. 사용자는 Telegram에 메시지를 보내면 에이전트가 응답하고, 그 에이전트가 다시 도구를 호출하는 간접적 구조다. **MiClaw는 이 채널 중간자를 제거했다.** OS가 직접 에이전트 런타임이 되어, 사용자가 자연어로 말하면 스마트폰 OS 레벨 API를 직접 호출한다.

Xiaomi는 MiClaw를 "AI가 도구를 위한 도구(phone becomes a tool for AI)"로 표현했다. 이는 기존 AI 비서(Siri, Bixby)가 "AI가 정보 제공 도구"였던 것과 근본적으로 다른 철학이다.

---

## 3. 아키텍처

### 3.1 실행 루프

Xiaomi가 공개한 아키텍처는 **"추론-실행 루프(Inference-Execution Loop)"**다:

```
사용자 자연어 입력
       ↓
  MiMo LLM (의도 파악 + 도구 선택)
       ↓
  Tool Call (시스템 API / 서드파티 앱 / IoT 기기)
       ↓
  결과 평가 (LLM이 결과를 검증)
       ↓
  다음 단계 판단 (완료 or 재시도 or 다음 도구)
       ↓
  응답 반환 or 추가 실행
```

이는 ReAct(Reasoning + Acting) 패턴의 모바일 OS 구현이다. 핵심 차별점은 **중간 채널 없이** HyperOS 시스템 레이어에 직접 바인딩된다는 점이다.

### 3.2 레이어 구조

```
┌─────────────────────────────────────────┐
│         사용자 인터페이스 (HyperOS)        │
├─────────────────────────────────────────┤
│         MiClaw Agent Runtime             │
│   - MiMo LLM (의도 파악 + 계획)           │
│   - Tool Router (50+ 기능 디스패치)        │
│   - Result Evaluator (성공/실패 판단)      │
├──────────────┬──────────────────────────┤
│  System APIs │  3rd Party App APIs      │
│  (50+ built-in│  (앱 인텐트 기반)          │
│  functions)  │                          │
├──────────────┴──────────────────────────┤
│         IoT / Ecosystem Layer            │
│  (스마트홈 기기, 차량, 웨어러블)             │
└─────────────────────────────────────────┘
```

### 3.3 주요 컴포넌트

| 컴포넌트 | 역할 | 특이사항 |
|---------|------|---------|
| `MiMo LLM` | 의도 이해 + 도구 선택 | 온디바이스 or 하이브리드 |
| `agent_xiaomi_home` | 홈 IoT 제어 에이전트 | 실시간 기기 상태 조회 |
| `Tool Registry` | 50+ 시스템 기능 등록 | SMS, 파일, 캘린더, 스마트홈 |
| `Context Injector` | IoT 상태 → LLM 컨텍스트 | 실시간 물리 환경 반영 |

---

## 4. 도구 및 기능 계층

MiClaw가 공개한 **50+ 기능**은 3개 카테고리로 분류된다:

### 4.1 시스템 레벨 기능

| 기능 분류 | 예시 |
|---------|------|
| **커뮤니케이션** | SMS 읽기/쓰기, 연락처 관리, 전화 발신 |
| **파일 관리** | 파일 읽기/쓰기, 사진 정리, 문서 검색 |
| **캘린더/일정** | 일정 생성, 알림 설정, 일정 조회 |
| **기기 제어** | 화면 밝기, 볼륨, Bluetooth, Wi-Fi |

### 4.2 서드파티 앱 통합

앱을 직접 조작할 수 있는 Android Intent 기반 연동이다. 특정 앱의 API가 아닌 OS 레벨 인텐트 시스템을 통해 앱을 에이전트 도구로 전환한다.

### 4.3 Xiaomi 에코시스템 도구

```
agent_xiaomi_home 에이전트 예시 동작:
  입력: "친구가 30분 뒤 와"
  처리:
    1. 실시간 홈 상태 조회 → 로봇청소기 작동 중 확인
    2. 커튼 상태 확인 → 닫혀 있음
    3. 에어컨 상태 확인
  출력:
    → 로봇청소기 일시정지 스케줄링
    → 커튼 오픈 명령
    → "30분 뒤 청소 완료, 커튼 열림 완료" 응답
```

이 예시에서 LLM은 사용자의 **암묵적 의도**(청소 완료 + 거실 정돈)를 추론하여 명시적으로 요청하지 않은 작업을 자율 실행했다.

---

## 5. Human × Car × Home — 생태계 컨텍스트 통합

MiClaw의 가장 독특한 특징은 **물리 세계 상태를 LLM 컨텍스트의 1등급 입력**으로 취급하는 구조다. Xiaomi는 이를 "Human × Car × Home" 삼각형으로 표현한다.

```
        [사용자 (Human)]
            /     \
           /       \
    [스마트폰]    [자동차]
           \       /
            \     /
          [스마트홈]
```

### 실제 데모 사례

**시나리오**: 사용자가 Xiaomi 스토어 방문 중
- MiClaw가 사용자 위치(Xiaomi Store) 인식
- 집 기기 상태 조회 (냉장고 재고, 현재 작동 기기 목록)
- 크로스레퍼런스: 위치 + 집 상태 → 맥락적 쇼핑 리스트 생성

이는 단순 도구 호출(tool use)을 넘어 **분산된 물리 환경의 실시간 상태를 LLM 컨텍스트에 동적 주입**하는 패턴이다. 기존 9개 Claw 프레임워크는 텍스트/API 컨텍스트만 다뤘고, 물리 환경 상태(IoT 기기, 위치, 차량 상태)를 에이전트 컨텍스트로 통합한 사례가 없었다.

---

## 6. 보안 및 권한 모델

클로즈드 베타 단계이므로 세부 내용은 제한적이다. 공개된 정보 기준:

| 항목 | 내용 |
|------|------|
| **권한 모델** | 사용자 명시적 승인(authorization) 필요 |
| **데이터 처리** | 로컬(온디바이스) 또는 하이브리드 |
| **보안 계층** | HyperOS 권한 시스템 위임 (추정) |
| **Sandbox** | Android 앱 샌드박스 (시스템 앱 권한) |

비공개 클로즈드 소스이므로 보안 아키텍처의 내부 구현(Tier 분류)은 현재로선 판단 불가. HyperOS가 제공하는 OS 레벨 권한 분리가 기반 방어선이 된다.

**보안 Tier: 미분류** (공개 정보 부족)

---

## 7. 신규 패턴 (R-번호)

### R27: OS-Native Agent Runtime

**정의**: 에이전트 런타임이 메신저 채널 또는 터미널을 거치지 않고 모바일 OS 시스템 레이어에 직접 통합되는 아키텍처.

구현한 프레임워크: Xiaomi MiClaw

원리: 기존 모든 Claw 프레임워크는 "외부 채널(Telegram, Discord, CLI) → 에이전트 → 도구 호출"의 간접 구조다. MiClaw는 HyperOS에 에이전트 런타임을 시스템 서비스로 내장하여 채널 중간자를 제거한다. 50+ 시스템 API를 에이전트 도구로 직접 바인딩하고, OS 레벨 권한으로 앱 인텐트 시스템에 접근한다.

시사점: "AI 비서 앱"이 아닌 "OS 기능으로서의 AI"라는 배포 패러다임. Claude Code(R26, npm vendor에 BPF 동봉)가 개발 환경에서 OS 통합을 시도했다면, MiClaw는 소비자 기기에서 동일 패러다임을 구현한다.

---

### R28: Physical World Context Injection

**정의**: 실시간 IoT 기기 상태, 사용자 위치, 차량 상태 등 물리 환경 데이터를 LLM 컨텍스트의 1등급 입력으로 동적 주입하는 에이전트 아키텍처.

구현한 프레임워크: Xiaomi MiClaw (Human × Car × Home)

원리: 사용자 텍스트 입력 외에 "현재 집의 물리 상태(로봇청소기 ON, 커튼 상태)"를 Context Injector가 실시간으로 수집해 LLM 프롬프트에 주입한다. LLM은 이 물리 컨텍스트를 바탕으로 **사용자가 명시하지 않은 암묵적 의도**를 추론하고 실행한다. 기존 프레임워크의 컨텍스트는 대화 히스토리 + DB 조회였으나, R28은 실세계 센서/기기 상태가 컨텍스트 소스가 된다.

시사점: 에이전트 메모리 아키텍처 연구(memory_architecture_report.md)가 다룬 "컨텍스트 출처"를 텍스트 DB에서 물리 환경으로 확장. IoT-aware agent context가 다음 단계 에이전트 아키텍처의 핵심 레이어가 될 수 있음.

---

## 8. 비교 테이블

| 항목 | Xiaomi MiClaw | OpenClaw | Hermes Agent | Apple Intelligence |
|------|--------------|---------|-------------|-------------------|
| **채널** | OS 시스템 레이어 | Telegram/Discord/etc | 6종 메시징 | Siri/앱 인텐트 |
| **기반 모델** | Xiaomi MiMo (독점) | 모델 무관 | 모델 무관 | Apple 자체 모델 |
| **도구 수** | 50+ 시스템 기능 | MCP 표준 (무제한) | agentskills.io | SiriKit 액션 |
| **IoT 통합** | 네이티브 (HyperOS) | 없음 | 없음 | HomeKit 제한적 |
| **오픈소스** | 비공개 | 완전 오픈소스 | MIT | 비공개 |
| **배포 형태** | OS 빌트인 | 셀프호스팅 | 셀프호스팅 | OS 빌트인 |
| **RL 학습** | MiMo 사전훈련 | OpenClaw-RL (별도) | Atropos 환경 | 비공개 |
| **보안 Tier** | 미분류 | Tier 2 | Tier 2+ | 미분류 |

---

## 9. 한계 (Limitations)

1. **Xiaomi 에코시스템 종속**: HyperOS + Xiaomi 기기 전용. 타사 스마트폰, 타사 IoT 기기 지원 미정. Android 표준 Intent는 지원하지만 최적화는 Xiaomi 기기 중심.

2. **클로즈드 소스**: 내부 아키텍처 검증 불가. 보안 감사 불가. 커뮤니티 확장 불가. 오픈소스 Claw 생태계와의 기여 단절.

3. **클로즈드 베타 단계**: 공개 정보가 제한적. 실제 시스템 프롬프트, 도구 구현, 실패 처리 로직 미공개.

4. **규제 리스크**: 스마트폰 시스템 레벨 접근(SMS, 파일, 위치) + AI 에이전트 = GDPR/개인정보 규제 고위험. 글로벌 출시 일정 불확실.

5. **오프라인 한계**: 온디바이스 MiMo 추론의 품질 한계. 복잡한 다단계 작업에서 하이브리드(클라우드 fallback) 필요.

6. **MiMo 모델 고정**: OpenClaw처럼 사용자가 모델을 교체할 수 없음. MiMo의 추론 품질에 에이전트 전체 성능이 종속.

---

## 10. 참고 링크

- [Xiaomi begins limited closed beta of OpenClaw-like mobile AI agent Xiaomi miclaw (TechNode, 2026-03-06)](https://technode.com/2026/03/06/xiaomi-begins-limited-closed-beta-of-openclaw-like-mobile-ai-agent-xiaomi-miclaw/)
- [Xiaomi announces miclaw, an autonomous AI assistant for smartphones (Gizmochina)](https://www.gizmochina.com/2026/03/06/xiaomi-announces-miclaw-an-autonomous-ai-assistant-for-smartphones/)
- [Xiaomi Testing Experimental AI Agent Miclaw (Gadgets360)](https://www.gadgets360.com/ai/news/xiaomi-miclaw-ai-agent-experimental-closed-beta-testing-performs-complex-tasks-smartphones-devices-details-11177661)
- [Xiaomi Miclaw AI Demo: See the MiMo AI Automate Your Entire Life (XiaomiTime)](https://xiaomitime.com/xiaomi-miclaw-ai-demo-92244/)
- [Xiaomi Miclaw: Revolutionary Mobile AI Agent (AICost)](https://aicost.org/blog/xiaomi-miclaw-mobile-ai-agent-review)
- [Xiaomi MiMo-V2-Pro (OpenRouter)](https://openrouter.ai/xiaomi/mimo-v2-pro/performance)
- [Xiaomi MiMo-V2-Flash GitHub](https://github.com/XiaomiMiMo/MiMo-V2-Flash)
- 관련 보고서: `reports/repos_applied/repos_applied_report.md`
- 관련 보고서: `reports/repos/details/claude_code_report.md` (R26 OS 통합 비교)
- 관련 보고서: `reports/repos/memory_architecture_report.md` (컨텍스트 아키텍처 비교)
