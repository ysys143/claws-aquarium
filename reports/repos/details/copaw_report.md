# CoPaw 상세 분석 보고서

---

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub URL** | https://github.com/agentscope-ai/CoPaw |
| **Stars** | 13.6k |
| **언어** | Python 3.10-3.13 |
| **LOC** | 약 84,733줄 (src/ Python 파일 기준) |
| **라이선스** | Apache-2.0 |
| **기반 프레임워크** | AgentScope 1.0.18 + agentscope-runtime 1.1.2b2 |
| **개발 팀** | agentscope-ai (Alibaba 생태계) |
| **카탈로그 위치** | repos/ (16번째 프레임워크) |

---

## 2. 핵심 특징

CoPaw는 "개인 환경에서 실행되는 개인 비서"로, Skills 시스템을 통해 기능을 정의하고 14개 메시징 채널을 통해 사용자와 상호작용하는 standalone 에이전트 제품이다. AgentScope 1.0.18을 런타임 기반으로 사용하지만, 14개 채널 구현(84K LOC의 절반 이상), 독자적인 UnifiedQueueManager(3-tuple 키 격리), Playwright 기반 브라우저 자동화(3,460줄), 3단계 보안 스캐닝을 자체적으로 구현하고 있어 단순한 AgentScope 래퍼가 아닌 독립적인 제품이다. 기존 Claw 생태계에서 가장 많은 채널 수를 지원한다는 점에서 채널 다양성 기준으로는 생태계 최상위에 위치한다.

---

## 3. 아키텍처

### 3.1 디렉토리 구조

```
copaw/src/copaw/
├── agents/                   # 에이전트 핵심 로직
│   ├── react_agent.py        # ReAct 에이전트 구현
│   ├── routing_chat_model.py # 모델 라우팅
│   ├── command_handler.py    # 명령 처리
│   ├── skills_manager.py     # 스킬 실행 관리자 (2,557줄)
│   ├── skills_hub.py         # 스킬 허브/레지스트리 (1,671줄)
│   ├── tool_guard_mixin.py   # 도구 보안 믹스인
│   ├── memory/               # 에이전트 메모리
│   ├── skills/               # 내장 스킬 구현
│   └── tools/
│       └── browser_control.py # 브라우저 자동화 (3,460줄)
├── app/
│   ├── channels/             # 14개 채널 구현 (총 14,793줄)
│   │   ├── dingtalk/         # DingTalk (2,792줄)
│   │   ├── feishu/           # Feishu/Lark (2,100줄)
│   │   ├── qq/               # QQ (1,431줄)
│   │   ├── xiaoyi/           # Xiaoyi (1,423줄)
│   │   ├── telegram/         # Telegram (1,045줄)
│   │   ├── wecom/            # WeCom (1,036줄)
│   │   ├── mattermost/       # Mattermost (1,014줄)
│   │   ├── weixin/           # WeChat (896줄)
│   │   ├── discord_/         # Discord (629줄)
│   │   ├── imessage/         # iMessage (636줄)
│   │   ├── console/          # 콘솔 (570줄)
│   │   ├── matrix/           # Matrix (512줄)
│   │   ├── mqtt/             # MQTT (470줄)
│   │   └── voice/            # 음성 (239줄)
│   └── routers/
│       ├── skills.py         # 스킬 REST API (1,279줄)
│       └── skills_stream.py  # 스킬 스트리밍 API
├── channels/
│   └── unified_queue_manager.py # 3-tuple 큐 관리자
├── config/
│   └── config.py             # 설정 관리 (1,360줄)
├── providers/
│   └── provider_manager.py   # LLM 제공자 관리 (1,153줄)
├── security/
│   ├── tool_guard/rules/     # 도구 보안 규칙
│   └── skill_scanner/        # 스킬 보안 스캐닝
├── cli/
│   └── channels_cmd.py       # 채널 CLI (1,223줄)
└── console/                  # 웹 콘솔 빌드 산출물
```

### 3.2 핵심 의존성 (pyproject.toml)

| 패키지 | 버전 | 역할 |
|--------|------|------|
| agentscope | ==1.0.18 | 에이전트 런타임 기반 |
| agentscope-runtime | ==1.1.2b2 | 확장 런타임 |
| playwright | >=1.49.0 | 브라우저 자동화 |
| reme-ai | ==0.3.1.6 | ReMe 메모리 통합 |
| apscheduler | >=3.11.2,<4 | 작업 스케줄링 |
| dingtalk-stream | >=0.24.3 | DingTalk 채널 |
| lark-oapi | >=1.5.3 | Feishu/Lark 채널 |
| python-telegram-bot | >=20.0 | Telegram 채널 |
| paho-mqtt | >=2.0.0 | MQTT 채널 |
| matrix-nio | >=0.24.0 | Matrix 채널 |
| wecom-aibot-python-sdk | ==1.0.2 | WeCom 채널 |
| discord-py | >=2.3 | Discord 채널 |
| twilio | >=9.10.2 | iMessage/SMS 채널 |
| transformers | >=4.30.0 | 로컬 모델 |
| mss | >=9.0.0 | 화면 캡처 |

### 3.3 실행 흐름

```
CoPaw 시작
  --> config/config.py (설정 로드)
  --> channels/unified_queue_manager.py (큐 매니저 초기화)
  --> app/channels/{채널}/ (활성 채널 서버 시작)
        --> 메시지 수신
              --> unified_queue_manager.enqueue(
                      channel_id, session_id, priority_level, payload
                  )
              --> 동적 consumer 생성 (큐키별)
              --> agents/react_agent.py (ReAct 추론)
                    --> agents/skills_manager.py (스킬 선택)
                    --> security/tool_guard (도구 보안 검사)
                    --> agents/tools/browser_control.py (브라우저 작업)
              --> 채널로 응답 전송
```

---

## 4. 채널 아키텍처 (14채널)

CoPaw는 현재 분석된 Claw 생태계 프레임워크 중 가장 많은 채널을 지원한다. 채널 구현은 84K LOC의 상당 부분을 차지한다.

| 채널 | LOC | 라이브러리 | 특이사항 |
|------|-----|-----------|---------|
| DingTalk | 2,792 | dingtalk-stream | AI 카드, 마크다운 렌더링 |
| Feishu/Lark | 2,100 | lark-oapi | 기업 메신저 |
| QQ | 1,431 | (내장) | 중국 최대 메신저 |
| Xiaoyi | 1,423 | (내장) | Alibaba 음성 비서 |
| Telegram | 1,045 | python-telegram-bot | |
| WeCom | 1,036 | wecom-aibot-python-sdk | 기업용 WeChat |
| Mattermost | 1,014 | (REST API) | 오픈소스 Slack 대안 |
| WeChat | 896 | (내장) | 개인 WeChat |
| iMessage | 636 | twilio | Twilio 경유 |
| Console | 570 | (내장) | 로컬 CLI |
| Discord | 629 | discord-py | |
| Matrix | 512 | matrix-nio | 탈중앙화 프로토콜 |
| MQTT | 470 | paho-mqtt | IoT/임베디드 기기 |
| Voice | 239 | (내장) | 음성 입출력 |

특히 MQTT 채널은 이 생태계에서 유일하게 IoT 기기와의 통신을 지원한다.

---

## 5. UnifiedQueueManager (3-tuple 키 격리)

CoPaw의 가장 독창적인 내부 구조는 `channels/unified_queue_manager.py`다. 모든 채널의 메시지를 3-tuple 키 `(channel_id, session_id, priority_level)`로 관리한다.

```python
# 큐 키 = (채널ID, 세션ID, 우선순위)
QueueKey = Tuple[str, str, int]  # (channel_id, session_id, priority_level)

# 핵심 특성:
# 1. 다른 세션 → 동시 처리 (병렬)
# 2. 같은 세션, 다른 우선순위 → 동시 처리 (병렬)
# 3. 같은 세션 + 같은 우선순위 → 직렬화 (순서 보장)
# 4. 컨슈머 온디맨드 생성 (고정 워커 풀 없음)
# 5. 유휴 큐 자동 정리 (메모리 누수 방지)
```

이 설계는 기존 프레임워크들의 단순 FIFO 큐와 근본적으로 다르다. 14개 채널에서 수백 개 동시 세션이 발생할 때, 한 세션의 느린 처리가 다른 세션을 블록하지 않는다.

---

## 6. 스킬 시스템

### 6.1 SkillsManager (2,557줄)

에이전트가 어떤 스킬을 실행할지 라우팅하고, 스킬 실행 컨텍스트(세션, 채널, 사용자 정보)를 관리한다.

### 6.2 SkillsHub (1,671줄)

내장 스킬들의 레지스트리 및 실행 허브다. 내장 스킬 예시:
- PDF/Office 문서 처리
- 뉴스 다이제스트
- 파일 읽기/쓰기
- Cron 스케줄링
- 사용자 정의 스킬 (Python 함수 등록)

스킬은 API를 통해 동적으로 추가/제거할 수 있으며, `app/routers/skills.py` (1,279줄)가 REST API를 제공한다.

---

## 7. 브라우저 자동화 (browser_control.py, 3,460줄)

Playwright 기반의 브라우저 자동화 구현으로, Claw 생태계에서 단일 파일 기준 가장 큰 브라우저 제어 구현이다.

| 기능 | 설명 |
|------|------|
| 페이지 탐색 | URL 열기, 뒤로/앞으로 |
| 요소 조작 | 클릭, 입력, 선택 |
| 화면 캡처 | mss 라이브러리 연동 |
| 스크린샷 | Playwright 내장 |
| 스크롤 | 페이지 스크롤 제어 |
| 대화상자 처리 | alert, confirm, prompt |

---

## 8. 보안 아키텍처 (3단계)

### 8.1 Tool Guard (도구 실행 보안)

`security/tool_guard/rules/`에 YAML 규칙 파일로 도구 실행 정책을 정의한다. `tool_guard_mixin.py`가 모든 에이전트에 믹스인으로 주입된다.

### 8.2 File Access Guard (파일 접근 보안)

파일 시스템 접근 시 허용 경로와 차단 경로를 규칙 기반으로 검사한다.

### 8.3 Skill Security Scanning

`security/skill_scanner/`에서 새로운 스킬 코드를 등록 전 정적 분석으로 보안 검사한다. `security/skill_scanner/data/`에 알려진 악성 패턴 데이터베이스를 포함한다.

---

## 9. 신규 패턴 (R-번호)

**R42: 3-tuple QueueKey 채널-세션-우선순위 격리**
구현: CoPaw `channels/unified_queue_manager.py`
원리: 다채널 에이전트의 메시지 큐를 `(channel_id, session_id, priority_level)` 3-tuple로 키잉함으로써 채널 간, 세션 간, 우선순위 간 독립적 병렬 처리를 보장한다. 컨슈머를 고정 풀이 아닌 첫 메시지 도착 시 온디맨드로 생성하고, 유휴 시 자동 정리해 메모리 효율을 유지한다.
시사점: 14개 채널 × 다수 동시 세션 환경에서 단일 느린 세션이 다른 모든 세션을 블록하는 "head-of-line blocking" 문제를 구조적으로 제거한다.

---

## 10. 비교 테이블

| 항목 | CoPaw | OpenClaw | GoClaw |
|------|-------|---------|--------|
| 채널 수 | 14 | 5 | 7 |
| 기반 런타임 | AgentScope 1.0.18 | 독립 | 독립 (Go) |
| 브라우저 자동화 | Playwright (3,460줄) | 없음 | go-rod |
| 큐 격리 | 3-tuple 키 | 없음 | 없음 |
| 스킬 스캐닝 | 정적 분석 | 없음 | 없음 |
| IoT 채널 | MQTT 지원 | 없음 | 없음 |
| 중국 채널 | DingTalk, Feishu, QQ, WeChat, WeCom, Xiaoyi | 없음 | Feishu |
| Stars | 13.6k | ~5k | 1.4k |

---

## 11. 한계

- **AgentScope 버전 고정**: `agentscope==1.0.18` 핀 버전 — AgentScope 업스트림 변경에 즉시 대응하기 어려움
- **중국 채널 편향**: 14개 중 6개(DingTalk, Feishu, QQ, WeCom, WeChat, Xiaoyi)가 중국 플랫폼 — 글로벌 사용자 관점에서 유지보수 공백 가능성
- **OTel 부재**: AgentScope가 OTel을 내장하지만 CoPaw 레벨에서 별도 트레이싱 설정이 없음
- **멀티테넌트 없음**: 단일 사용자 "개인 비서" 전제 설계 — 기업 다중 사용자 배포에는 별도 고려 필요
- **onnxruntime 버전 제약**: `onnxruntime<1.24` 핀으로 일부 환경에서 충돌 가능

---

## 12. 참고 링크

- GitHub: https://github.com/agentscope-ai/CoPaw
- 기반 프레임워크: `reports/repos/details/agentscope_report.md`
- 채널 비교: `reports/repos/framework_catalog.md`
- 보안 비교: `reports/repos/security_report.md`
