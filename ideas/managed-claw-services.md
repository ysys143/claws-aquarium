
# idea5: 24/7 Claw형 AI 에이전트 - 서비스/제품 사례

**기준**: 24시간 상주 구동 + 메신저 인터페이스 + 자율 작업 수행 + 다양한 커넥터

---

## 매니지드 SaaS (직접 호스팅해주는 서비스)

### Claw 계열 직접 호스팅

kimi claw (Moonshot AI)
https://www.kimi.com/bot

z.ai openclaw
https://docs.z.ai/devpack/tool/openclaw

alibaba openclaw
https://www.alibabacloud.com/en/campaign/ai-openclaw?_p_lc=1

baidu duclaw - 제로-배포, RMB 17.8/월, 2026-03-11 출시
https://www.prnewswire.com/news-releases/baidu-launches-duclaw-enables-zero-deployment-access-to-openclaw-302710924.html

zhipu autoclaw - 출시일 주가 16%↑
https://www.chinadailyasia.com/article/630187

openclawd - 상업 관리형 플랫폼
https://finance.yahoo.com/news/openclawd-releases-major-platform-openclaw-150000544.html

국가超算互联网 (국가 슈퍼컴퓨팅 인터넷) - OpenClaw 사용자 대상 1000만 토큰 무료 제공, 이후 0.1元/백만 토큰
https://finance.eastmoney.com/a/202603113669117567.html

doneclaw - "Telegram 위의 개인 AI 에이전트"로 포지셔닝한 매니지드 서비스
https://doneclaw.com/blog/personal-ai-agent-telegram/

### Claw 계열 미출시/베타

tencent qclaw - 원클릭 배포, 내부 테스트 중
https://technode.com/2026/03/09/tencent-reportedly-tests-qclaw-ai-agent-with-one-click-openclaw-deployment/

tencent workbuddy - WeChat 통합, 출시일 주가 7.3%↑
https://www.bloomberg.com/news/articles/2026-03-10/tencent-zhipu-shares-jump-on-launches-of-ai-agents-tapping-into-openclaw

tencent wechat ai agent - Mini Program 생태계 연동 (미출시, Q3 2026 목표)
https://pandaily.com/we-chat-s-in-house-ai-model-reportedly-in-development-launch-planned-within-the-year/

xiaomi miclaw - 모바일 특화, 클로즈드 베타

nvidia nemoclaw

### Claw형 특성을 가진 독자 서비스

perplexity personal computer - 전용 Mac mini + Perplexity 서버 결합. "지속적 디지털 프록시", 파일·앱·세션에 상시 연결. 대기 목록 모집 중
https://news.hada.io/topic?id=27438

manus ai - 멀티스텝 자율 에이전트, Telegram/WhatsApp/LINE/Slack/Discord 통합 (2026-02 출시)
https://siliconangle.com/2026/02/16/manus-launches-personal-ai-agents-telegram-messaging-apps-come/

lindy - 24/7 inbox·캘린더·미팅 자율 관리, "runs while you sleep"
https://www.lindy.ai/

ai magicx - WhatsApp·Telegram·웹 채널, 인프라 관리 불필요
https://www.aimagicx.com/blog/openclaw-alternatives-comparison-2026

---

## 빅테크 앰비언트 에이전트

google gemini agent - 웹브라우징·캘린더·Gmail·Drive 멀티스텝 자율 실행 (AI Ultra 구독)
https://gemini.google/overview/agent/

microsoft copilot actions - Windows 11 네이티브, M365 연동 예약/트리거 자율 작업
https://beebom.com/real-world-ai-agents-examples/

---

---

## 클라우드 배포 레포 (아키텍처·보안 특성이 다른 구현체)

### AWS

serverless-openclaw - Lambda 기반 서버리스 (콜드스타트 있음, 상시 구동 아님)
https://github.com/serithemage/serverless-openclaw

openclaw-lab-on-cloud - EC2 + Terraform, 비용 최적화 스케줄링
https://github.com/carlosacchi/openclaw-lab-on-cloud

aws lightsail
https://aws.amazon.com/ko/blogs/korea/introducing-openclaw-on-amazon-lightsail-to-run-your-autonomous-private-ai-agents/

### GCP

openclaw-gcp-setup
https://github.com/kubony/openclaw-gcp-setup

### Azure

openclaw-azure - HTTPS·IP제한·백업, ~$13/월
https://github.com/aerolalit/openclaw-azure

openclaw-on-azure - VMSS + Bicep (인스턴스당 전용 공인 IP)
https://github.com/deankroker/openclaw-on-azure

openclaw-azure-appservice
https://github.com/seligj95/openclaw-azure-appservice

### Kubernetes / Helm

openclaw-helm (serhanekicii)
https://github.com/serhanekicii/openclaw-helm

openclaw-helm (Chrisbattarbee)
https://github.com/Chrisbattarbee/openclaw-helm

### Serverless / Edge

cloud-claw - Cloudflare Workers + Containers (Worker가 라우팅·인증 처리)
https://github.com/miantiao-me/cloud-claw

moltworker (Cloudflare 공식) - Cloudflare R2·Browser Rendering·AI Gateway·Zero Trust 활용
https://github.com/cloudflare/moltworker

### 기타

openclaw-terraform-hetzner - Hetzner Cloud, 방화벽·cloud-init 자동화
https://github.com/andreesg/openclaw-terraform-hetzner

clawhost - 오픈소스 셀프호스팅 플랫폼, 1분 배포
https://github.com/bfzli/clawhost

coolify 기반 openclaw (281 stars)
https://github.com/coollabsio/openclaw

oracle cloud always free - 무료 24/7 호스팅 (커뮤니티 검증)
https://agenteer.com/learn/tutorials/openclaw-oracle-cloud/

### 클라우드 제공업체 공식 1-Click

tencent cloud lighthouse
https://www.tencentcloud.com/act/pro/intl-openclaw

hostinger vps
https://www.hostinger.com/vps/docker/openclaw

digitalocean 1-click app
https://www.digitalocean.com/community/tutorials/how-to-run-openclaw

cloudtype (한국)
https://www.youtube.com/watch?v=LS1ub3_BEow

railway (공식)
https://docs.openclaw.ai/install/railway

render (공식)
https://docs.openclaw.ai/install/render

northflank (공식)
https://docs.openclaw.ai/install/northflank

---

## 참고: 시장 분석

중국 빅테크들의 OpenClaw 무료 설치 경쟁 ("China's OpenClaw Frenzy" - Bloomberg)
https://www.caixinglobal.com/2026-03-09/chinese-tech-giants-offer-free-openclaw-installations-to-boost-cloud-services-102421293.html

OpenClaw 4개월 만에 GitHub 30만 스타, 중국 규제기관 데이터 유출 경고 → 20여 증권사 사용 제한 → "설치→제거 서비스"로 시장 급변
https://www.36kr.com/p/3721246695487881

2026 개인 AI 에이전트 시장 overview
https://till-freitag.com/blog/personal-ai-assistant-market-overview-2026

Claw 파생 생태계 전체 비교표 (RAM·stars 포함)
https://rywalker.com/openclaw-alternatives-2026

Claw 파편화 분석 - "Claw Wars 2026"
https://medium.com/@schemata/claw-wars-2026-why-personal-ai-agents-are-fragmenting-into-lightweight-specialized-clones-and-b0fbeceafce6
