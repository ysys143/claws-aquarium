# GoClaw 상세 분석 보고서

---

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub URL** | https://github.com/nextlevelbuilder/goclaw |
| **Stars** | 1.4k |
| **언어** | Go 1.26 |
| **LOC** | 약 176,000줄 (Go 파일 기준) |
| **라이선스** | CC BY-NC 4.0 (비상업적 사용만 허용) |
| **커밋 수** | 832+ |
| **개발 팀** | nextlevelbuilder (개인/소규모 팀) |
| **카탈로그 위치** | repos/ (14번째 프레임워크) |

---

## 2. 핵심 특징

GoClaw는 "Go port of OpenClaw with enhanced security, multi-tenant PostgreSQL, and production-grade observability"를 표방하는 멀티테넌트 AI 에이전트 게이트웨이다. Python 기반의 기존 Claw 생태계와 달리 Go로 구현되어 단일 바이너리로 배포되며, PostgreSQL 기반 테넌트 분리, Docker 샌드박스, AES 암호화, Tailscale VPN 통합, OpenTelemetry 기반 관찰 가능성을 조합한 엔터프라이즈급 보안 스택을 제공한다. 20개 이상의 LLM 제공자와 7개 메시징 채널을 지원하며, 에이전트 팀 구성 시 블록 의존성 그래프(blocked_by)를 통해 오케스트레이션을 선언적으로 정의한다.

---

## 3. 아키텍처

### 3.1 디렉토리 구조

```
goclaw/
├── cmd/                      # CLI 진입점 + 게이트웨이 로직 (9,872줄)
│   ├── gateway.go            # 메인 게이트웨이 서버 (1,188줄)
│   ├── gateway_agents.go     # 에이전트 팀 + 임베딩 (275줄)
│   ├── gateway_consumer_handlers.go  # 채널 메시지 핸들러 (611줄)
│   ├── gateway_providers.go  # LLM 제공자 등록 (448줄)
│   ├── gateway_otel.go       # OTel 트레이싱 초기화
│   └── skills_cmd.go         # 스킬 CLI 명령
├── internal/                 # 핵심 내부 모듈
│   ├── agent/                # 에이전트 런타임
│   ├── channels/             # 7개 채널 구현
│   ├── crypto/aes.go         # AES 암호화 (122줄)
│   ├── knowledgegraph/       # 지식 그래프
│   ├── mcp/                  # MCP 프로토콜
│   ├── memory/               # 임베딩 + 벡터 메모리
│   ├── oauth/                # OAuth 인증
│   ├── permissions/          # RBAC 권한 관리
│   ├── sandbox/sandbox.go    # Docker 샌드박스 (219줄)
│   ├── scheduler/            # 작업 스케줄러
│   ├── sessions/             # 세션 관리
│   ├── skills/               # 스킬 실행
│   ├── store/                # PostgreSQL + SQLite 저장소
│   ├── tasks/                # 에이전트 태스크 보드
│   ├── tracing/              # OTel 트레이싱
│   └── tts/                  # 텍스트-음성 변환
├── migrations/               # DB 마이그레이션
├── skills/                   # 내장 스킬 정의
├── ui/desktop/               # Wails v2 데스크톱 UI
├── docker-compose*.yml       # 환경별 Compose 파일 (9개)
├── Dockerfile.sandbox        # 샌드박스 전용 이미지
└── go.mod                    # 의존성 정의
```

### 3.2 주요 의존성 (go.mod)

| 패키지 | 버전 | 역할 |
|--------|------|------|
| go.opentelemetry.io/otel | v1.40.0 | 분산 트레이싱 |
| github.com/jackc/pgx/v5 | v5.6.0 | PostgreSQL 드라이버 |
| github.com/redis/go-redis/v9 | v9.18.0 | Redis 캐시 |
| tailscale.com | v1.94.2 | VPN 메시 네트워킹 |
| github.com/go-rod/rod | v0.116.2 | 브라우저 자동화 |
| github.com/wailsapp/wails/v2 | v2.11.0 | 데스크톱 UI 프레임워크 |
| github.com/mymmrac/telego | v1.6.0 | Telegram 채널 |
| github.com/bwmarrin/discordgo | v0.29.0 | Discord 채널 |
| github.com/slack-go/slack | v0.19.0 | Slack 채널 |
| github.com/gorilla/websocket | v1.5.4 | WebSocket 채널 |
| github.com/adhocore/gronx | v1.19.6 | Cron 스케줄링 |
| github.com/zalando/go-keyring | v0.2.8 | OS 키체인 자격증명 |
| golang.org/x/time | v0.14.0 | 속도 제한(rate limiting) |

### 3.3 실행 흐름

```
main.go
  --> cmd/gateway.go (서버 초기화)
        --> internal/store (PostgreSQL 테넌트 초기화)
        --> internal/channels (7개 채널 등록)
        --> internal/providers (LLM 제공자 로드)
        --> internal/sandbox (Docker 샌드박스 준비)
        --> cmd/gateway_otel.go (OTel 트레이서 시작)
        --> 채널별 메시지 수신 루프
              --> cmd/gateway_consumer_handlers.go
                    --> internal/agent (에이전트 실행)
                          --> internal/tools / internal/skills
                          --> internal/memory (임베딩 검색)
```

---

## 4. 멀티테넌트 아키텍처

GoClaw의 핵심 차별점은 PostgreSQL 기반 완전한 멀티테넌트 격리다. 모든 데이터(에이전트, 세션, 메시지, 제공자 설정)는 `tenant_id`로 분리되며, `store.MasterTenantID`는 시스템 레벨 설정(임베딩 제공자, 시스템 설정)에만 사용된다.

```go
// internal/store의 테넌트 컨텍스트 패턴
masterCtx := store.WithTenantID(context.Background(), store.MasterTenantID)

// 임베딩 제공자 우선순위:
// 1. system_configs "embedding.provider" (tenant = master)
// 2. DB 제공자 중 settings.embedding.enabled = true 자동 감지
```

기존 Claw 프레임워크는 모두 단일 테넌트(single-user) 구조이며, GoClaw가 유일하게 멀티테넌트 PostgreSQL 격리를 구현한다.

---

## 5. 보안 아키텍처 (5계층)

### 5.1 Docker 샌드박스 (3축 제어)

```go
// internal/sandbox/sandbox.go (219줄)
// Mode: 어느 에이전트를 샌드박싱할지
type Mode string
const (
    ModeOff     Mode = "off"      // 샌드박스 없음
    ModeNonMain Mode = "non-main" // main 제외 전체
    ModeAll     Mode = "all"      // 모든 에이전트
)

// Access: 워크스페이스 파일시스템 권한
type Access string
const (
    AccessNone Access = "none"  // 파일 접근 차단
    AccessRO   Access = "ro"    // 읽기 전용
    AccessRW   Access = "rw"    // 읽기/쓰기
)

// Scope: 컨테이너 재사용 범위
type Scope string
const (
    ScopeSession Scope = "session"  // 세션당 1 컨테이너
    ScopeAgent   Scope = "agent"    // 에이전트당 1 컨테이너
    ScopeShared  Scope = "shared"   // 공유 컨테이너
)
```

이 3축(Mode x Access x Scope) 조합은 기존 다른 Claw 프레임워크의 단순 on/off 샌드박스보다 훨씬 세밀한 보안 정책 설정을 가능하게 한다.

### 5.2 AES 암호화

`internal/crypto/aes.go` (122줄)에서 API 키 및 민감 데이터를 AES로 암호화해 DB에 저장한다. OpenClaw, NanoClaw 등 Python 기반 프레임워크들이 평문 저장하는 것과 대비된다.

### 5.3 OAuth 및 RBAC

`internal/oauth/` 모듈과 `internal/permissions/`로 OAuth 2.0 기반 인증 및 역할 기반 접근 제어(RBAC)를 구현한다.

### 5.4 OS 키체인 자격증명 (go-keyring)

`github.com/zalando/go-keyring`를 통해 API 키를 OS 네이티브 키체인(macOS Keychain, Windows Credential Manager, Linux Secret Service)에 저장한다.

### 5.5 Tailscale VPN 통합

`tailscale.com v1.94.2` 및 `cmd/gateway_tsnet.go`를 통해 에이전트 게이트웨이를 Tailscale 메시 네트워크에 참가시킨다. 공용 인터넷 노출 없이 VPN 전용 접속이 가능하며, 이는 기존 Claw 생태계 전체에서 유일한 VPN-native 네트워킹 방식이다.

---

## 6. 채널 아키텍처 (7채널)

| 채널 | 라이브러리 | 특이사항 |
|------|-----------|---------|
| Telegram | telego v1.6.0 | 봇 API |
| Discord | discordgo v0.29.0 | 슬래시 명령 지원 |
| Slack | slack-go v0.19.0 | 앱 소켓 모드 |
| Feishu/Lark | (내장) | 기업용 |
| Zalo | (내장) | 베트남 메신저 |
| WhatsApp | (내장) | Twilio 또는 공식 API |
| WebSocket | gorilla/websocket | REST/웹 프론트엔드 |

---

## 7. 에이전트 팀 & 오케스트레이션

`cmd/gateway_agents.go` (275줄)와 `internal/tasks/`는 에이전트 팀 구성을 위한 공유 태스크 보드를 제공한다. 태스크에 `blocked_by` 필드를 선언함으로써 에이전트 간 의존성을 DAG(방향 비순환 그래프)로 정의할 수 있다.

```
에이전트 팀 구성:
  Agent A (리서치) ──blocked_by──> Agent B (데이터 수집)
  Agent C (요약) ──blocked_by──> Agent A
  => 자동 실행 순서: B -> A -> C
```

이 선언적 의존성 그래프 방식은 NemoClaw의 파이프라인 체인, OpenJarvis의 서브에이전트 호출과는 다른 접근이다.

---

## 8. 관찰 가능성 (OTel)

`cmd/gateway_otel.go`와 `internal/tracing/`에서 OpenTelemetry v1.40.0을 완전히 통합한다. gRPC(`otlptracegrpc`)와 HTTP(`otlptracehttp`) 양쪽 내보내기를 지원하며, `docker-compose.otel.yml`로 로컬 Jaeger/Tempo 환경을 즉시 구성할 수 있다.

---

## 9. 추가 기능

| 기능 | 구현 파일 | 설명 |
|------|----------|------|
| 브라우저 자동화 | go-rod v0.116.2 | 헤드리스 Chrome 제어 |
| 데스크톱 앱 | Wails v2 + ui/desktop/ | 크로스플랫폼 GUI |
| Cron 스케줄링 | gronx + cmd/cron_cmd.go | 크론 표현식 태스크 |
| TTS | internal/tts/ | 음성 출력 |
| 지식 그래프 | internal/knowledgegraph/ | 엔티티-관계 저장소 |
| MCP 프로토콜 | internal/mcp/ | 외부 도구 연동 |
| i18n | internal/i18n/ | 다국어 지원 |

---

## 10. 신규 패턴 (R-번호)

**R38: 3축(Mode/Access/Scope) 샌드박스 아키텍처**
구현: GoClaw `internal/sandbox/sandbox.go`
원리: Docker 샌드박스를 단일 on/off 스위치가 아닌 세 개의 독립 축으로 설계한다 — (1) 어느 에이전트를 격리할지(Mode), (2) 파일시스템 권한(Access), (3) 컨테이너 재사용 범위(Scope). 조합 수: 3 x 3 x 3 = 27가지 보안 프로파일.
시사점: 개발(non-main + rw + shared) vs 운영(all + none + session) 환경을 같은 코드베이스에서 설정만으로 전환 가능.

**R39: VPN-native 에이전트 게이트웨이 (Tailscale tsnet)**
구현: GoClaw `cmd/gateway_tsnet.go`, `tailscale.com v1.94.2`
원리: 에이전트 게이트웨이 자체를 Tailscale 노드로 등록해 공용 인터넷 없이 조직 내부 VPN 메시에서만 접근 가능하도록 한다. 기존 프레임워크들이 방화벽/리버스 프록시에 의존하는 것과 달리, 네트워크 격리가 에이전트 런타임에 내장된다.
시사점: Kubernetes 없이도 Zero Trust 네트워킹 가능; 팀 배포에서 인증서 관리 불필요.

---

## 11. 비교 테이블

| 항목 | GoClaw | OpenClaw (Python) | NemoClaw |
|------|--------|-------------------|----------|
| 언어 | Go 1.26 | Python 3.x | Python 3.x |
| 멀티테넌트 | PostgreSQL 완전 격리 | 없음 (단일 사용자) | 없음 |
| 샌드박스 | Docker (3축 제어) | 없음 | Docker (단순) |
| 암호화 | AES 내장 | 없음 | 없음 |
| VPN 통합 | Tailscale 내장 | 없음 | 없음 |
| OTel | 코어 의존성 | 없음 | 없음 |
| 라이선스 | CC BY-NC 4.0 | MIT | Apache-2.0 |
| 채널 수 | 7 | 5 | 3 |
| 데스크톱 UI | Wails v2 | 없음 | 없음 |

---

## 12. 한계

- **라이선스 제약**: CC BY-NC 4.0으로 상업적 사용 불가 — 기업 도입 시 상용 라이선스 협의 필요
- **Go 생태계**: AI/ML 라이브러리가 Python에 비해 부족 — PyTorch/HuggingFace 직접 연동 불가
- **Stars 대비 커뮤니티 규모**: 1.4k stars는 AgentScope(22k), CoPaw(13.6k)에 비해 작아 생태계 플러그인이 적다
- **WASM 부재**: IronClaw처럼 WASM 기반 코드 실행 격리는 없음 (Docker 전용)

---

## 13. 참고 링크

- GitHub: https://github.com/nextlevelbuilder/goclaw
- 문서: https://docs.goclaw.sh
- 관련 보고서: `reports/repos/details/openclaw_report.md`, `reports/repos/details/ironclaw_report.md`
- 보안 비교: `reports/repos/security_report.md`
