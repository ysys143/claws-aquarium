# Hermes Agent 단독 테스트 & 운영 계획 (상세)

> **작성일**: 2026-04-15
> **대상**: 사용자 개인 에이전트 시스템 — Outbound HITL 4-에이전트 파이프라인을 Hermes 단독으로 구축
> **선행 결정**: `project_user_personal_agent.md` (Hermes standalone, OCI 단일 노드)
> **선행 계획**: `hands_on_benchmark_plan.md` (범용 6-프레임워크 벤치마크), `personal_agent_setup_plan.md` (원 OpenClaw 설계, 역사 참조용)
> **Hermes 버전 타깃**: `NousResearch/hermes-agent` main (2026-04-14 기준)

---

## 목차

1. [목표와 성공 기준](#1-목표와-성공-기준)
2. [전제 조건 및 준비물](#2-전제-조건-및-준비물)
3. [Phase 0 — 용어 / 파일 레이아웃](#3-phase-0--용어--파일-레이아웃)
4. [Phase 1 — 단일 노드 부트스트랩 (1일)](#4-phase-1--단일-노드-부트스트랩-1일)
5. [Phase 2 — Monitoring Agent · T1 (3–5일)](#5-phase-2--monitoring-agent--t1-35일)
6. [Phase 3 — Writing / Publisher · T2 확장 (1주)](#6-phase-3--writing--publisher--t2-확장-1주)
7. [Phase 4 — Research Agent · T3-A/B (1–2주)](#7-phase-4--research-agent--t3-ab-12주)
8. [Phase 5 — LLM Wiki + Pluggable Memory (2주)](#8-phase-5--llm-wiki--pluggable-memory-2주)
9. [Phase 6 — Burst GPU · T3-C (선택, 1주)](#9-phase-6--burst-gpu--t3-c-선택-1주)
10. [Phase 7 — Self-Evolution (3–6개월 후)](#10-phase-7--self-evolution-36개월-후)
11. [검증 매트릭스](#11-검증-매트릭스)
12. [위험 & 대응](#12-위험--대응)
13. [체크포인트 일정](#13-체크포인트-일정)

---

## 1. 목표와 성공 기준

### 1.1 최종 목표

사용자의 4-에이전트 파이프라인(Monitoring / Research / Writing / Publisher)을 **Hermes Agent 단독**으로 OCI 단일 노드에 구축하고, **30일 이상 사람 개입 없이** Outbound HITL 사이클이 자동 순환하는 상태.

### 1.2 성공 기준 (Go 결정 기준)

| 지표 | 목표값 |
|------|--------|
| **Monitoring → Telegram 자동 알림 성공률** | ≥ 95% (30일) |
| **Cron 발송 시각 오차** | ≤ 5분 |
| **중복 알림** | 0건 |
| **"experiment" 명령에서 Research Agent 스폰까지** | ≤ 60초 |
| **24h 연속 무중단** | ≥ 3일 (최초), ≥ 7일 (안정화 후) |
| **Crash 후 자동 복구** | ≤ 5분 (systemd restart 정책) |
| **월 총 운영비** | ≤ $80 (OCI $0 + Claude API + 선택적 Modal burst) |

### 1.3 비목표 (Out of Scope)

- 멀티 디바이스 분산 (Gateway-Node) — `project_multi_device_analysis.md`에서 과잉으로 판정됨
- OCR 모델 **로컬 inference** 벤치마크 — Phase 6 burst GPU에서 선택적으로만
- 본 계획에서는 **Hermes Agent 하나만** 운영 — 6-프레임워크 비교는 `hands_on_benchmark_plan.md`에서 별도

---

## 2. 전제 조건 및 준비물

### 2.1 인프라

- **OCI Always Free ARM A1 인스턴스**: 4 OCPU / 24 GB RAM / 200 GB 스토리지, Ubuntu 22.04 LTS, Asia/Seoul 리전 우선 시도 (KST 레이턴시)
- 네트워크: 인바운드 22(SSH)만 허용, 나머지 차단. Outbound 무제한
- DNS/고정 IP 불필요 (Hermes는 아웃바운드 only)

### 2.2 계정 / API 키

| 항목 | 발급처 | 용도 |
|------|--------|------|
| **Anthropic API 키** | console.anthropic.com | Claude Sonnet 4.6 호출 |
| **Telegram Bot Token** | @BotFather `/newbot` | 봇 고유 토큰 |
| **Telegram User ID** | @userinfobot | 허용 사용자 ID |
| **Blog API 토큰** | 개인 블로그(티스토리/워드프레스/Ghost 등) | Publisher Agent 업로드용 |
| **Modal API 토큰** | modal.com (선택, Phase 6) | Burst GPU |
| **GitHub PAT** | (선택) | 레포 push, hermes-agent-self-evolution PR용 |

### 2.3 로컬 툴체인 (OCI 인스턴스에)

- Python 3.10+ (기본)
- Docker 28+ (Research Agent 실험 격리)
- git, curl, jq, tmux, screen

### 2.4 스펜딩 가드

- **Anthropic Console**: 월 $100 한도 설정
- **OCI Billing Alerts**: $1 트리거 (사실상 Free Tier 벗어남 감지)
- **Modal**: workspace $20/월 hard cap (Phase 6 진입 시)

---

## 3. Phase 0 — 용어 / 파일 레이아웃

### 3.1 Hermes 디렉토리

```
~/.hermes/
├── config.yaml              # 전역 설정 (provider, telegram, cron, memory)
├── .env                     # 비밀 (Telegram token, API keys, Blog tokens)
├── memories/
│   ├── MEMORY.md            # ~800 토큰, Frozen Snapshot 시스템 프롬프트에 포함 (R17)
│   └── USER.md              # ~500 토큰, 사용자 프로필 (R18)
├── sessions/                # session_db — 멀티 세션 기록, Self-Evolution 평가 소스
├── skills/                  # 커스텀 스킬 (번들 스킬 덮어쓰기 가능)
│   ├── monitoring/
│   │   └── SKILL.md
│   ├── research/
│   ├── writing/
│   └── publisher/
├── wiki/                    # LLM Wiki 지식 베이스 (R46)
│   ├── SCHEMA.md
│   ├── index.md
│   ├── log.md
│   ├── raw/                 # 불변 원본
│   ├── entities/
│   ├── concepts/
│   ├── comparisons/
│   └── queries/
└── logs/                    # 런타임 로그
```

### 3.2 에이전트 역할 매핑

| 파이프라인 에이전트 | Hermes 구현 | 핵심 스킬 |
|--------------------|-------------|-----------|
| **Monitoring** | 루트 세션 + Cron 트리거 | `blogwatcher` (번들) + 커스텀 `monitoring` |
| **Research** | 서브에이전트 (Bounded Delegation R21) | `arxiv`, `duckduckgo-search`, `domain-intel`, 커스텀 `research` (Tirith 검증 포함) |
| **Writing** | 서브에이전트 | `ml-paper-writing` (번들) + 커스텀 `writing` |
| **Publisher** | 서브에이전트 (블로그 API 호출 전용) | 커스텀 `publisher` (수동 승인 게이트) |

---

## 4. Phase 1 — 단일 노드 부트스트랩 (1일)

### 4.1 단계별 명령

```bash
# 1. OCI 인스턴스 SSH 접속 후
sudo apt update && sudo apt install -y python3.10 python3-pip docker.io jq tmux
sudo usermod -aG docker $USER && newgrp docker

# 2. Hermes 설치
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc

# 3. 프로바이더 + 도구 설정
hermes setup   # → Anthropic 선택, API 키 입력, 도구 기본값 수락
hermes doctor  # → 모든 체크 ✅ 확인

# 4. 첫 대화 검증 (인터랙티브)
hermes
❯ hello
```

### 4.2 검증 체크포인트

- [ ] `hermes doctor` 에러 0건
- [ ] `hermes` 인터랙티브 모드에서 "hello" 응답 3초 이내
- [ ] `~/.hermes/config.yaml` 존재, provider: anthropic 확인
- [ ] 디스크 사용량 <500MB, RAM <1GB

### 4.3 예상 함정

- **Anthropic OAuth 흐름이 OCI SSH 환경에서 실패**: 브라우저 팝업 불가. 해결 — API key 직접 입력 또는 로컬 PC에서 토큰 받아 복사
- **ARM 휠 미존재**: pip 일부 패키지가 ARM64 wheel 부재 시 source build 필요. 30분 추가 소요 가능

---

## 5. Phase 2 — Monitoring Agent · T1 (3–5일)

### 5.1 목표

매일 09:00 KST, Hacker News 상위 10개를 한국어 3줄 요약으로 Telegram 전송. 중복 제외. 30일 연속 운영 성공률 ≥95%.

### 5.2 Telegram 연동

```bash
hermes gateway setup
# → Telegram 선택
# → BotFather 토큰 입력
# → 허용 User ID 입력 (@userinfobot에서 조회한 숫자)
# → privacy mode 권고대로 OFF 처리 (그룹용, 개인 DM이면 무관)

# 양방향 테스트
hermes
❯ /send test from hermes    # 에이전트 → Telegram
# Telegram에서 봇에게 "ping" 전송 → 에이전트 응답 수신 확인
```

### 5.3 Monitoring 스킬 작성

파일: `~/.hermes/skills/monitoring/SKILL.md`

```markdown
---
name: monitoring
description: Scan HN front page every 6h, score novelty, dedupe by story ID, push notable stories to Telegram.
version: 0.1.0
author: user
license: private
metadata:
  hermes:
    tags: [hn, monitoring, outbound]
    category: productivity
    config:
      - key: monitoring.state_path
        description: Seen HN story IDs cache
        default: "~/.hermes/state/hn_seen.json"
      - key: monitoring.min_score
        description: HN minimum score threshold
        default: 150
---

# HN Monitoring Skill

When invoked, execute:

1. Fetch `https://hn.algolia.com/api/v1/search?tags=front_page` (top 30 stories).
2. Filter: `points >= ${monitoring.min_score}` AND NOT in `state_path` seen set.
3. For each unseen story:
   - Fetch URL content (or HN discussion if text post).
   - Claude Sonnet 4.6: "다음 기사를 한국어 3줄로 요약하고, 실험 가치가 있으면 [EXPERIMENT] 태그를 맨 앞에 붙여라."
   - Compose Telegram message: title + URL + 요약.
4. Send all to Telegram chat (origin chat of this skill invocation).
5. Append story IDs to `state_path`.
6. Log to `~/.hermes/logs/monitoring-$(date +%F).jsonl`.

## Deduplication

Story ID set in `state_path`. Persist to disk after each run. If corrupted, start fresh (accept one-time duplicate batch).

## HITL Trigger

If `[EXPERIMENT]` tag appears, user may reply `experiment #N` to activate Research Agent (see research skill).
```

### 5.4 Cron 등록

```bash
# Hermes CLI에서
hermes
❯ /cron add "0 9 * * *" "Run HN monitoring check" --skill monitoring --name "HN daily digest"
❯ /cron list   # 등록 확인
```

**스케줄 표현식 옵션**:
- `"0 9 * * *"` — 매일 09:00 KST (단 Hermes 기본 TZ가 UTC면 `0 0 * * *`)
- `"every 6h"` — 자연어도 수용 (cron.md 예시)

**타임존 확인**: `~/.hermes/config.yaml`에 `timezone: Asia/Seoul` 추가. 없으면 UTC로 동작해 9시간 어긋남.

### 5.5 검증 체크포인트

- [ ] `hermes` interactive로 수동 호출 시 정상 발송
- [ ] Cron 첫 실행 성공 (`~/.hermes/logs/cron-$(date +%F).log` 확인)
- [ ] 중복 탐지 — 동일 스토리 두 번 노출되지 않음
- [ ] 3–5일 연속 자동 발송 (사람 개입 0회)
- [ ] RAM idle <1.5 GB, peak <3 GB
- [ ] Telegram SendMessage API 호출 실패 시 다음 runs에 skip 없이 복구

### 5.6 실패 모드 인위 트리거 (런북 §9)

| 시나리오 | 트리거 | 기대 복구 동작 |
|---------|-------|---------------|
| HN API 다운 | `iptables -A OUTPUT -d news.ycombinator.com -j DROP` 5분 | 다음 cron 실행까지 대기, 재시도 |
| Anthropic rate limit | 의도적 대량 요청 | 지수 백오프, 실패 시 Telegram 알림 |
| 디스크 풀 | `dd if=/dev/zero of=/fill bs=1M count=10000` | 로그 쓰기 실패 graceful, 메시지 발송은 계속 |
| Telegram BotFather 토큰 revoke | 토큰 재생성 | 에이전트가 친절한 에러 메시지 (Tirith-style) |

---

## 6. Phase 3 — Writing / Publisher · T2 확장 (1주)

### 6.1 목표

Monitoring 알림에서 사용자가 "more on #3" 또는 "블로그 써줘 #3"라 답하면:
- **more on**: Writing Agent가 해당 기사를 재요약·심층 설명 (6–10문단)
- **블로그 써줘**: Writing Agent 초안 → 사용자 피드백 루프 → 최종 승인 → Publisher가 블로그 API 업로드

### 6.2 Writing 스킬

`~/.hermes/skills/writing/SKILL.md` — 스타일 시드(사용자 기존 블로그 글 3–5개 발췌 포함)로 문체 정렬. Anthropic prompt caching 혜택을 위해 **스타일 시드는 스킬 파일에 정적 embedding**. 한 번 로드되면 Frozen Snapshot(R17)에 포함.

### 6.3 Publisher 스킬

`~/.hermes/skills/publisher/SKILL.md`:

```markdown
---
name: publisher
description: Upload a finalized post to the user's blog via API. Requires explicit user approval.
version: 0.1.0
metadata:
  hermes:
    tags: [blog, publish, hitl-gate]
    trust_level: trusted   # R20 Skills Trust - NOT community/agent-created
    config:
      - key: publisher.api_endpoint
        default: https://mylab.example.com/api/posts
      - key: publisher.draft_dir
        default: ~/.hermes/drafts
---

# Publisher Skill

**HITL Gate**: This skill MUST ask user to confirm "PUBLISH #<draft_id>" before any HTTP POST. Never auto-publish.

Workflow:
1. List drafts in `draft_dir`, echo titles + IDs to Telegram.
2. Wait for explicit `PUBLISH #<id>` reply.
3. POST to `api_endpoint` with draft body + YAML frontmatter.
4. On 2xx: move draft to `draft_dir/posted/`.
5. Report final URL to Telegram.
6. On failure: keep draft, report error, do NOT retry automatically.
```

### 6.4 검증

- [ ] Writing 초안 첫 발송 <3분 (Claude API 왕복 포함)
- [ ] "서론 너무 길어" 피드백 루프 2회 이내 수렴
- [ ] Publisher는 명시적 `PUBLISH #N` 없으면 업로드 불가 (수동 확인)
- [ ] 블로그 업로드 후 URL 회신 성공

---

## 7. Phase 4 — Research Agent · T3-A/B (1–2주)

### 7.1 목표

Monitoring이 `[EXPERIMENT]` 태그 붙인 기사에 사용자가 "experiment #N"이라 답하면:
- Research Agent 서브에이전트 스폰 (Hermes R21 Bounded Delegation, MAX_DEPTH=2, MAX_CONCURRENT=3 제약 내)
- 기사에서 언급된 **외부 도구·모델·API** 추출
- **T3-A**: API 기반 비교 (Claude Vision / GPT-4V / GLM-4V 등) — 기본값
- **T3-B**: 로컬 경량 CPU 도구 (pymupdf, tesseract, easyocr) — 필요 시
- **T3-C는 Phase 6까지 보류**

### 7.2 Research 스킬 + Tirith 통합 (R22)

핵심: **외부 도구를 pip/npm으로 설치하기 전에 Tirith가 SHA-256 + cosign 검증**.

`~/.hermes/skills/research/SKILL.md`:

```markdown
---
name: research
description: Spawn Research subagent to run API-based or local-CPU benchmark on extracted tools from a news story.
version: 0.1.0
metadata:
  hermes:
    tags: [benchmark, research, hitl]
    trust_level: trusted
    tirith_required: true      # R22 pre-exec 스캔 필수
    requires_confirmation: true # 첫 호출 시 Telegram 승인
    config:
      - key: research.workspace_dir
        default: ~/.hermes/research/workspaces
      - key: research.max_tools_per_bench
        default: 5
      - key: research.default_mode
        description: "api | local-cpu"
        default: api
---

# Research Skill

## Invocation

User replies `experiment #<story_id> [mode=api|local-cpu]` in Telegram.

## Workflow

1. Create isolated workspace `${workspace_dir}/<story_id>-<timestamp>/`.
2. Extract tool/model names from story (LLM call with structured output schema).
3. Ask user: "다음 5개 중 뭐 비교할까?" (Telegram inline). 사용자 확정.
4. For each selected tool:
   - **API mode (T3-A)**: Use official provider SDK (Replicate/Together/DashScope/OpenAI/Anthropic Vision).
   - **Local-CPU mode (T3-B)**: `pip install` with **Tirith pre-exec scan**:
     ```
     tirith scan --package <name>==<version>
     # Checks: SHA-256 from pypi.org, cosign sig if available, homograph in name.
     # Abort if any check fails.
     ```
5. Run benchmark on fixed test set (5–10 sample inputs).
6. Compose comparison table → Telegram.
7. Archive raw outputs in workspace for later LLM Wiki ingestion.

## Safety Gates

- Before first install: confirm with user via Telegram.
- Max parallel installs: 1 (sequential, not concurrent — avoid resource spikes).
- Timeout per tool: 5min.
- Clean up workspace if `>1GB` after run.
```

### 7.3 Tirith 자동 설치 확인

Hermes가 Tirith를 백그라운드에서 auto-install (시작 블로킹 없음). 상태 확인:

```bash
hermes
❯ /tools
# tirith 체크 (enabled=true 확인)
```

### 7.4 검증

- [ ] "experiment #N" → Research Agent 스폰 <60초
- [ ] Tirith가 의도적으로 가짜 패키지(예: `pymupdg` 오타) 설치 시도 차단 ✅
- [ ] 정상 도구 (pymupdf 등) 설치 성공, 실행 완료
- [ ] 5개 도구 API 비교 결과 마크다운 테이블로 Telegram 수신
- [ ] 서브에이전트 완료 후 루트 세션으로 정상 복귀 (hang 없음)

### 7.5 제약 확인

- MAX_DEPTH=2: Monitoring(root) → Research(child) — OK. Research 안에서 Tool 호출은 서브에이전트 아니라 direct tool call이므로 depth 카운트 안 들어감.
- MAX_CONCURRENT=3: 한 번에 최대 3개 도구 병렬 설치 가능. 5개 비교 시 내부적으로 batch 2번.

---

## 8. Phase 5 — LLM Wiki + Pluggable Memory (2주)

### 8.1 LLM Wiki 초기화 (R46)

```bash
hermes
❯ activate skill llm-wiki
❯ Create a wiki at ~/.hermes/wiki for the domain "AI agent frameworks and tech news benchmarking".
```

에이전트가 `SCHEMA.md` 템플릿 기반으로 도메인별 태그 taxonomy 작성, 첫 `index.md` + `log.md` 생성. 사용자는 태그 taxonomy 수정·승인.

### 8.2 Research 결과 자동 Ingest

Research Agent 완료 후 Workflow 7단계 추가: "Wiki Ingest".

```markdown
# research/SKILL.md 끝에 추가
## Post-run Wiki Ingest

After benchmark complete:
1. Copy `story.html`, `bench_results.json`, `tools-<name>.md` to `wiki/raw/articles/`.
2. Invoke llm-wiki Ingest action on the new raw files.
3. llm-wiki creates/updates entity pages for each tool compared.
4. Log summary line to `wiki/log.md`.
```

결과: **같은 도구가 다음 기사에서 재등장하면 Research Agent가 Query action으로 이전 결과 확인 → 중복 벤치마크 회피**.

### 8.3 Pluggable Memory (Layer 4)

```bash
hermes memory setup
# → Holographic (local SQLite) 선택 추천 (데이터 주권)
# 또는 Mem0/Hindsight (엔터티 해결 강함, 클라우드)
hermes memory status
```

**초기엔 Holographic (로컬)로 시작**, 30일 후 session_db 규모 보고 클라우드 제공자 재평가.

### 8.4 검증

- [ ] Wiki Ingest 한 번 실행 후 `wiki/entities/` 생성 확인
- [ ] 같은 도구(예: "pymupdf") 두 번째 등장 시 Research Agent가 "이미 벤치마크함, 기존 요약:" 반환
- [ ] `hermes memory status`에서 활성 provider 확인, storage 증가 추이 정상

---

## 9. Phase 6 — Burst GPU · T3-C (선택, 1주)

### 9.1 진입 조건

T3-A/B로 커버 불가능한 로컬 ML OCR(olmOCR, Nougat, Marker, 로컬 GLM-4V-OCR)을 3회 이상 요청한 뒤에만 Phase 6 진입. 이전엔 과투자.

### 9.2 Modal 통합

```bash
pip install modal
modal token new   # OAuth
# hermes-agent/config.yaml에 Modal 백엔드 추가
```

`~/.hermes/skills/research/gpu_modal.py` (스킬 옆 Python):

```python
import modal

stub = modal.Stub("hermes-ocr-bench")
image = (
    modal.Image.debian_slim()
    .pip_install("olmocr==0.1.0", "torch==2.3.0", "transformers>=4.44")
)

@stub.function(gpu="L4", image=image, timeout=1800)
def run_olmocr_batch(pdfs: list[bytes]) -> list[dict]:
    from olmocr import OlmOCRPipeline
    pipe = OlmOCRPipeline.from_pretrained("allenai/olmOCR-7B-0225-preview")
    return [{"text": pipe(p), "pages": len(p) // 2048} for p in pdfs]
```

스킬 SKILL.md에 명시:

```markdown
## GPU Mode (T3-C)

When user says `experiment #N mode=gpu`, use `gpu_modal.run_olmocr_batch` instead of local CPU installs. Max monthly: 5 runs (hard cap at $15/month).
```

### 9.3 안전 장치 5가지 (plan §4.5 참조)

1. Modal workspace hard cap: **$20/월**
2. `timeout=1800` (30분/함수)
3. Tirith 스캔: Modal image 빌드 전 `olmocr` 패키지 SHA-256 검증
4. Telegram 승인: 첫 GPU 호출 시 사용자 확인
5. Dry-run: CPU 이미지로 입출력 shape 선검증

### 9.4 검증

- [ ] Modal `run_olmocr_batch.remote([tiny_pdf])` 성공 <5분 (cold start 포함)
- [ ] 비용 실측 $0.10 이내 (단일 작은 PDF)
- [ ] 완료 후 Modal 자동 teardown, 비용 0으로 복귀

---

## 10. Phase 7 — Self-Evolution (3–6개월 후)

### 10.1 진입 조건

session_db가 **3개월 이상 축적 + 200+ 세션**일 때. 그 이전엔 평가 데이터 부족.

### 10.2 설치 + 첫 실행

```bash
cd ~/src
git clone https://github.com/NousResearch/hermes-agent-self-evolution.git
cd hermes-agent-self-evolution
pip install -e ".[dev]"

export HERMES_AGENT_REPO=$HOME/.hermes   # 본 시스템 위치

# 첫 타겟: monitoring 스킬
python -m evolution.skills.evolve_skill \
    --skill monitoring \
    --iterations 10 \
    --eval-source sessiondb
```

### 10.3 5-Layer 가드레일 통과 확인

Self-Evolution이 생성한 PR 검토:
1. `pytest tests/ -q` 100% pass
2. 변형된 SKILL.md ≤ 15KB
3. prompt caching 파손 안 함 (mid-conversation 변경 없음)
4. 의미 보존 (원본 의도 유지)
5. 수동 승인 후 `~/.hermes/skills/monitoring/SKILL.md`에 반영

### 10.4 비용 추적

분기당 4개 스킬 × $2–10 = **$8–40/분기**. Telegram으로 월 1회 "Self-Evolution 지출: $X" 요약.

---

## 11. 검증 매트릭스

Phase별 런북 최종 점검표. 전부 ✅여야 다음 Phase 진입:

| 지표 | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Phase 5 | Phase 6 | Phase 7 |
|------|---------|---------|---------|---------|---------|---------|---------|
| `hermes doctor` 0 에러 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 기본 인터랙션 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Telegram 양방향 | — | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Cron 스케줄 정확도 | — | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 중복 발송 0건 | — | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 24h 무중단 | — | ≥ 3일 | ≥ 3일 | ≥ 5일 | ≥ 7일 | ≥ 7일 | ≥ 14일 |
| Writing HITL 루프 | — | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Publisher 승인 게이트 | — | — | ✅ | ✅ | ✅ | ✅ | ✅ |
| Tirith 가짜 패키지 차단 | — | — | — | ✅ | ✅ | ✅ | ✅ |
| Research subagent spawn <60s | — | — | — | ✅ | ✅ | ✅ | ✅ |
| LLM Wiki Ingest 성공 | — | — | — | — | ✅ | ✅ | ✅ |
| Wiki 중복 감지 | — | — | — | — | ✅ | ✅ | ✅ |
| Modal GPU 호출 | — | — | — | — | — | ✅ | ✅ |
| Modal 비용 ≤ $15/월 | — | — | — | — | — | ✅ | ✅ |
| Self-Evolution PR 1건 | — | — | — | — | — | — | ✅ |

---

## 12. 위험 & 대응

| 위험 | 확률 | 영향 | 대응 |
|------|------|------|------|
| OCI Free Tier capacity 부족 (Seoul/Tokyo) | 중 | 높음 | US 리전 대안, GCP $300 크레딧으로 90일 버팀 |
| Anthropic API 비용 급증 (버그 루프) | 중 | 중 | Console 월 $100 한도, Telegram 비용 알림 |
| Telegram 봇 spam 신고 → 차단 | 저 | 중 | 개인 DM만, privacy mode ON, 그룹 회피 |
| session_db 디스크 폭증 | 중 | 저 | Hermes auto-compaction, 90일 이상 세션 아카이브 |
| Tirith false positive → 정상 도구 차단 | 중 | 저 | 수동 bypass 승인 매커니즘 (Skills Trust R20 활용) |
| Self-Evolution이 스킬 퇴행 | 중 | 중 | 5-Layer 가드레일 + 인간 PR 리뷰 (기본 탑재), 롤백 branch |
| Hermes 업스트림 breaking change | 중 | 중 | `~/.hermes` snapshot 주 1회 백업, 버전 pin |
| MAX_CONCURRENT=3 한도 걸림 | 낮 | 낮 | Research 내 도구 병렬을 2개씩 batch, 구조 조정 없음 |
| Modal 계정 정지 (Phase 6) | 낮 | 저 | Baseten 또는 RunPod 서브옵션 준비 |
| LLM Wiki SCHEMA drift | 중 | 저 | Lint action 주 1회 cron 실행 |

---

## 13. 체크포인트 일정

**권장 케이던스 — 평일 2h/일 + 주말 (18h/주 기준)**:

| 기간 | Phase | 산출물 |
|------|-------|--------|
| D1–D2 | Phase 1 | OCI 부팅, Hermes 설치, `hermes doctor` clean |
| D3–D7 | Phase 2 | Monitoring 스킬 + Cron, 5일 연속 자동 발송 로그 |
| D8–D14 | Phase 3 | Writing/Publisher 스킬, 첫 블로그 글 업로드 |
| D15–D28 | Phase 4 | Research Agent + Tirith, T3-A 5개 도구 비교 완료 |
| D29–D42 | Phase 5 | LLM Wiki 초기화, Research 결과 자동 Ingest, 중복 감지 검증 |
| D43–D49 | Phase 6 (선택) | Modal 통합, 첫 olmOCR GPU 호출 |
| M3–M6 | Phase 7 | Self-Evolution 첫 PR 머지 |

**중도 회귀 트리거**: 어느 Phase든 검증 매트릭스 항목 2개 이상 실패 → Phase 재작업, 다음 단계 진입 보류. **3회 실패하면 Hermes 재검토** (Option B 하이브리드 회귀 또는 OpenClaw로 pivot).

---

## 부록 A — Hermes CLI 치트 시트

| 명령 | 용도 |
|------|------|
| `hermes` | 인터랙티브 세션 |
| `hermes doctor` | 헬스체크 |
| `hermes model` | LLM provider 변경 |
| `hermes tools` | 도구 enable/disable |
| `hermes setup` | 통합 설정 마법사 |
| `hermes gateway setup` | 메시징 플랫폼 (Telegram/Discord/Slack/...) |
| `hermes memory setup` | Pluggable Memory Layer 4 |
| `hermes skill create <name>` | 스킬 스캐폴딩 |
| `/cron add "schedule" "prompt" --skill X` | Cron 등록 (세션 내) |
| `/cron list` / `/cron edit` / `/cron remove` | Cron 관리 |

## 부록 B — 월 운영비 재산정 (Hermes standalone)

| 항목 | 월 비용 |
|------|--------|
| OCI Always Free | $0 |
| Claude API (T1/T2 + 주 1–2회 Research) | $30–50 |
| Telegram | $0 |
| Blog API | $0 (자체 호스팅) |
| Tirith | $0 (번들) |
| LLM Wiki | $0 (로컬) |
| Pluggable Memory (Holographic 로컬) | $0 |
| Modal Burst (Phase 6, 월 1–2회) | $0–15 |
| Self-Evolution (Phase 7, 분기 1회) | $0–13 (분기 $40 ÷ 3) |
| **월 총** | **$30–78** |

## 부록 C — 참고 링크

- **Hermes 공식 문서**: https://hermesagent.ai/docs (또는 `repos/hermes-agent/website/docs/`)
- **설치 스크립트**: https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh
- **Cron 가이드**: `repos/hermes-agent/website/docs/user-guide/features/cron.md`
- **Telegram 설정**: `repos/hermes-agent/website/docs/user-guide/messaging/telegram.md`
- **Skills 카탈로그**: `repos/hermes-agent/website/docs/reference/skills-catalog.md`
- **Self-Evolution PLAN**: `repos_research/hermes-agent-self-evolution/PLAN.md`
- **LLM Wiki SKILL**: `repos/hermes-agent/skills/research/llm-wiki/SKILL.md`
- **본 계획 선행 문서**:
  - `reports/deployment/hands_on_benchmark_plan.md` (범용 벤치마크 — Hermes 외 5개도 포함)
  - `reports/deployment/personal_agent_setup_plan.md` (원 OpenClaw 계획 — 역사 참조)
  - `reports/repos/details/hermes_agent_report.md` (Hermes 구조 분석 — R17–R22)
  - `reports/repos_research/details/hermes_self_evolution_report.md` (Self-Evolution R47)

---

**최종 수정**: 2026-04-15
