# PicoClaw 상세 분석 보고서

> **소스**: `reports/repos/framework_catalog.md` §6에서 추출
> **조사일**: 2026-02-25 (최종 업데이트)
> **GitHub**: [sipeed/picoclaw](https://github.com/sipeed/picoclaw) — 17,000~19,000 stars

---

## 목차

1. [기본 정보](#기본-정보)
2. [핵심 특징](#핵심-특징)
3. [리소스 사용량](#리소스-사용량)
4. [아키텍처](#아키텍처)
5. [구형 안드로이드 폰 지원](#구형-안드로이드-폰-지원)
6. [95% AI 생성 코드베이스](#95-ai-생성-코드베이스)
7. [개발사: Sipeed](#개발사-sipeed)
8. [주의사항](#주의사항)
9. [참고 링크](#참고-링크)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **GitHub** | [sipeed/picoclaw](https://github.com/sipeed/picoclaw) |
| **Stars** | 17,000~19,000 |
| **개발** | Sipeed (중국 RISC-V 하드웨어 제조사) |
| **라이선스** | MIT |
| **출시** | 2026.2.9 (1주 만에 12,000+ stars) |

---

## 핵심 특징

- Go 컴파일 → **단일 정적 바이너리**, 런타임 의존성 제로
- 로컬 LLM을 실행하지 않음 — **클라우드 API 호출 전용 메시지 브로커**
- 메시지 라우팅 + 도구 호출 + 스케줄링만 담당하는 **마이크로커널** 설계

---

## 리소스 사용량

| 지표 | PicoClaw | OpenClaw |
|------|----------|----------|
| RAM | <10MB (초기), 10-20MB (최근) | >1GB |
| 시작 시간 | <1초 | 수분 |
| 바이너리 | 단일 파일 | Node.js 프로세스 트리 |

---

## 아키텍처

```
User Interface Layer    → CLI + 멀티채널 (Telegram, Discord, WhatsApp, QQ, DingTalk)
Gateway Service Layer   → 채널 오케스트레이션, 스케줄링, 헬스 모니터링
Core Application        → AgentLoop + MessageBus pub/sub 시스템
LLM Abstraction Layer   → 13개+ 모델용 Provider 인터페이스
Tool Ecosystem          → 파일 작업, 웹 검색, 셸 실행, Cron 스케줄링
```

---

## 구형 안드로이드 폰 지원

```bash
# Termux에서 실행
pkg install proot wget
wget https://github.com/sipeed/picoclaw/releases/download/v0.1.1/picoclaw-linux-arm64
chmod +x picoclaw-linux-arm64
termux-chroot ./picoclaw-linux-arm64 onboard
```

- ARM64 정적 바이너리 → 추가 라이브러리 불필요
- 지원 아키텍처: `linux-arm64`, `linux-armv6`, `linux-mips64`, `linux-riscv64`, `linux-x86_64`

---

## 95% AI 생성 코드베이스

- NanoBot(Python) → Go 포팅을 **AI 에이전트가 자체 수행** (셀프 부트스트래핑)
- 인간이 아키텍처 감독 + 리뷰 + 스펙 작성 담당
- 95%는 **생성된 원시 코드 줄** 기준이며, 아키텍처 설계는 인간이 주도
- HN 커뮤니티 평가: "성능 향상은 Go 언어 특성에서 기인하며 신뢰할 만함"

---

## 개발사: Sipeed

- 중국의 RISC-V 개발 보드 전문 제조사 (Maix, LicheeRV, Tang FPGA)
- PicoClaw는 자사 초저가 하드웨어($10 LicheeRV-Nano 등)에 소프트웨어 생태계를 구축하려는 전략의 일환

---

## 주의사항

- v1.0 전까지 **프로덕션 배포 비권장** (네트워크 보안 미해결)
- 최근 PR 급증으로 메모리 10-20MB로 증가 추세
- 오프라인/로컬 LLM 미지원 (클라우드 API 키 필수)

---

## 참고 링크

- [GitHub — sipeed/picoclaw](https://github.com/sipeed/picoclaw)
- [CNX Software — Technical Deep Dive](https://www.cnx-software.com/2026/02/10/picoclaw-ultra-lightweight-personal-ai-assistant-run-on-just-10mb-of-ram/)
- [Hackster.io — OpenClaw Alternative for $10](https://www.hackster.io/news/forget-the-mac-mini-run-this-openclaw-alternative-for-just-10-da23b2819d25)
- [Hacker News Discussion](https://news.ycombinator.com/item?id=47004845)
