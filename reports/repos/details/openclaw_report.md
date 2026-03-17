# OpenClaw 상세 분석 보고서

> **소스**: `reports/repos/framework_catalog.md` §1에서 추출
> **조사일**: 2026-02-25 (최종 업데이트)
> **GitHub**: [openclaw/openclaw](https://github.com/openclaw/openclaw) — 223,000+ stars

---

## 목차

1. [기본 정보](#기본-정보)
2. [이름 변천사](#이름-변천사)
3. [핵심 특징](#핵심-특징)
4. [보안 위기](#보안-위기)
5. [참고 링크](#참고-링크)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub** | [openclaw/openclaw](https://github.com/openclaw/openclaw) |
| **Stars** | 223,000+ |
| **개발자** | Peter Steinberger (PSPDFKit 창시자, 이후 OpenAI 합류) |
| **라이선스** | MIT |

---

## 이름 변천사

- **ClawdBot** (2025.11) → **Moltbot** (2026.1.27) → **OpenClaw** (2026.1.30)
- "Clawd"가 Anthropic의 "Claude" 상표와 유사해 법적 통보를 받고 개명
- 개명 과정에서 스캐머가 버려진 `@clawdbot` 트위터 계정을 탈취해 가짜 `$CLAWD` 토큰 발행 (시총 $16M까지 급등 후 붕괴)

---

## 핵심 특징

- **셀프 호스팅** 개인 AI 에이전트 플랫폼 (코딩 도구가 아닌 범용 AI 비서)
- WhatsApp, Telegram, Slack, Discord, Signal, iMessage 등 **12개+ 메시징 채널** 지원
- 음성 모드 (ElevenLabs 통합), Canvas 비주얼 워크스페이스, 브라우저 제어
- **ClawHub** 스킬 마켓플레이스, Cron 자동화, 웹훅
- Lobster 워크플로우 셸: 스킬/도구를 조합 가능한 파이프라인으로 전환

---

## 보안 위기 (폭발적 성장의 그늘)

OpenClaw의 급격한 성장과 함께 심각한 보안 사고가 동시다발적으로 발생했습니다.

### CVE-2026-25253 (CVSS 8.8) — 1-Click 원격 코드 실행

- Control UI의 `gatewayUrl` 쿼리 파라미터 검증 부재 악용
- 공격자 지정 주소로 WebSocket 연결을 자동 수립, 인증 토큰 전송
- WebSocket 오리진 헤더 미검증 → 크로스사이트 WebSocket 하이재킹
- 공격 체인: 악성 웹페이지 → 인증 토큰 탈취 → 보안 확인 비활성화 → 컨테이너 이스케이프 → 호스트에서 임의 명령 실행
- v2026.1.29에서 패치됨

### ClawHavoc 공급망 공격

- ClawHub에 **1,184개 악성 스킬 패키지** 유포
- 암호화폐 거래 자동화 도구로 위장 (ByBit, Polymarket, Axiom 등 실제 브랜드 사용)
- 91%가 프롬프트 인젝션을 동시 사용 → AI 안전 메커니즘과 전통적 보안 도구 모두 우회
- 주요 페이로드: Atomic macOS Stealer (AMOS)
- Koi Security 감사: 2,857개 스킬 중 341개(12%)가 악성, 335개가 단일 조직 캠페인

### 노출 규모

- Censys 추적: 2026.1.25~31 사이 ~1,000개에서 21,000개+ 인스턴스로 급증
- 42,665개 노출 인스턴스 중 93.4%가 인증 우회 상태
- 전체 보안 감사에서 512개 취약점 발견, 8개가 Critical 등급

> 이러한 보안 문제가 수많은 대안들의 등장을 직접적으로 촉발했습니다.

---

## 참고 링크

- [From Clawdbot to Moltbot to OpenClaw — CNBC](https://www.cnbc.com/2026/02/02/openclaw-open-source-ai-agent-rise-controversy-clawdbot-moltbot-moltbook.html)
- [OpenClaw Complete Guide 2026 — NxCode](https://www.nxcode.io/resources/news/openclaw-complete-guide-2026)
- [OpenClaw Bug Enables One-Click RCE — The Hacker News](https://thehackernews.com/2026/02/openclaw-bug-enables-one-click-remote.html)
- [ClawHavoc Poisons ClawHub — CyberPress](https://cyberpress.org/clawhavoc-poisons-openclaws-clawhub-with-1184-malicious-skills/)
- [Personal AI Agents Are a Security Nightmare — Cisco Blogs](https://blogs.cisco.com/ai/personal-ai-agents-like-openclaw-are-a-security-nightmare)
