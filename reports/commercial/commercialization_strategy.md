# AI 에이전트 상용화 전략 비교 (Commercialization Strategy Comparison)

**분석 대상**: Tencent QClaw, Tencent WorkBuddy, Tencent WeChat AI Agent, Xiaomi MiClaw, Nvidia NemoClaw, **Crew.you (AI3), Perplexity Personal Computer, Genspark Claw**
**작성일**: March 17, 2026
**목적**: 각 서비스의 아키텍처, 기능성, 보안, 상용화 전략을 체계적으로 비교 (중국 5개 + 글로벌 3개 = 8개 서비스)

---

## 1. 핵심 속성 비교표

### 8개 서비스가 보여주는 시장의 분화

2026년 AI 에이전트 상용화 시장은 더 이상 중국 내수 경쟁만으로 설명할 수 없다. Crew.you, Perplexity Personal Computer, Genspark Claw의 등장은 글로벌 AI 에이전트 시장이 본격적인 상용화 국면에 진입했음을 보여준다. 중국 5개 서비스가 WeChat이라는 단일 메신저 생태계를 중심으로 분화했다면, 글로벌 3개 서비스는 각각 전혀 다른 접근법 — 멀티 메신저 SaaS(Crew.you), 하드웨어 번들(Perplexity), 전용 클라우드 인스턴스(Genspark) — 을 택했다.

주목할 점은 가격 전략의 극명한 차이다. 중국 서비스들은 생태계 보조금(WeChat Agent 무료, WorkBuddy 포함, MiClaw 사전설치)으로 사용자를 확보하는 반면, 글로벌 서비스들은 독립 SaaS 과금 모델($20-$99/월)을 채택했다. Perplexity는 한 발 더 나아가 하드웨어($500+)와 소프트웨어를 묶는 Apple식 전략을 시도한다. 이는 AI 에이전트가 단순 소프트웨어를 넘어 하드웨어-소프트웨어 통합 제품으로 진화하고 있음을 시사한다.

배포 상태 역시 흥미롭다. Genspark Claw는 2026년 3월 12일 GA 출시 후 11개월 만에 $200M ARR을 달성하며 가장 빠른 상용화 속도를 보이고 있다. 반면 Perplexity는 웨이트리스트 단계로 하드웨어 공급망 제약이 출시 속도를 결정하는 새로운 변수를 만들어냈다.

| 속성 | QClaw | WorkBuddy | WeChat Agent | MiClaw | NemoClaw | Crew.you | Perplexity PC | Genspark Claw |
|------|-------|-----------|--------------|--------|----------|----------|---------------|---------------|
| **상태** | Beta (internal) | GA (launched 2026-03-10) | In development (Q3 2026 target) | Closed beta | Alpha v0.1.0 (정식 출시, 2026-03-17) | GA | Waitlist (Max subscribers priority) | GA (launched 2026-03-12) |
| **회사** | Tencent | Tencent | Tencent | Xiaomi | Nvidia | AI3 (2026) | Perplexity AI | Genspark.ai (Mainfunc Inc.), $1.6B valuation |
| **배포 모델** | Cloud (Tencent Lighthouse) | Fully managed SaaS | Fully managed SaaS | Device-resident (hybrid) | OpenClaw Plugin (sandboxed container) | Cloud SaaS (OpenClaw-based hosted) | M4 Mac mini hardware + cloud hybrid (24/7) | Dedicated cloud instance per user |
| **주 플랫폼** | WeChat + QQ + Tencent Docs | WeChat Work | WeChat (consumer) | Xiaomi devices + IoT | REST API (customer-integrated) | Slack, Teams, Discord, Telegram, WhatsApp, LINE, KakaoTalk (7+) | Comet browser (proprietary AI browser) | WhatsApp, Telegram, Teams, Slack |
| **설정 난이도** | 1-click (Lighthouse) | Zero (built-in) | Zero (built-in) | Zero (pre-installed) | High (infrastructure setup) | Low (SaaS signup) | Medium (hardware setup) | Low (SaaS signup) |
| **월 가격** | RMB 49-500 (cloud infra) | Included in WeChat Work | Free (in WeChat) | Free (pre-installed) | TBD (likely Capex + Opex) | Free($0/2K) → Basic($20) → Plus($49) → Pro($99) | TBD ($500+ hardware bundle) | Free($0) → Plus($24.99) → Claw($39.99/mo) |
| **대상 사용자** | SMB (1,000-100,000 users) | Enterprise (50M users) | Consumer (1.4B users) | Consumer (Xiaomi ecosystem) | Developer / GPU-infra 보유 기업 | Global SMB/Prosumer (messenger-first) | Knowledge workers, analysts, enterprise (global) | Global prosumer/SMB |
| **자율성 수준** | Supervised execution | ReadOnly suggestions | Supervised execution | Full autonomy | Full autonomy (configurable) | Supervised → Full Autopilot (14 autonomous modules) | Goal-oriented execution (approval-gated) | Full (AI Employee concept) |
| **지원 메신저** | WeChat, QQ, Tencent Docs | WeChat Work | WeChat | Voice, SMS, WeChat | None (customer integration) | Slack, Teams, Discord, Telegram, WhatsApp, LINE, KakaoTalk | Comet browser only | WhatsApp, Telegram, Teams, Slack |
| **오프라인 기능** | 5% | 0% | 0% | 80% | 100% (on-premise) | 0% | Partial (local model) | 0% |
| **24/7 가동** | Yes (Lighthouse) | Yes (managed) | Yes (managed) | Partial (device dependent) | Yes (customer managed) | Yes (cloud) | Yes (always-on hardware) | Yes (dedicated instance) |
| **최대 동시 사용자** | 50-unlimited (CVM) | Unlimited (Tencent-scaled) | Unlimited | 1 per device | Unlimited (multi-GPU) | Unlimited (multi-tenant) | 1 per device | 1 per instance (dedicated) |

---

## 2. 아키텍처 비교

### 배포 전략의 세 가지 패러다임

AI 에이전트 상용화의 배포 전략은 크게 세 가지 패러다임으로 수렴하고 있다. 첫째, Tencent의 QClaw나 WorkBuddy처럼 기존 메신저 생태계에 깊이 통합하는 **플랫폼 네이티브** 접근이다. 이 모델은 사용자 획득 비용을 극도로 낮추지만, 플랫폼 종속성이라는 대가를 치른다. 둘째, Crew.you나 Genspark Claw처럼 **크로스플랫폼 SaaS**로 다수의 메신저와 도구를 연결하는 접근이다. 이 모델은 유연성이 높지만, 각 플랫폼과의 통합 깊이가 네이티브에 비해 얕을 수밖에 없다.

셋째, Perplexity Personal Computer가 제시하는 **하드웨어-소프트웨어 통합** 패러다임은 가장 급진적이다. M4 Mac mini에 전용 AI 브라우저(Comet)를 탑재하여 로컬 추론과 클라우드 추론을 결합한다. 이는 Apple의 전략을 AI 에이전트 도메인에 적용한 것으로, 하드웨어 제어를 통해 응답 지연(< 100ms 로컬)과 프라이버시를 동시에 달성하려는 시도다.

데이터 거주지(data residency) 측면에서도 패러다임이 갈린다. 중국 서비스 4종은 모두 China-only(PIPL 필수)인 반면, Crew.you와 Genspark은 글로벌 클라우드, Perplexity는 로컬+클라우드 하이브리드, NemoClaw는 고객 선택이다. Genspark의 "사용자별 전용 인스턴스" 모델은 프라이버시와 성능을 동시에 추구하는 새로운 접근으로 주목된다.

| 아키텍처 측면 | QClaw | WorkBuddy | WeChat Agent | MiClaw | NemoClaw | Crew.you | Perplexity PC | Genspark Claw |
|-------------|-------|-----------|--------------|--------|----------|----------|---------------|---------------|
| **클라우드 vs 로컬** | Cloud-only (Tencent Cloud) | Cloud-only (WeChat backend) | Cloud-only (WeChat backend) | Device-first (optional cloud) | OpenClaw Plugin (sandboxed container, GPU inference routing) | Cloud SaaS (OpenClaw-based) | Hybrid (M4 local + cloud) | Dedicated cloud instance per user |
| **메신저 통합** | API webhook (외부) | Native backend (내부) | Native backend (내부) | Device OS native | 무통합 (API 제공만) | Multi-messenger API (7+ platforms) | Comet browser (proprietary) | Multi-messenger API (4 platforms) |
| **LLM 위치** | Cloud inference | Cloud inference | Cloud inference | On-device + cloud fallback | Customer's GPU cluster | Cloud inference (multi-model) | Local + cloud hybrid | Dedicated cloud (multi-model) |
| **상태 저장소** | Tencent CosDB/TOS | WeChat backend | WeChat Cloud Drive | Device storage + Mi Cloud | Customer's DB | Cloud (persistent memory) | Local + cloud sync | Dedicated instance storage |
| **에코시스템** | Tencent Cloud APIs (8+) | WeChat Work native (6+) | Mini Programs (500-1000 of 3.8M) | Xiaomi IoT (200M devices) | OpenClaw plugin API (sandbox security + GPU inference routing) | 50+ tool integrations (Gmail, Calendar, Drive, GitHub, Notion, etc.) | 500+ enterprise integrations + premium data (Statista, CB Insights, PitchBook) | Google Workspace, Outlook, Slack, Teams, Notion, Salesforce, X + Chrome extension |
| **데이터 거주지** | China (Tencent servers) | China (Tencent servers) | China (Tencent servers) | Device (customer controls) | On-premise (customer controls) | Global cloud (multi-region) | Local device + global cloud | Global cloud (per-user isolation) |

---

## 3. 자율성 & 권한 비교

### 자율성 스펙트럼: ReadOnly에서 Full Autopilot까지

8개 서비스의 자율성 수준은 놀라울 정도로 다양하다. WorkBuddy의 ReadOnly(제안만)에서 Crew.you의 14개 자율 모듈, 멀티데이 미션 수행까지, 자율성의 스펙트럼이 극단에서 극단으로 펼쳐져 있다. 이 차이는 단순한 기술 역량의 차이가 아니라 **시장 전략의 차이**에서 비롯된다.

중국 서비스들은 규제(PIPL, 중국 AI 규제법)와 기업 고객의 보수성을 고려해 대체로 Supervised 또는 ReadOnly를 택했다. 반면 글로벌 서비스들은 "AI Employee"(Genspark), "Full Autopilot"(Crew.you)처럼 공격적인 자율성을 마케팅 핵심으로 내세운다. Perplexity는 중간 지점으로, 목표 기반 실행이되 승인 게이트를 두어 신뢰를 구축하는 전략이다.

안전장치 설계에서도 철학적 차이가 드러난다. Tencent는 플랫폼 수준의 감사 로그(30-365일)에 의존하고, Crew.you는 PII 자동 마스킹과 사용자별 샌드박스를 적용하며, Perplexity는 CrowdStrike 파트너십과 킬 스위치라는 물리적 안전장치까지 갖추었다. Genspark의 프라이버시 바이 아이솔레이션(사용자별 전용 인스턴스)은 아키텍처 수준에서 격리를 보장하는 새로운 접근이다.

| 차원 | QClaw | WorkBuddy | WeChat Agent | MiClaw | NemoClaw | Crew.you | Perplexity PC | Genspark Claw |
|------|-------|-----------|--------------|--------|----------|----------|---------------|---------------|
| **계획 수립** | Full | No | Full | Full | Full | Full (14 autonomous modules) | Full (goal-oriented) | Full (AI Employee) |
| **도구 선택** | Supervised | Full | Full | Full | Full | Full (50+ tools) | Full (500+ integrations) | Full (Workspace 3.0) |
| **실행** | Conditional (파일 쓰기는 확인 필요) | None (제안만) | Conditional (첫 실행 확인) | Full | Conditional (정책 기반) | Supervised → Full Autopilot | Conditional (approval-gated) | Full (AI Employee) |
| **복구** | Full | N/A | Full | Full | Full | Full | Full | Full |
| **안전장치** | Tencent audit logs | WeChat Work logs | WeChat logs + user perms | Device TEE + biometric | Customer vault + RBAC | Per-user sandbox + PII auto-masking | Kill switch + CrowdStrike + sandboxed queries | Privacy-by-isolation + sandbox + whitelist |
| **감사 기록** | 30-365일 보관 | Org admin 제어 | 180일 기본값 | 7일 (로컬) | SIEM 통합 | SOC 2 Type II audit trails | Audit trails (SOC 2 Type II) | Sandbox audit logs |

---

## 4. 기능성 비교

### 기능 격차가 만드는 시장 기회

기능성 비교에서 가장 두드러지는 차이는 **도구 통합 범위**다. 중국 서비스들이 Tencent/Xiaomi 자체 생태계 도구(WeChat Pay, Tencent Meeting, Xiaomi IoT)에 집중하는 반면, Crew.you는 50개 이상, Perplexity는 500개 이상의 써드파티 도구를 통합한다. 이는 중국 시장의 "벽으로 둘러싸인 정원(walled garden)" 전략과 글로벌 시장의 "개방형 생태계" 전략의 근본적 차이를 반영한다.

Perplexity의 프리미엄 데이터 접근(Statista, CB Insights, PitchBook, SEC, FactSet)과 40개 이상의 금융 도구(Plaid, Coinbase, Polymarket)는 다른 서비스에서 찾아볼 수 없는 고유한 강점이다. 이는 Perplexity가 일반 소비자가 아닌 지식 노동자와 분석가를 명확히 타겟팅하고 있음을 보여준다.

멀티 메신저 지원에서는 Crew.you가 7개 이상의 플랫폼(Slack, Teams, Discord, Telegram, WhatsApp, LINE, KakaoTalk)으로 압도적이다. 특히 LINE(일본)과 KakaoTalk(한국)을 포함한 것은 아시아 시장 공략 의지를 보여준다. Genspark Claw는 4개 메신저를 지원하며, 크롬 확장과 Speakly(iOS/Android) 앱으로 접점을 확장한다.

| 기능 | QClaw | WorkBuddy | WeChat Agent | MiClaw | NemoClaw | Crew.you | Perplexity PC | Genspark Claw |
|------|-------|-----------|--------------|--------|----------|----------|---------------|---------------|
| **음성 입력** | 계획중 | 지원 (필사만) | 지원 (응답은 미정) | 지원 | 없음 (REST API만) | 지원 | 지원 (Comet browser) | 지원 (Speakly app) |
| **음성 응답** | 없음 | 없음 | Q4 2026 계획 | 없음 | 없음 | 없음 | 지원 | 없음 |
| **이미지 처리** | 없음 | 없음 | 없음 | 카메라 (AR 기반) | 가능 (RAG) | 없음 | 지원 (multimodal) | 지원 |
| **파일 접근** | TOS | Tencent Docs | WeChat Cloud Drive | Device storage | 고객 데이터 소스 | Google Drive, Outlook, Notion, etc. | Local files + cloud | Google Workspace, Outlook, Notion, Salesforce |
| **스마트홈 제어** | 없음 | 없음 | 없음 | Xiaomi IoT (200M 장치) | 없음 (고객 통합) | 없음 | 없음 | 없음 |
| **결제 통합** | WeChat Pay | 없음 | WeChat Pay | WeChat Pay / Xiaomi Pay | 없음 (고객 통합) | 없음 | Plaid, Coinbase, Polymarket (40+ financial tools) | 없음 |
| **일정/회의** | Tencent Meeting | Tencent Meeting (베타) | Tencent Meeting | 없음 | Outlook/Workday (커넥터) | Google Calendar, Outlook | 지원 (500+ integrations) | Scheduling + meeting bot |
| **멀티 메신저** | Yes (4+) | No (WeChat Work만) | No (WeChat만) | Limited (WeChat relay) | No (고객 통합) | Yes (7+ platforms) | No (Comet browser만) | Yes (4 platforms) |
| **프리미엄 데이터** | 없음 | 없음 | 없음 | 없음 | 없음 | 없음 | Statista, CB Insights, PitchBook, SEC, FactSet | 없음 |
| **코드 배포** | 없음 | 없음 | 없음 | 없음 | 가능 (developer tool) | GitHub 통합 | Sandbox (API) | 지원 (code deploy) |

---

## 5. 보안 & 규정 준수

### 보안 철학의 세 가지 축: 플랫폼 신뢰, 사용자 격리, 하드웨어 제어

보안 설계에서 8개 서비스는 세 가지 철학으로 나뉜다. 첫째, Tencent 서비스들(QClaw, WorkBuddy, WeChat Agent)은 **플랫폼 수준 보안**에 의존한다. WeChat의 E2EE, Tencent Cloud의 KMS 등 기존 인프라를 활용하되, 에이전트 고유의 보안 메커니즘은 상대적으로 미약하다. 이는 이미 검증된 보안 인프라 위에 구축한다는 장점이 있지만, 에이전트의 자율적 행동에 대한 세밀한 통제가 부족할 수 있다.

둘째, Crew.you와 Genspark Claw는 **사용자 수준 격리**를 핵심으로 한다. Crew.you의 per-user sandbox와 PII 자동 마스킹, Genspark의 privacy-by-isolation(사용자별 전용 인스턴스)은 멀티테넌트 환경에서 데이터 유출 위험을 최소화한다. 특히 Crew.you는 Google CASA 인증, SOC 2 Type II, GDPR을 모두 획득하여 글로벌 규정 준수에서 가장 앞서 있다.

셋째, Perplexity와 MiClaw는 **하드웨어 기반 보안**을 추구한다. Perplexity의 킬 스위치와 CrowdStrike 파트너십, MiClaw의 TEE(Trusted Execution Environment)와 생체인식은 소프트웨어만으로는 달성할 수 없는 물리적 보안 계층을 제공한다. GDPR vs PIPL의 규제 차이도 중요하다. 중국 서비스는 PIPL 준수가 필수이고, 글로벌 서비스는 GDPR + SOC 2가 기본 요건이 되었다.

| 보안 차원 | QClaw | WorkBuddy | WeChat Agent | MiClaw | NemoClaw | Crew.you | Perplexity PC | Genspark Claw |
|----------|-------|-----------|--------------|--------|----------|----------|---------------|---------------|
| **인증** | Tencent Cloud 2FA | WeChat Work SSO | WeChat 2FA | 생체인식 + 암호 | SAML 2.0 / LDAP | OAuth 2.0 (multi-platform) | SAML SSO | OAuth 2.0 |
| **자격증명 저장** | Tencent KMS (AES-256) | WeChat 암호화 | WeChat 암호화 | TEE (하드웨어 키) | 고객 Vault | AES-256-GCM | CrowdStrike partnership | Sandbox isolation |
| **데이터 암호화** | 전송중: TLS; 저장시: AES-256 | 전송중: E2EE; 저장시: WeChat | 전송중: E2EE; 저장시: WeChat | 전송중: TLS; 저장시: TEE | 고객 정책 | 전송중: TLS; 저장시: AES-256-GCM | 전송중: TLS; 저장시: local encryption | 전송중: TLS; 저장시: instance-level encryption |
| **감사 로그** | 30-365일 | WeChat 감사 추적 | 180일 기본 | 7일 로컬 | SIEM 통합 | SOC 2 Type II audit trails | Audit trails | Sandbox audit logs |
| **GDPR/PIPL** | PIPL 준수 | PIPL 준수 | PIPL 준수 | PIPL 준수 (기본값) | 고객 책임 | GDPR + SOC 2 Type II | SOC 2 Type II | 프라이버시 바이 아이솔레이션 |
| **SOC2 준비** | 자동 보고 (계획) | 아니오 | 아니오 | 부분 | 예 (권장) | SOC 2 Type II 인증 완료 | SOC 2 Type II | 미공개 |
| **데이터 거주지** | China-only | China-only | China-only | Device (고객 선택) | On-premise (고객) | Global (multi-region) | Local device + global cloud | Global (per-user isolation) |
| **PII 보호** | Tencent 정책 | WeChat 정책 | WeChat 정책 | Device TEE | 고객 정책 | PII 자동 마스킹 | Sandboxed queries | Whitelist + sandbox |

---

## 6. 시장 포지셔닝 & 경쟁

### 시장의 재편: 중국 내수에서 글로벌 경쟁으로

글로벌 3개 서비스의 등장으로 AI 에이전트 시장의 경쟁 구도가 근본적으로 변화하고 있다. 중국 서비스들이 Tencent/Xiaomi 생태계 안에서 세그먼트별로 분화(소비자/SMB/엔터프라이즈)했다면, 글로벌 서비스들은 **사용 패턴**을 기준으로 시장을 재정의한다: 멀티 메신저 중심 업무(Crew.you), 심층 리서치 및 분석(Perplexity), AI 기반 완전 자동화(Genspark).

Genspark Claw의 $200M ARR(11개월)과 $1.6B 밸류에이션은 AI 에이전트 시장의 상업적 잠재력을 입증했다. 이는 Crew.you와 Perplexity에게도 공격적 확장의 근거를 제공한다. 반면 중국 서비스들은 WeChat 생태계라는 해자(moat) 덕에 중국 내수에서는 견고하지만, 글로벌 확장 가능성은 제한적이다.

생태계 잠금(lock-in) 수준에서도 명확한 패턴이 보인다. WeChat Agent(매우 높음)에서 NemoClaw(낮음)까지의 스펙트럼에, Crew.you(낮음, 멀티 메신저/멀티 도구)와 Perplexity(높음, 하드웨어 종속)가 추가되어 사용자의 선택 폭이 넓어졌다.

| 측면 | QClaw | WorkBuddy | WeChat Agent | MiClaw | NemoClaw | Crew.you | Perplexity PC | Genspark Claw |
|------|-------|-----------|--------------|--------|----------|----------|---------------|---------------|
| **목표 시장** | 중국 SMB | 중국 엔터프라이즈 | 중국 소비자 | 중국 + 동남아 소비자 | 개발자 / GPU 보유 기업 (글로벌) | 글로벌 SMB/Prosumer | 글로벌 지식 노동자/분석가/엔터프라이즈 | 글로벌 Prosumer/SMB |
| **주 경쟁사** | OpenClaw, DuClaw | DingTalk AI, Feishu AI | QClaw | Google Assistant | OpenClaw plugin 생태계 | OpenClaw (self-hosted), Genspark Claw | Microsoft Copilot PC, Apple Intelligence | Crew.you, OpenClaw |
| **주요 강점** | 원클릭 배포 + Tencent ecosystem | 네이티브 WeChat Work 통합 | 1.4B 사용자 기반 | 프라이버시 + IoT | 샌드박스 보안 + GPU 추론 라우팅 (오픈소스) | 최다 메신저 지원 (7+), Proactive Intelligence | 하드웨어+소프트웨어 번들, 프리미엄 데이터, <100ms 로컬 | 전용 클라우드 인스턴스, AI Employee, 최속 ARR 성장 |
| **주요 약점** | 신뢰성 미검증 (베타) | 자율성 없음 | 규제 불확실 | 기기 종속 (Xiaomi) | Alpha 단계, 엔터프라이즈 커넥터 미포함 | 자체 LLM 없음 (의존) | 하드웨어 종속, 높은 초기 비용 | 전용 인스턴스 확장 비용 |
| **가격 경쟁력** | 저가 (RMB 49/월) | 포함 (WeChat Work) | 무료 (포함) | 무료 (사전설치) | 오픈소스 (무료), GPU 인프라 별도 | 프리미엄 ($20-$99/월) | 프리미엄 ($500+ 하드웨어) | 중간 ($0-$39.99/월) |
| **생태계 잠금** | 높음 (Tencent Cloud) | 높음 (WeChat Work) | 매우 높음 (WeChat) | 높음 (Xiaomi) | 낮음 (API만) | 낮음 (멀티 메신저, 멀티 도구) | 높음 (하드웨어 종속) | 중간 (멀티플랫폼이나 전용 클라우드) |
| **국제화** | 중국 중심 | 중국 중심 | 중국 중심 | 제한적 (동남아) | 글로벌 | 글로벌 (14개 언어) | 글로벌 | 글로벌 |

---

## 7. 기술 스펙 요약

### 성능 지표가 말해주는 설계 철학

응답 지연(latency) 데이터는 각 서비스의 설계 철학을 압축적으로 보여준다. MiClaw(<50ms 로컬)과 Perplexity(<100ms 로컬)는 온디바이스 추론으로 극한의 저지연을 달성하지만, 이는 하드웨어 제약이라는 대가와 함께 온다. Tencent의 클라우드 서비스들은 200ms-3s 범위로 네트워크 지연이 불가피하며, Crew.you(<1s 클라우드)와 Genspark(<500ms 전용 인스턴스)는 클라우드 최적화로 중간 지점을 잡았다.

컨텍스트 윈도우에서도 흥미로운 분화가 보인다. WorkBuddy(50 메시지)처럼 의도적으로 짧은 컨텍스트를 유지하는 서비스부터, Crew.you(무제한, persistent memory)처럼 대화 간 기억을 영구 유지하는 서비스까지 다양하다. Crew.you의 크로스 대화 영속적 메모리는 "개인 비서"로서의 연속성을 제공하며, 이는 단발성 대화 중심의 중국 서비스들과 차별화되는 핵심 기능이다.

LLM 선택에서 글로벌 서비스들은 멀티모델 전략(Claude Opus 4.6, GPT-5.4 등)을 택한 반면, 중국 서비스들은 Hunyuan(Tencent) 또는 Qwen(Xiaomi) 중심이다. NemoClaw만이 NeMo + HuggingFace 호환으로 두 세계를 연결한다.

| 기술 스펙 | QClaw | WorkBuddy | WeChat Agent | MiClaw | NemoClaw | Crew.you | Perplexity PC | Genspark Claw |
|----------|-------|-----------|--------------|--------|----------|----------|---------------|---------------|
| **응답 지연** | <2s (WeChat), <3s (QQ) | <500ms (p50), <2s (p99) | <200ms (agent) + 가변 | <50ms (로컬), <500ms (cloud) | <50ms (로컬 GPU) | <1s (cloud) | <100ms (local) + variable (cloud) | <500ms (dedicated instance) |
| **메모리 풋프린트** | 256MB + 512MB/대화 | N/A (관리형) | N/A (관리형) | 256-512MB | 40-80GB (모델 크기) | N/A (관리형) | Variable (M4 Mac mini) | N/A (dedicated instance) |
| **저장소 풋프린트** | 최소 5GB | N/A | N/A | 200-300MB | 고객 선택 | N/A (cloud) | Local SSD + cloud | N/A (dedicated instance) |
| **최대 동시** | 50-unlimited | Unlimited | Unlimited | 1 (기기당) | 100-500 (GPU당) | Unlimited (multi-tenant) | 1 (device당) | 1 (instance당) |
| **컨텍스트 윈도우** | 10,000 메시지 | 50 메시지 + 문서 | 30 메시지 + 선호도 | 5,000 메시지 | 무제한 (고객) | 무제한 (persistent memory) | Variable (local + cloud) | Variable (dedicated) |
| **지원 LLM** | Hunyuan, GPT-4, Claude, Qwen | Hunyuan (주), 제3자 | Hunyuan (주), 제3자 | Qwen-7B (기기), cloud fallback | NeMo + HF compatible | Multi-model (OpenClaw compatible) | Proprietary (Perplexity models) | Multi-model (Claude Opus 4.6, GPT-5.4, Nemotron 3 Super) |
| **배포 제약** | Tencent Cloud 한정 | 내부 (확장 불가) | 내부 (확장 불가) | Xiaomi 기기 한정 | 무제한 (고객 GPU) | 없음 (global cloud) | M4 Mac mini 필수 | 없음 (global cloud) |

---

## 8. 배포 & 비용 비교

### 비용 구조의 근본적 차이: 생태계 보조금 vs 독립 과금

비용 비교에서 가장 눈에 띄는 패턴은 **생태계 보조금 모델 vs 독립 과금 모델**의 대비다. Tencent의 WorkBuddy와 WeChat Agent는 사실상 무료이며, 이는 WeChat 플랫폼의 광고 수익과 결제 수수료가 에이전트 서비스를 보조하는 구조다. Xiaomi의 MiClaw도 하드웨어 판매로 소프트웨어 비용을 흡수한다.

반면 Crew.you($20-$99/월)와 Genspark($24.99-$39.99/월)는 에이전트 서비스 자체로 수익을 창출해야 하는 독립 SaaS 모델이다. 흥미로운 점은 이 두 서비스 모두 무료 티어를 제공하여 프리미엄(freemium) 전략을 취한다는 것이다. Perplexity는 하드웨어 번들로 한 번에 $500+ 이상의 초기 투자를 요구하여 가장 높은 진입 장벽을 가진다.

100명 사용자 기준 연간 운영비를 비교하면, WeChat Agent/WorkBuddy(무료) → MiClaw(무료) → QClaw(RMB 600-100K+) → Genspark($0-$48K) → Crew.you($0-$119K) → Perplexity(하드웨어 $50K+ + 구독) 순으로, 가격 스펙트럼이 매우 넓다.

### 초기 설정 비용

| 항목 | QClaw | WorkBuddy | WeChat Agent | MiClaw | NemoClaw | Crew.you | Perplexity PC | Genspark Claw |
|------|-------|-----------|--------------|--------|----------|----------|---------------|---------------|
| **초기 설정비** | 무료 (1-click) | 무료 (built-in) | 무료 (built-in) | 무료 (사전설치) | 무료 (오픈소스), GPU 인프라 별도 | 무료 (SaaS signup) | $500+ (hardware bundle) | 무료 (SaaS signup) |
| **기관 관리자 교육** | 필요 | 필요 | 필요 | 필요 없음 | 필요 (개발자 수준) | 최소 (직관적 UI) | 필요 (Comet browser 학습) | 최소 (직관적 UI) |
| **인프라 구성** | 5분 (Lighthouse 1-click) | 0분 (즉시 사용) | 0분 (즉시 사용) | 0분 (사전설치) | 시간 단위 (plugin 설치 + GPU 설정) | 5분 (계정 생성 + 메신저 연결) | 30분+ (hardware unboxing + setup) | 5분 (계정 생성 + 메신저 연결) |

### 연간 운영 비용 (100명 사용자 기준)

| 시나리오 | QClaw | WorkBuddy | WeChat Agent | MiClaw | NemoClaw | Crew.you | Perplexity PC | Genspark Claw |
|---------|-------|-----------|--------------|--------|----------|----------|---------------|---------------|
| **가벼운 사용** (월 1,000 요청) | RMB 600 | 포함 | 무료 | 무료 | 무료 (오픈소스) + GPU Opex | $0 (Free tier 2K credits) | TBD (hardware amortized) | $0 (Free tier) |
| **중간 사용** (월 50,000 요청) | RMB 5,000 | 포함 | 무료 | 무료 | 무료 (오픈소스) + GPU Opex | ~$24K/yr (Basic $20×100) | TBD | ~$30K/yr (Plus $24.99×100) |
| **대량 사용** (월 1,000,000 요청) | RMB 100K+ | 포함 | 무료 | 무료 | 무료 (오픈소스) + GPU Opex (규모 경제) | ~$119K/yr (Pro $99×100) | TBD | ~$48K/yr (Claw $39.99×100) |

**결론**: 중국 서비스는 생태계 보조금으로 무료/저가; 글로벌 SaaS는 독립 과금($20-$99/월); Perplexity는 하드웨어 번들로 최고 초기비용. Genspark은 대량 사용 시 Crew.you 대비 60% 저렴.

---

## 9. 글로벌 vs 중국 시장 전략 비교

### 두 개의 세계: 규제, 생태계, 가격의 구조적 차이

AI 에이전트 시장은 사실상 두 개의 별개 시장으로 작동하고 있다. 중국 시장과 글로벌 시장은 규제 환경, 메신저 생태계, 가격 전략에서 근본적으로 다르며, 이 차이가 각 서비스의 전략을 결정짓는다.

**생태계 잠금 전략의 차이**: 중국 서비스들(Tencent, Xiaomi)은 자사 플랫폼에 대한 깊은 통합을 통해 사용자를 잠금(lock-in)하는 전략이다. WeChat Agent는 13억 사용자의 WeChat에 내장되어 전환 비용이 사실상 무한대에 가깝다. 반면 글로벌 서비스들은 멀티플랫폼 접근(Crew.you 7+, Genspark 4개 메신저)으로 전환 비용을 낮추고, 서비스 품질 자체로 사용자를 유지해야 한다. Perplexity만이 하드웨어 종속이라는 높은 전환 비용 전략을 택했다.

**규제 환경**: 중국 PIPL(개인정보보호법)은 데이터의 중국 내 저장을 요구하며, AI 서비스에 대한 사전 심사가 필수다. WeChat Agent의 Q3 2026 출시 지연도 이 규제 승인 과정에 기인한다. 반면 GDPR(EU)은 사용자 동의와 투명성에 초점을 맞추며, SOC 2 Type II가 B2B 시장의 사실상 표준이 되었다. Crew.you가 Google CASA + SOC 2 + GDPR을 모두 획득한 것은 글로벌 규정 준수의 높은 기준을 반영한다.

**메신저 전략**: 중국에서 WeChat은 사실상 유일한 소비자 메신저이며, 이것이 Tencent에 절대적 우위를 부여한다. 글로벌 시장에서는 Slack(업무), WhatsApp(개인), Discord(커뮤니티), Telegram(개인/커뮤니티), LINE(일본), KakaoTalk(한국) 등 메신저가 분산되어 있어, 멀티 메신저 지원이 핵심 경쟁력이 된다.

| 비교 차원 | 중국 (QClaw/WorkBuddy/WeChat Agent/MiClaw) | 글로벌 SaaS (Crew.you/Genspark Claw) | 하드웨어 번들 (Perplexity PC) | 오픈소스 (NemoClaw) |
|----------|------------------------------------------|--------------------------------------|------------------------------|-------------------|
| **생태계 잠금** | 매우 높음 (WeChat/Xiaomi 독점) | 낮음-중간 (멀티플랫폼) | 높음 (하드웨어 종속) | 낮음 (API 기반) |
| **규제 프레임워크** | PIPL (데이터 중국 내 저장 필수) | GDPR + SOC 2 Type II | SOC 2 Type II + SAML SSO | 고객 책임 |
| **메신저 전략** | WeChat 독점 (단일 플랫폼) | 멀티 메신저 (4-7개) | 전용 브라우저 (Comet) | 고객 통합 |
| **가격 철학** | 생태계 보조금 (무료/포함) | 독립 SaaS 과금 (freemium) | 하드웨어 번들 (고가 일회성) | 오픈소스 + 인프라 비용 |
| **사용자 획득** | 기존 플랫폼 사용자 전환 (CAC ≈ 0) | 마케팅 + freemium + viral | 하드웨어 판매 채널 | 개발자 커뮤니티 |
| **국제화 가능성** | 제한적 (중국 내수 중심) | 높음 (글로벌 네이티브) | 높음 (하드웨어 글로벌 유통) | 높음 (오픈소스) |
| **데이터 주권** | China-only | Multi-region | Local + cloud | 고객 선택 |
| **수익 구조** | 간접 (플랫폼 수수료, 광고) | 직접 (구독, 사용량) | 직접 (하드웨어 + 구독) | 간접 (GPU 인프라 판매) |

### 중국-글로벌 교차 경쟁 가능성

중국 서비스의 글로벌 진출과 글로벌 서비스의 중국 진입 가능성은 모두 낮다. 중국 서비스는 WeChat 의존성과 PIPL 제약으로 해외 확장이 어렵고, 글로벌 서비스는 GFW(Great Firewall)와 중국 AI 규제로 진입이 사실상 불가능하다. 유일한 예외는 NemoClaw(오픈소스)과 OpenClaw 기반 서비스(Crew.you)로, 이들은 기술적으로 양쪽 시장에서 모두 배포 가능하지만, 실질적으로 규제 장벽이 존재한다.

---

## 10. 시장 점유율 & 경쟁 각도

### 2026년 시장 세분화

```
AI 에이전트 시장 (2026 글로벌 + 중국):

├─ 소비자 (Consumer):
│  ├─ 중국:
│  │  └─ WeChat AI Agent (Q3-Q4 2026) - 독점 위치
│  │  └─ MiClaw (2027) - 틈새 (프라이버시/IoT)
│  │  └─ Baidu DuClaw - 경량 사용자
│  └─ 글로벌:
│     └─ Crew.you - 멀티 메신저 개인 비서
│     └─ Genspark Claw - AI Employee (자동화 중심)
│     └─ Perplexity PC - 지식 노동자/분석가
│
├─ SMB (1K-100K 직원):
│  ├─ 중국:
│  │  └─ QClaw - Tencent ecosystem 고착 사용자
│  │  └─ OpenClaw - DIY / 저비용 우선 사용자
│  │  └─ DuClaw SaaS - 다목적 사용자
│  └─ 글로벌:
│     └─ Crew.you Pro - 팀 협업 자동화
│     └─ Genspark Claw - 업무 자동화
│     └─ OpenClaw self-hosted - DIY / 저비용
│
├─ 엔터프라이즈 (100K+ 직원):
│  ├─ 중국:
│  │  └─ WorkBuddy - Tencent ecosystem 사용자
│  │  └─ DingTalk AI (Alibaba) - Aliyun 사용자
│  │  └─ Feishu AI (ByteDance) - ByteDance 생태계
│  └─ 글로벌:
│     └─ Perplexity PC - 분석/리서치 팀
│     └─ Microsoft Copilot - M365 사용자
│     └─ Salesforce Einstein - CRM 중심
│
└─ 규제 산업 (Finance, Healthcare, Gov) / 개발자:
   ├─ 중국/글로벌:
   │  └─ NemoClaw - OpenClaw Plugin (Alpha), 샌드박스 보안 + GPU 라우팅
   │  └─ OpenClaw self-hosted - DIY + 저비용
   └─ 글로벌:
      └─ Perplexity PC - 프리미엄 데이터 접근 (금융)
```

### 상호 경쟁 매트릭스 (8×8)

| vs | QClaw | WorkBuddy | WeChat Agent | MiClaw | NemoClaw | Crew.you | Perplexity PC | Genspark Claw |
|----|-------|-----------|--------------|--------|----------|----------|---------------|---------------|
| **QClaw** | - | Low (다른 대상) | High (둘다 WeChat 소비자) | Low (기기 vs 클라우드) | Low (다른 대상) | Medium (SMB 겹침) | Low (다른 시장) | Medium (SMB 겹침) |
| **WorkBuddy** | Low | - | Low (엔터프라이즈 vs 소비자) | Low | Low | Low (다른 시장) | Low | Low |
| **WeChat Agent** | High | Low | - | Medium (모두 무료) | Low | Low (다른 시장/국가) | Low | Low |
| **MiClaw** | Low | Low | Medium | - | Low | Low (다른 시장) | Low | Low |
| **NemoClaw** | Low | Low | Low | Low | - | Low | Low | Low |
| **Crew.you** | Medium | Low | Low | Low | Low | - | Medium (글로벌 지식 노동자) | High (글로벌 SMB/Prosumer) |
| **Perplexity PC** | Low | Low | Low | Low | Low | Medium | - | Medium (글로벌 지식 노동자) |
| **Genspark Claw** | Medium | Low | Low | Low | Low | High | Medium | - |

**경쟁 강도 해석**:
- High = 같은 사용자 기반, 유사한 가격대, 같은 지역
- Medium = 부분적 겹침 (가격 또는 사용자 기반 또는 지역)
- Low = 명확히 다른 시장 세그먼트 또는 지역

**핵심 경쟁 축**:
- **중국 내수**: QClaw vs WeChat Agent (Tencent 자체 경쟁 지속)
- **글로벌 SMB/Prosumer**: Crew.you vs Genspark Claw (직접 경쟁, 최고 강도)
- **글로벌 지식 노동자**: Perplexity PC vs Crew.you vs Genspark Claw (3파전)
- **개발자/규제 산업**: NemoClaw (경쟁자 부재, 오픈소스 독자 포지션)

---

## 11. 시장 영향 & 전략적 함의

### Tencent의 자체 경쟁 (내부 카니발리즘)

| 제품 | 대상 | WeChat Agent와의 충돌 |
|------|------|----------------------|
| **QClaw** | SMB | High - 소비자가 무료 에이전트 받으면 SMB는 왜 유료? |
| **WorkBuddy** | 엔터프라이즈 | Low - 다른 에코시스템 (WeChat Work vs personal) |
| **WeChat Agent** | 소비자 | High - 자체 제품 간 판매 충돌 (무료가 유료 QClaw 시장 침범) |

**해결책** (Tencent의 가능한 전략):
- QClaw -> 전문화된 에이전트 마켓플레이스로 전환
- WeChat Agent -> 기본 소비자 작업 (쇼핑, 결제, 알림)
- WorkBuddy -> 엔터프라이즈 지능형 검색 (자율성 없음)

### Alibaba & Baidu의 응전

| 회사 | 제품 | Tencent에 대한 대응 |
|------|------|---------------------|
| **Alibaba** | DingTalk AI | DingTalk의 엔터프라이즈 깊이로 WorkBuddy 출옥 |
| **Baidu** | DuClaw | 가격 공세 (RMB 17.8/월 = 싸게) + 클라우드-불가지론 |
| **ByteDance** | Feishu AI | 젊은 사용자층 타겟 + Douyin consumer agent |

### 글로벌 플레이어의 중국 진출

| 회사 | 전략 | 중국 진출 | Tencent와의 경쟁 |
|------|------|----------|------------------|
| **Google** | Gemini Agent | WeChat integration 불가 (GFW) | No |
| **OpenAI** | ChatGPT + Agent | 공식적 중국 서비스 없음 | No |
| **Anthropic** | Claude Agent | 중국 미진출 | No |
| **AI3 (Crew.you)** | Multi-messenger SaaS | 중국 진출 불가 (GFW, 메신저 차단) | No |
| **Perplexity** | Hardware + AI browser | 중국 진출 어려움 (규제) | No |
| **Genspark** | Dedicated cloud | 중국 진출 불가 (GFW) | No |

**결론**: 중국과 글로벌 시장은 사실상 분리되어 운영됨. Tencent는 중국 소비자/SMB 시장에서 독점적 지위를 유지하고, 글로벌 시장은 Crew.you, Perplexity, Genspark 3파전 + 빅테크(Microsoft, Google) 경쟁 구도.

---

## 12. 최종 권장사항 & 선택 기준

### 사용자별 추천

| 사용자 프로필 | 추천 제품 | 이유 |
|------------|---------|------|
| **개인 (중국)** | WeChat AI Agent | 무료, 내장, 1.4B 사용자 기반 |
| **Xiaomi 사용자** | MiClaw | 프라이버시, 오프라인, IoT 통합 |
| **SMB (중국)** | QClaw 또는 DuClaw | 원클릭 배포, Tencent ecosystem 또는 가격 |
| **엔터프라이즈 (중국)** | WorkBuddy 또는 DingTalk AI | 네이티브 통합, compliance, 규모 |
| **개발자 / GPU 보유 팀** | NemoClaw | 오픈소스, 샌드박스 보안, GPU 추론 라우팅 (Alpha v0.1.0) |
| **개발자 (DIY)** | OpenClaw | 오픈소스, 전체 제어, 비용 절감 |
| **비용 민감 (엔터프라이즈)** | DuClaw (Baidu) | RMB 17.8/월 - 가장 싼 SaaS |
| **글로벌 멀티 메신저 사용자** | Crew.you | 7+ 메신저, 50+ 도구, Proactive Intelligence, 14개 언어 |
| **리서치/분석 중심** | Perplexity PC | 프리미엄 데이터, 500+ 통합, <100ms 로컬 추론 |
| **글로벌 SMB (자동화 중심)** | Genspark Claw | AI Employee, 전용 인스턴스, $39.99/월 |
| **메신저 다양 + 저비용** | Crew.you Free/Basic | 무료 2K 크레딧 또는 $20/월 |
| **하드웨어 선호 + 프라이버시** | Perplexity PC | 로컬 추론, 킬 스위치, 하드웨어 격리 |

### 의사결정 트리

```
[구매 의사결정]

Q1: 중국 내 사용자인가?
  YES -> Q2 (중국 경로)
  NO -> Q6 (글로벌 경로)

--- 중국 경로 ---
Q2: GPU 인프라 보유 + 샌드박스 보안/GPU 라우팅 필요?
  YES -> NemoClaw (OpenClaw Plugin, Alpha v0.1.0, 오픈소스)
  NO -> Q3

Q3: 소비자 또는 SMB?
  소비자 -> WeChat AI Agent (무료)
  SMB -> Q4

Q4: Tencent ecosystem을 이미 사용 중인가?
  YES -> QClaw (1-click, 생태계 잠금)
  NO -> Q5

Q5: 최저 가격 추구?
  YES -> DuClaw (RMB 17.8/month)
  NO -> OpenClaw (오픈소스, 전체 제어)

--- 글로벌 경로 ---
Q6: 하드웨어 기반 솔루션 선호? (로컬 추론 + 프리미엄 데이터)
  YES -> Perplexity Personal Computer ($500+)
  NO -> Q7

Q7: 멀티 메신저 지원 필요? (3개 이상)
  YES -> Crew.you (7+ 메신저, $0-$99/월)
  NO -> Q8

Q8: 완전 자율 "AI Employee" 필요?
  YES -> Genspark Claw ($0-$39.99/월)
  NO -> Q9

Q9: 개발자 / 셀프 호스팅 선호?
  YES -> NemoClaw 또는 OpenClaw (오픈소스)
  NO -> Crew.you Basic ($20/월, 범용)
```

---

## 13. 미래 전망 (2026-2027)

### 예상 시장 진화

| 시기 | QClaw | WorkBuddy | WeChat Agent | MiClaw | NemoClaw | Crew.you | Perplexity PC | Genspark Claw |
|------|-------|-----------|--------------|--------|----------|----------|---------------|---------------|
| **Q3 2026** | Beta -> GA | 안정화 | 소비자 공개 베타 | 폐쇄 베타 | Alpha -> Beta (plugin 생태계 확장) | 글로벌 확장 (아시아 강화) | GA 출시 (하드웨어 양산) | $300M+ ARR, 엔터프라이즈 진출 |
| **Q4 2026** | SMB 확대 | 확대 사용 | 대규모 배포 (1억+ 사용자) | 공개 베타 | 커넥터 확대 + 엔터프라이즈 지원 추가 | 엔터프라이즈 tier 출시 | 글로벌 유통 확대 | 1,000개+ 기업 고객 |
| **Q1 2027** | 경쟁 압박 | 공고화 | 도미넌트 | 초기 채택 | GA (엔터프라이즈 파일럿 시작) | IPO 또는 대형 라운드 | 2세대 하드웨어 | Series C / IPO 검토 |

### 2026-2027 주요 예측

**Crew.you (AI3)**:
- 2026 Q3: LINE/KakaoTalk 통합 강화로 아시아 시장 공략 본격화. 14개 언어 → 20개 이상 확장
- 2026 Q4: Enterprise tier 출시로 B2B 시장 진입. SSO, 관리자 콘솔, SLA 추가
- 2027: OpenClaw 생태계의 "호스티드 레퍼런스"로 포지셔닝. 멀티 메신저 AI 에이전트 시장의 선두 유지 가능성 높음

**Perplexity Personal Computer**:
- 2026 Q3: 웨이트리스트 해소 후 GA 출시. 초기 반응은 하드웨어 품질과 Comet 브라우저 완성도에 따라 결정
- 2026 Q4: 프리미엄 데이터 파트너십 확대 (Bloomberg, Reuters 추가 가능). 금융/컨설팅 업계 타겟
- 2027: 2세대 하드웨어 출시. Apple Silicon 기반 커스텀 칩 가능성. 하드웨어 가격 $300대로 하락하면 대중화 가속

**Genspark Claw**:
- 2026 Q3: $200M ARR → $300M+ 성장 지속. "AI Employee" 컨셉의 엔터프라이즈 버전 출시
- 2026 Q4: Salesforce, HubSpot 등 CRM 딥 통합. 엔터프라이즈 파일럿 1,000개+ 달성
- 2027: Series C 또는 IPO 검토. 시장 밸류에이션 $5B+ 가능. 전용 인스턴스 모델의 확장 비용이 핵심 리스크

### 위험 요소

| 제품 | 주요 위험 | 확률 | 영향 |
|------|---------|-----|------|
| **QClaw** | 신뢰성 부족 (베타 상태) | High | Medium (SMB 채택 지연) |
| **WorkBuddy** | 자율성 부족 (정보 조회만) | Low | Low (설계 선택) |
| **WeChat Agent** | 규제 승인 지연 | High | High (Q3 2026 놓칠 수 있음) |
| **MiClaw** | Xiaomi 시장점유율 감소 | Medium | High (전체 시장 축소) |
| **NemoClaw** | Alpha 단계 성숙도 부족 (엔터프라이즈 커넥터 미포함) | Medium | Medium (개발자 채택 속도에 달림) |
| **Crew.you** | LLM 의존 리스크 (자체 모델 없음) | Medium | High (OpenClaw 모델 비용 변동) |
| **Perplexity PC** | 하드웨어 공급망 리스크 + 높은 초기 가격 | High | High (대중화 지연) |
| **Genspark Claw** | 전용 인스턴스 확장 비용 (unit economics) | Medium | High (마진 압박) |

---

## 14. 결론

### 핵심 비교

**배포 방식**:
- 클라우드 SaaS (플랫폼 네이티브): QClaw, WorkBuddy, WeChat Agent (Tencent 대부분)
- 기기 네이티브: MiClaw (Xiaomi)
- OpenClaw Plugin (sandboxed container): NemoClaw (Nvidia, Alpha v0.1.0)
- 크로스플랫폼 Cloud SaaS: Crew.you (AI3), Genspark Claw (Genspark.ai)
- 하드웨어-소프트웨어 번들: Perplexity Personal Computer (Perplexity AI)

**자율성**:
- Full Autopilot: Crew.you (14 autonomous modules), Genspark Claw (AI Employee)
- Full: MiClaw, NemoClaw
- Goal-oriented (approval-gated): Perplexity PC
- Supervised: QClaw, WeChat Agent
- ReadOnly: WorkBuddy

**시장 포지셔닝**:
- 중국 소비자 (무료): WeChat AI Agent (Tencent 독점)
- 중국 SMB (저가): QClaw, DuClaw (경쟁)
- 중국 엔터프라이즈: WorkBuddy, DingTalk AI (경쟁)
- 글로벌 SMB/Prosumer: Crew.you, Genspark Claw (직접 경쟁)
- 글로벌 지식 노동자: Perplexity PC (하드웨어 프리미엄)
- 개발자 / GPU 보유 기업: NemoClaw (OpenClaw Plugin, Alpha 오픈소스)

**2026년 승자 예측**:
1. **WeChat AI Agent** (Q3 2026 출시 성공시) - 1.4B 잠재 사용자, 중국 소비자 독점
2. **Genspark Claw** - $200M ARR (11개월), 가장 빠른 상용화 성공, "AI Employee" 포지셔닝
3. **Crew.you** - 최다 메신저 지원 (7+), Proactive Intelligence, 글로벌 SMB/Prosumer 선두
4. **WorkBuddy** - 50M WeChat Work 사용자, 무료
5. **Perplexity PC** - 하드웨어 번들이라는 새로운 카테고리 개척 (GA 출시 여부가 관건)
6. **QClaw** - Tencent Cloud 사용자 lock-in (하지만 자체 경쟁으로 압박)
7. **DuClaw (Baidu)** - 가격 경쟁력
8. **MiClaw** - 틈새 (프라이버시)
9. **NemoClaw** - Alpha 출시 완료 (2026-03-17), 개발자 채택 단계; 엔터프라이즈 GA는 2027년 이후

### 시장 구조적 전망

2026년 AI 에이전트 시장은 **중국 내수**(Tencent 독점)와 **글로벌 시장**(3파전 + 빅테크)으로 이원화되었다. 두 시장 간 교차 경쟁은 규제와 생태계 장벽으로 인해 거의 불가능하며, 각각 독립적으로 진화할 것이다.

글로벌 시장에서는 Crew.you(멀티 메신저 SaaS), Perplexity(하드웨어 번들), Genspark(전용 클라우드)라는 세 가지 완전히 다른 비즈니스 모델이 경쟁하며, 이는 시장이 아직 지배적 모델을 찾지 못했음을 시사한다. 2027년까지 이 세 모델 중 하나가 "표준"으로 수렴할 가능성이 높으며, Genspark의 ARR 성장 속도($200M/11개월)가 현재 가장 강력한 시장 신호다.

---

## Appendix: 참고 자료

**분석 근거**:
- idea5.md (compare_claws 프로젝트)
- 공개 뉴스 (Bloomberg, TechNode, Panda Daily, 36kr, TechCrunch, The Verge)
- 공시 및 보도 자료 (주가 움직임, 출시 발표, 투자 라운드)
- 아키텍처 추론 (공개 API 문서, 플랫폼 특성)
- NemoClaw 소스코드 직접 분석 (Alpha v0.1.0, 25,650 LOC, 2026-03-17 출시)
- Crew.you 공개 문서 및 가격표 (2026-03)
- Perplexity AI 공식 발표 및 웨이트리스트 정보 (2026-03)
- Genspark.ai IR 자료 (Series B $385M, $1.6B valuation, $200M ARR)

**심층 분석 보고서**:
- [`details/tencent_qclaw_analysis.md`](details/tencent_qclaw_analysis.md) — QClaw 아키텍처 및 상용화 분석
- [`details/tencent_workbuddy_analysis.md`](details/tencent_workbuddy_analysis.md) — WorkBuddy 엔터프라이즈 전략 분석
- [`details/tencent_wechat_agent_analysis.md`](details/tencent_wechat_agent_analysis.md) — WeChat AI Agent 소비자 전략 분석
- [`details/xiaomi_miclaw_analysis.md`](details/xiaomi_miclaw_analysis.md) — MiClaw 디바이스 전략 분석
- [`details/nvidia_nemoclaw_analysis.md`](details/nvidia_nemoclaw_analysis.md) — NemoClaw 포지셔닝 및 보안 분석 (출시 전 예측 기반)
- [`details/nvidia_nemoclaw_commercial.md`](details/nvidia_nemoclaw_commercial.md) — NemoClaw 상용화 전략 개별 분석
- [`../repos/details/nemoclaw_report.md`](../repos/details/nemoclaw_report.md) — NemoClaw 소스코드 기반 심층 분석 (Alpha v0.1.0 실제 구현)
- [`details/crew_you_analysis.md`](details/crew_you_analysis.md) — Crew.you (AI3) 멀티 메신저 전략 분석
- [`details/perplexity_personal_computer_analysis.md`](details/perplexity_personal_computer_analysis.md) — Perplexity Personal Computer 하드웨어-소프트웨어 번들 분석
- [`details/genspark_claw_analysis.md`](details/genspark_claw_analysis.md) — Genspark Claw AI Employee 전략 분석

> **포지셔닝 변화 주목**: 출시 전 분석은 NemoClaw를 Fortune 500 대상 온프로미스 엔터프라이즈 인프라 플레이로 예측했으나, 실제 Alpha는 OpenClaw Plugin 형태로 출시됨. 핵심 기능은 샌드박스 보안 + GPU 추론 라우팅이며, 50+ 엔터프라이즈 커넥터 대신 개발자 친화적 플러그인 아키텍처를 채택. 엔터프라이즈보다 개발자 생태계 먼저 공략하는 전략으로 해석됨.

**한계**:
- NemoClaw: Alpha 단계 (v0.1.0), 엔터프라이즈 GA 기능 미완성
- WeChat AI Agent: 규제 승인 불확실
- Crew.you: 자체 LLM 없음, OpenClaw 의존
- Perplexity PC: 미출시 (웨이트리스트), 하드웨어 사양/가격 미확정
- Genspark Claw: 전용 인스턴스 장기 unit economics 미검증
- 가격 정보: 공시되지 않은 부분 추정

**업데이트 필요**:
- WeChat AI Agent 출시 (Q3 2026)
- NemoClaw Beta 출시 (예상 Q3 2026)
- Xiaomi MiClaw 공개 베타 (예상 H2 2026)
- Perplexity PC GA 출시 (예상 Q3 2026)
- Crew.you Enterprise tier 출시 (예상 Q4 2026)
- Genspark Claw Series C / IPO (예상 2027)
- 규제 승인 뉴스 (중국 LLM 규제 동향, EU AI Act 적용)
