[배경]
내 생각에 결국 claw류의 핵심은 이거야.

deepagents(claude code/codex-cli/gemini-cli 등)류의 planning(tasks) + tool caling(특히 list/read/write file) 기능을 갖추고, skill을 통해 절차적 지식을 progressive disclosure 방식으로 로딩할 수 있는 대화형 에이전트가 있어.

이걸 24시간 띄워놓고, 메신저 API(telegram)를 연결해서 터미널이 아니라 메신저로 언제 어디서나 말을 걸 수 있게 해. 

이게 다야.

[문제]
보통은 '에이전트에게 실세계 권한을 안전하게 주는가'가 주된 쟁점인데, 그거말고 다른 쟁점이 있어.

바로 세션과 컨텍스트 관리 전략이야.

그냥 auto-compaction으로 놔두어도 대화 자체는 지속이 가능해.

그렇지만 이를테면 메일을 읽고, 첨부파일 내용을 확인하고, 기존 일정을 확인해서 주간 계획을 세운 뒤, 나에게 알려줘야 한다고 생각해봐.

이때 메일을 읽고, 파일을 읽는 등의 작업은 별도의 컨텍스트로 분리되어야 정확도와 속도, 효율성을 유지할 수 있어.

또 리서치 요청을 받고, 실행계획을 세운 뒤, 적절한 작업용 디렉토리를 생성해서 필요한 문서들을 잘 정리한 뒤, 분석노트를 작성하고, 추가조사 자료 수집 계획을 세우고, 필요하면 데이터를 받아서 통계분석을 하거나 코드를 작성해서 시뮬레이션을 돌린다고 생각해봐. 이런 작업은 반드시 적절한 서브에이전트 혹은 Agent Team(Claude Code의 TeamCreate tool)을 통해 복수의 분리된 컨텍스트를 가지는 에이전트가 있어야지만 제대로 작동해. 

우리가 보통 coding agent를 가지고 뭔가 작업을 할 때는 디렉토리를 하나 만들어서 해당 경로에서 작업을 격리된 상태로 시작하지. 거기서도 터미널을 여러개 띄워서 작업하고.

[지시]
각 claw agent runtime들이 이 문제를 각자 어떻게 풀었는지 다각적으로 코드를 검토해서 조사해줘. 

TeamCreate tool을 이용해 Agent Team을 구성해.

반드시 각 에이전트마다 별도의 하나 이상의 에이전트를 배정해서 병렬로 빠르고 깊이 있게 조사해야해.

[조사 결과]
7개 구현체(OpenClaw, Nanobot, NanoClaw, IronClaw, ZeroClaw, PicoClaw, TinyClaw)의 소스코드를 병렬 분석했다. 상세 분석은 session_context_report.md 참조.

핵심 발견 3가지:

1. **3가지 격리 아키타입으로 수렴**
   - 프로세스/컨테이너 격리: NanoClaw, TinyClaw (실제 별도 프로세스나 컨테이너)
   - 세션 키 기반 논리적 격리: OpenClaw, Nanobot, PicoClaw, ZeroClaw (같은 프로세스 안에서 ID로 분리)
   - 보안 계층 기반 격리: IronClaw

2. **전부 기본적으로 단일 세션 단일 스레드**
   서브에이전트 스폰 기능을 가진 구현체(OpenClaw, Nanobot, NanoClaw, TinyClaw, IronClaw)가 있긴 하지만, "이 작업은 별도 컨텍스트로 분리해야 한다"는 판단을 프레임워크 레벨에서 자동으로 내리는 곳은 없다. 전부 LLM의 도구 호출 판단에 맡긴다.

3. **"프로젝트 세션" 추상화가 없다**
   작업 디렉토리 생성 → 오케스트레이터 스폰 → 서브에이전트 동적 할당 → 결과 수집 → 아카이브라는 프로젝트 수명주기를 프레임워크 레벨에서 관리하는 구현체가 없다. 이건 아직 아무도 안 만든 층위다.

[논의]

**논의 1: "빠진 기능"이 아니라 "안 넣은 기능"이다**

NanoClaw는 이미 `allowedTools`에 TeamCreate, SendMessage 등을 전부 열어놓고 있다. 기술적으로 서브에이전트 오케스트레이션을 막는 게 없다. 결국 시스템 프롬프트에 "복잡한 작업이 오면 TeamCreate로 팀을 구성해라"는 절차적 지식, 즉 skill을 넣으면 된다. idea.md에서 말한 "skill을 통해 절차적 지식을 progressive disclosure 방식으로 로딩"이 정확히 이 지점을 가리키고 있었다. 프레임워크 코드의 문제가 아니라 시스템 프롬프트(=소프트웨어)의 문제다.

**논의 2: bash에서 claude를 직접 호출하는 방법**

모든 claw 프레임워크가 bash tool을 쓸 수 있다. 따라서 bash로 특정 디렉토리에 cd한 뒤, claude를 headless + skip-permission으로 실행하고 프롬프트에 TeamCreate 사용 지시를 담으면 된다. 기술적으로 완전히 가능하고 실제로 작동한다. 다만 트레이드오프가 있다: 비용 폭발(서브에이전트마다 풀 시스템 프롬프트 반복 로딩), Anthropic API 종속성 심화, 프로세스 제어권 상실. 이건 "기술적 불가능"이 아니라 "비용과 제어권의 트레이드오프"다.

**논의 3: Karpathy의 연구 조직 실험**

Karpathy가 트윗에서 공개한 셋업: 에이전트 8개(Claude 4개 + Codex 4개), GPU 1개씩, nanochat 실험. 설계 선택들이 의미심장하다.

- Docker/VM 안 쓰고 git worktree로 격리 → 컨테이너 오버헤드 없이 파일시스템 격리
- 에이전트 간 통신을 파일로 → 가장 단순한 방법이 가장 안정적
- tmux -p 안 쓰고 인터랙티브 세션 유지 → 사람이 언제든 takeover 가능
- tmux 그리드 자체가 대시보드 겸 인터페이스

그리고 핵심 발견: "에이전트는 잘 정의된 아이디어를 구현하는 건 잘하지만, 창의적 실험을 설계하는 건 못한다." 이건 프레임워크나 skill의 문제가 아니라 현재 LLM의 근본적 능력 한계다. 비전은 "조직을 프로그래밍한다" — 소스코드가 프롬프트 + 스킬 + 도구 + 프로세스(데일리 스탠드업 같은 것들)의 조합이 되는 것.

**논의 4: 대시보드와 모니터링**

7개 중 TinyClaw만 대시보드를 가지고 있다(TUI + TinyOffice 웹 UI). Karpathy는 tmux 자체가 대시보드이자 takeover 인터페이스다. 나머지는 전부 메신저나 CLI로만 결과를 확인한다. 24시간 돌아가는 에이전트에서 뭔가 잘못될 때 어떻게 개입할 것인가의 문제는 아직 대부분의 구현체에서 풀리지 않았다.

**논의 5: 실세계 권한 부여 — 보안 전략 코드 기반 조사 결과** (2026-03-05, 상세 분석: security_report.md)

7개 구현체의 보안/권한 코드를 병렬 분석한 결과, 4단계 보안 성숙도 계층이 드러났다:
- **Tier 1 (Defense-in-Depth)**: IronClaw(AES-256-GCM 볼트+WASM+SafetyLayer 4중방어), ZeroClaw(ChaCha20-Poly1305+E-Stop+PromptGuard)
- **Tier 2 (Container-First)**: NanoClaw(stdin 시크릿+mount-security), OpenClaw(도구 deny/allow+환경변수 위생)
- **Tier 3 (Denylist-Based)**: Nanobot(정규식 차단), PicoClaw(77개 패턴+os.Root 샌드박스)
- **Tier 4 (Minimal)**: TinyClaw(`--dangerously-skip-permissions` 항상 사용)

핵심 발견: 암호화 볼트(2/7), HITL 도구 승인(3/7), 프롬프트 인젝션 전용 방어(2/7)를 구현한 곳이 소수. "에이전트에게 실세계 권한을 안전하게 주는 문제"는 IronClaw/ZeroClaw도 "사전 정의된 도구 목록"에 대한 보안이지 "임의의 실세계 액션"에 대한 보안이 아니다. 논의 1의 "빠진 기능이 아니라 안 넣은 기능" 테제는 부분적으로만 맞다 — IronClaw의 Zero-Exposure 프록시나 ZeroClaw의 E-Stop+OTP는 시스템 프롬프트 추가로 해결할 수 없는 아키텍처 수준 설계.

[열린 질문]

아직 답이 없는 것들:

1. **프로젝트 수명주기 관리자는 누가 되어야 하는가?** 오케스트레이터(메인 에이전트)가 직접 디렉토리를 만들고 서브에이전트를 스폰하는 게 맞는가, 아니면 이걸 별도 레이어(프레임워크 레벨)로 올려야 하는가?

2. **컨텍스트 분리의 기준은 어디인가?** "이 작업은 별도 컨텍스트가 필요하다"는 판단을 LLM에 맡기면 일관성이 없다. 명시적 규칙(토큰 수, 작업 유형)으로 강제할 수 있는가?

3. **Karpathy의 "실험 설계 못함" 문제를 skill로 보완할 수 있는가?** 실험 설계 절차를 skill로 만들어서 매번 로딩하면 어느 정도까지 커버되는가, 아니면 근본적으로 다른 접근이 필요한가?

4. **메신저 인터페이스와 takeover의 결합** Karpathy의 tmux takeover 개념을 텔레그램 기반 시스템에서 어떻게 구현하는가? 에이전트가 막혔을 때 사람이 개입할 수 있는 최소한의 인터페이스는 뭔가?

5. **동적 도구 위험도 평가**: 새로 등록되는 MCP 도구의 위험도를 자동 분류하고 적절한 승인 수준을 할당하는 메커니즘이 가능한가?

6. **비용 하드 한도의 보편화**: ZeroClaw만 일별 $5 한도를 구현. 24시간 에이전트에서 비용 폭발은 현실적 위험인데, 왜 다른 구현체는 이를 무시하는가?

7. **프롬프트 인젝션의 적응형 방어**: 알려진 패턴만 탐지하는 현재 접근의 한계를 어떻게 넘는가?

8. **E-Stop의 메신저 통합**: ZeroClaw의 E-Stop(4단계+OTP)을 Telegram 기반 시스템에서 구현하면 어떤 형태가 되는가?

**논의 6: 브라우저 자동화와 액션/도구 아키텍처** (2026-03-05, 상세 분석: browser_actions_report.md)

7개 구현체의 브라우저+도구 코드를 병렬 분석한 결과:

- **브라우저 자동화 4개만 보유**: OpenClaw(50+ 기능 풀스택), ZeroClaw(3 백엔드 교체 가능), NanoClaw(호스트 실행 안티탐지), IronClaw(테스트 전용). 나머지 3개는 없음.
- **도구 정의 3철학**: 타입 기반(Rust/Go trait), 스키마 기반(JSON Schema), 문서 기반(SKILL.md). 보안과 확장성이 반비례.
- **MCP가 사실상 표준**: 5/7이 MCP 래핑 패턴을 독립적으로 구현. 외부 도구 통합의 공통 프로토콜.
- **PicoClaw만 도구 병렬 실행 지원** (goroutine + WaitGroup). 나머지 6개는 전부 순차 실행.
- **IronClaw의 Zero-Exposure 크레덴셜 모델이 유일하게 도구 코드 버그에도 시크릿 안전**. 프록시 레이어에서 URL 패턴 매칭 -> 헤더 주입. WASM 도구는 secret-exists()만 호출 가능.
- **NanoClaw의 호스트 실행 브라우저가 독창적**: X가 자동화를 탐지/차단하므로 컨테이너가 아닌 호스트에서 실제 Chrome을 실행. IPC 파일 폴링으로 컨테이너 에이전트와 통신.

열린 질문 5번(동적 도구 위험도 평가)에 대해: IronClaw의 ApprovalRequirement가 가장 근접하지만 수동 설정. MCP 도구 자동 분류는 아직 없음. 열린 질문 8번(E-Stop 메신저 통합)에 대해: NanoClaw의 IPC 워처에 E-Stop 로직을 삽입하면 "텔레그램 /estop -> 모든 pending 작업 거부"가 아키텍처적으로 가능.

9. **도구 병렬 실행의 부재**: PicoClaw 외 6개가 순차 실행. 복잡한 멀티스텝 작업(메일 읽기+일정 확인+계획 수립)에서 도구 병렬 실행이 없으면 레이턴시가 선형 누적. 이걸 프레임워크 레벨에서 풀어야 하는가?

10. **브라우저 보안과 기능의 트레이드오프**: OpenClaw(50+ 기능, SSRF 방어) vs NanoClaw(6개 X 액션, 안티탐지). "만능 브라우저 도구"와 "특화된 플랫폼 자동화" 중 24시간 에이전트에 더 적합한 것은?

**논의 7: 기억 아키텍처 -- 중기/장기 기억의 설계 선택** (2026-03-05, 상세 분석: memory_architecture_report.md)

7개 구현체의 기억 시스템 코드를 병렬 분석한 결과, 3단계 기억 성숙도가 드러났다:
- **Tier 1 (Full Memory Stack)**: IronClaw(pgvector+RRF), OpenClaw(sqlite-vec+LanceDB+Weighted+Decay+MMR), ZeroClaw(SQLite+FTS5+linear fusion+Soul Snapshot)
- **Tier 2 (Structured Markdown)**: Nanobot(MEMORY.md+HISTORY.md, LLM consolidation), PicoClaw(MEMORY.md+3일 daily notes, mtime 캐시)
- **Tier 3 (Delegation/None)**: NanoClaw(CLAUDE.md 위임), TinyClaw(write-only 아카이브)

핵심 발견: "이중 주입 경로"(MEMORY.md 항상 로드 + DB 검색 동적 주입)가 Tier 1 공통 패턴. ZeroClaw의 Soul Snapshot(brain.db -> MEMORY_SNAPSHOT.md -> Git -> cold-boot 복원)이 유일한 자아 연속성 구현. 기억 보호 메커니즘이 보안 성숙도와 상관관계(security_report.md Tier 1 = memory Tier 1). 최적 조합 제안: ZeroClaw의 Soul Snapshot + OpenClaw의 하이브리드 검색 + IronClaw의 정체성 보호.

11. **기억 consolidation의 최적 주기는?** Nanobot(100 메시지), ZeroClaw(12h), OpenClaw(5초 debounce) -- 어떤 리듬이 24시간 에이전트에 맞는가?

12. **벡터 검색 vs FTS의 실전 recall 비교**: 메신저 대화 맥락에서 어느 쪽이 실제로 더 유용한 기억을 찾아오는가?

13. **Soul Snapshot의 "기억 롤백"은 실용적인가?** git 버전 관리를 활용해 특정 시점의 기억 상태로 에이전트를 되돌리는 것.

14. **기억 오염 방지의 근본적 해법은?** IronClaw의 하드코딩 보호는 특정 파일만 보호. 나머지 기억은 LLM 판단에 의존하는 한계.

15. **멀티에이전트 환경에서 기억 공유 vs 격리의 경계는?** IronClaw의 agent_id 칼럼 격리 vs NanoClaw의 컨테이너 격리 -- 서브에이전트가 메인 에이전트의 기억에 접근해야 하는가?