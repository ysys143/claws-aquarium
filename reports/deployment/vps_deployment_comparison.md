# VPS 배포 플랫폼 비교: AWS EC2/Lightsail, Azure, GCP, Hetzner, Oracle Free Tier

**작성일**: 2026년 3월 14일
**대상**: 메신저 호스팅(Telegram/WhatsApp), 24시간 상시 운영 기준

---

## 요약

5개 주요 VPS 플랫폼의 배포 복잡도, 월 비용, 보안, 확장성, 메신저 호스팅 적합성을 비교합니다.
- **최저 비용**: Oracle Free Tier (무료) / Hetzner ($5/월)
- **최고 용이성**: AWS Lightsail (원클릭)
- **최고 보안**: AWS EC2 + IronClaw (WASM 샌드박스)
- **최고 확장성**: GCP (Auto Scaling)
- **메신저 최적**: Hetzner + NanoClaw (컨테이너 격리, $5/월)

---

## 1. 비교 매트릭스

### 1.1 배포 복잡도 (터미널 명령어 수 기준)

| 플랫폼 | 최소 명령어 수 | 설정 시간 | 진입장벽 | UI 지원 |
|--------|---------------|----------|---------|--------|
| **AWS Lightsail** | 2-3 | 5분 | 최하 | 웹 콘솔 (완벽) |
| **AWS EC2** | 8-12 | 15-20분 | 중간 | 웹 콘솔 + CLI |
| **Azure VPS** | 6-10 | 10-15분 | 중간 | 웹 포탈 + CLI |
| **GCP Compute Engine** | 7-11 | 12-18분 | 중간 | 웹 콘솔 + gcloud |
| **Hetzner (Terraform)** | 3-5 | 8-10분 | 중간 | 웹 UI + Terraform |
| **Oracle Free Tier** | 4-6 | 10-12분 | 중간 | 웹 콘솔 |

#### 상세 배포 단계

**AWS Lightsail** (가장 간단)
```bash
# 웹 콘솔에서 1. Instance 생성 2. SSH 키 다운로드 3. SSH 접속
ssh -i lightsail_key.pem ubuntu@IP
# 단계: 3개
```

**AWS EC2** (표준)
```bash
# 1. Security Group 생성 (규칙 설정)
aws ec2 create-security-group --group-name my-sg --description "My SG"
# 2. Key Pair 생성
aws ec2 create-key-pair --key-name my-key
# 3. Instance 시작
aws ec2 run-instances --image-id ami-xxxxx --instance-type t3.micro \
  --security-group-ids sg-xxxxx --key-name my-key
# 4. Elastic IP 할당 (선택)
# 5. Terraform/CloudFormation (권장)
# 단계: 8-12개
```

**Azure VMS**
```bash
# 1. Resource Group 생성
az group create --name myResourceGroup --location eastus
# 2. Network 생성
az network vnet create --resource-group myResourceGroup --name myVnet
# 3. Subnet 생성
az network vnet subnet create --resource-group myResourceGroup --vnet-name myVnet
# 4. NSG 생성 및 규칙 추가
# 5. VM 생성
az vm create --resource-group myResourceGroup --name myVM --image UbuntuLTS
# 단계: 6-10개
```

**GCP Compute Engine**
```bash
# 1. 프로젝트 설정
gcloud config set project PROJECT_ID
# 2. Firewall 규칙 생성
gcloud compute firewall-rules create allow-ssh --allow tcp:22
# 3. Instance 생성
gcloud compute instances create my-instance --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud --machine-type=e2-micro
# 4. External IP 할당
gcloud compute addresses create my-ip --region=us-central1
# 단계: 7-11개
```

**Hetzner (Terraform 사용)**
```bash
# 1. Terraform 프로젝트 초기화
terraform init
# 2. main.tf 작성 (Cloud-Init 포함)
# 3. terraform plan
terraform plan
# 4. terraform apply
terraform apply
# 5. 확인
# 단계: 3-5개 (Terraform 사용 시)
```

**Oracle Free Tier**
```bash
# 1. VCN(Virtual Cloud Network) 생성
# 2. Subnet 생성
# 3. Internet Gateway 생성
# 4. Security List 규칙 추가
# 5. Compute Instance 생성 (Always Free image 사용)
# 6. SSH 키 설정
# 단계: 4-6개
```

---

### 1.2 월 비용 (24시간 상시 운영, 최소 사양)

#### 최소 사양 정의
- **CPU**: 1-2 vCPU
- **RAM**: 512MB - 1GB
- **디스크**: 10-20GB
- **대역폭**: 1-5TB/월 (메신저봇 기준)
- **영역**: us-east-1 (또는 가장 저렴한 지역)

| 플랫폼 | 인스턴스 | vCPU | RAM | 월 비용 | 비고 |
|--------|---------|------|-----|--------|------|
| **AWS Lightsail** | 512MB $3.50 | 0.5 | 512MB | $3.50 | 가장 저렴 (고정 요금) |
| **AWS EC2** | t3.micro (free) | 1 | 1GB | $0 (12개월) | 프리 티어 이후: ~$5-8 |
| **Azure** | B1s (free) | 1 | 1GB | $0 (12개월) | 프리 티어 이후: ~$7-10 |
| **GCP** | e2-micro (free) | 0.25-2 | 1GB | $0 (항상) | **영구 프리 티어** |
| **Hetzner** | CX11 | 1 | 1GB | €2.99 (~$3.25/월) | 유럽 가장 저렴 |
| **Oracle Free Tier** | 1 OCPU/1GB RAM | 1 | 1GB | $0 (영구) | **영구 무료** |

#### 상세 비용 분석

**AWS Lightsail (고정 요금 모델)**
- 512MB: $3.50/월 (고정)
- 1GB: $5/월
- 2GB: $10/월
- 기타: 스토리지 25GB, 데이터 전송 1TB/월 포함
- 장점: 예측 가능한 비용, 제일 간단
- 단점: 프리 티어 없음, 장기 비용 높음

**AWS EC2 (pay-as-you-go)**
- t3.micro: 프리 티어 12개월 $0
- 프리 티어 이후: ~$5-8/월
- t4g.micro (Graviton): ~$3/월
- 네트워크: 데이터 전송 1GB/월 무료, 초과 시 $0.09/GB
- RDS/EBS 추가 비용

**Azure (프리 티어)**
- B1s: 12개월 프리 티어 (750시간)
- 프리 티어 이후: ~$7.50/월 (종량제), ~$54/월 (예약)
- 데이터 전송: 첫 100GB/월 무료, 초과 시 $0.087/GB
- 저장소: 추가 비용

**GCP (영구 프리 티어)**
- e2-micro: 월 745시간 무료 (24/7 운영 시 한 달 한 대만)
- 추가 인스턴스: ~$6-8/월
- 스토리지: 30GB 무료
- 네트워크: 매월 1GB 무료, 초과 시 $0.12/GB
- **가장 긴 무료 기간**

**Hetzner (정액제)**
- CX11 (1 vCPU, 1GB): €2.99/월 (~$3.25)
- CX21 (2 vCPU, 4GB): €5.99/월 (~$6.50)
- 스토리지: 25GB SSD 포함
- 대역폭: 무제한 (소수 GB 제외)
- 백업: 자동, 비용별도
- **유럽/아시아에서 가장 저렴**

**Oracle Free Tier (영구 무료)**
- 1 OCPU + 1GB RAM: 영구 무료
- 20GB 블록 저장소: 무료
- 300GB 데이터 전송/월: 무료
- 기타: 로드밸런서, 데이터베이스 무료
- **진정한 무료**, 단 아시아 리전 제한

#### 연간 비용 비교 (12개월)

| 플랫폼 | 프리 티어 기간 | 연간 비용 | 비고 |
|--------|---------------|----------|------|
| **AWS Lightsail** | 없음 | $42 | 가장 저렴한 유료 옵션 |
| **AWS EC2** | 12개월 | $0 (이후 $60-96) | 12개월 후 비용 증가 |
| **Azure** | 12개월 | $0 (이후 ~$90-120) | 12개월 한정 |
| **GCP** | 무제한 | $0 (영구) | **영구 프리 티어** |
| **Hetzner** | 없음 | €35.88 (~$39) | 저렴하고 일관적 |
| **Oracle Free Tier** | 무제한 | $0 (영구) | **영구 무료** |

---

### 1.3 보안 (방화벽, 네트워크 격리, 백업)

| 차원 | AWS EC2 | Azure | GCP | Hetzner | Oracle |
|------|---------|-------|-----|---------|--------|
| **방화벽 (Inbound)** | Security Group | NSG | 방화벽 규칙 | 방화벽 (UI) | Security List |
| **방화벽 (Outbound)** | 별도 설정 | 별도 설정 | 기본 Allow | 기본 Allow | 별도 설정 |
| **네트워크 격리** | VPC 기반 | VNet 기반 | VPC 기반 | 프라이빗 네트워크 | VCN 기반 |
| **DDoS 보호** | AWS Shield (기본) | Azure DDoS (기본) | GCP Cloud Armor | 제한적 | 제한적 |
| **자동 백업** | 수동 (EBS Snapshot) | 자동 (저비용) | 스냅샷 | 스냅샷 ($2-3/월) | 수동 (Block Volume) |
| **암호화 (전송)** | TLS 지원 | TLS 지원 | TLS 지원 | TLS 지원 | TLS 지원 |
| **암호화 (저장소)** | EBS 암호화 | 디스크 암호화 | Persistent Disk 암호화 | 디스크 암호화 | 블록 볼륨 암호화 |
| **IAM/접근 제어** | S (세분화) | S (세분화) | S (세분화) | B (기본) | A (세분화) |
| **SSH 키 관리** | 수동 | 수동 | Metadata Service | 수동 | 수동 |
| **보안 감사** | CloudTrail | Azure Monitor | Cloud Logging | 기본 로깅 | Audit Logs |

#### 보안 성숙도 분류

**Tier 1 (엔터프라이즈급)**
- AWS EC2 (+ IronClaw WASM): Security Group + VPC + 다층 암호화
- Azure (+ Defender): NSG + VNet + Azure Defender
- GCP (+ Cloud Armor): VPC Firewall + Cloud Armor + Confidential VMs

**Tier 2 (중규모)**
- Hetzner: 기본 방화벽, 프라이빗 네트워크, 자동 스냅샷
- Oracle: VCN 격리, Security List, 암호화 지원

#### 권장 보안 설정 (메신저봇)

**최소 구성 (Development)**
```yaml
Inbound Rules:
  - 22/tcp (SSH) from 0.0.0.0/0  # 차단 권장: 특정 IP만
  - 80/tcp (HTTP) from 0.0.0.0/0
  - 443/tcp (HTTPS) from 0.0.0.0/0
Outbound Rules:
  - All (메신저 API 연결용)

Network:
  - VPC/VNet/VCN 기본값 사용
  - Private Subnet 미사용
```

**권장 구성 (Production)**
```yaml
# AWS Security Group 예
Inbound:
  - 22/tcp from {YOUR_IP}/32 (Bastion Host)
  - 443/tcp from 0.0.0.0/0 (HTTPS only)
Outbound:
  - 443/tcp to 0.0.0.0/0 (HTTPS, API calls)
  - 53/udp to 0.0.0.0/0 (DNS)

Network:
  - Private Subnet에 NAT Gateway 경유 배포
  - 자동 스냅샷 활성화 (일 1회 이상)
  - 암호화 (EBS, 전송) 필수

Monitoring:
  - CloudTrail 로깅 활성화
  - 알람: Unauthorized API calls, Failed logins
```

---

### 1.4 확장성 (트래픽 증가에 따른 대응)

| 플랫폼 | Auto Scaling | Load Balancer | 수평 확장 시간 | 수직 확장 | 다운타임 |
|--------|-------------|---------------|--------------|---------|---------|
| **AWS Lightsail** | 수동 | 별도 비용 ($18/월) | 2-3분 | 수동 (재시작) | 예 |
| **AWS EC2** | Auto Scaling 그룹 | ELB 무료 (기본) | 1-2분 | 수동 (재시작) | 예 |
| **Azure** | Virtual Machine Scale Sets | Load Balancer 무료 | 1-2분 | 수동 (재시작) | 예 |
| **GCP** | Instance Group + MIG | Cloud Load Balancing 무료 | 1-2분 | 수동 (재시작) | 예 |
| **Hetzner** | 수동 | 별도 ($5-10/월) | 2-3분 | 수동 (재시작) | 예 |
| **Oracle** | 수동 | Load Balancer ($0.025/시간) | 2-3분 | 수동 (재시작) | 예 |

#### 확장성 트리거 (메신저봇 기준)

**수평 확장 필요 시나리오**
- 일일 활성 사용자 > 10,000명
- 동시 연결 > 1,000개
- API 응답 시간 > 2초
- CPU 사용률 > 70%

**각 플랫폼별 확장 전략**

**AWS EC2 (가장 자동화)**
```
Auto Scaling Group 설정:
  - Min: 1, Max: 5, Target CPU: 50%
  - Health Check: ELB (60초)
  - Scaling Policy: Step (1분 에러 → +1, 5분 정상 → -1)
  - 예상 확장 시간: 2-3분
```

**GCP (가장 빠름)**
```
Instance Group (managed):
  - Min: 1, Max: 5
  - Scaling: CPU 60%, Memory 80%
  - Health Check: HTTP 200
  - Autoscaling delay: 1분
```

**Hetzner (제한적)**
- 수동 스케일링만 지원
- 신규 인스턴스 프로비저닝: 2-3분
- 로드밸런싱: 별도 ($5-10/월)

**Oracle (제한적)**
- Instance Pools로 수동 관리
- 확장 시간: 2-3분
- 로드밸런싱 비용: $0.025/시간 (약 $18/월)

#### 메신저 트래픽 확장 예상

```
Telegram 봇 예시 (10,000 사용자):
  - 평상시: 50 RPS (Request Per Second)
  - 피크: 500 RPS (이벤트 공지)
  - 필요 인스턴스: t3.small 1대 (50 RPS) → t3.medium 2대 (500 RPS)

예상 비용 증가:
  - AWS Lightsail: $3.50 → $10/월 (2대)
  - AWS EC2: $5 → $16/월 (t3.small x2)
  - GCP: $0 (프리 티어 범위 내)
  - Hetzner: €2.99 → €8.98/월 (CX21로 확장)
```

---

### 1.5 메신저 호스팅 적합성 (Telegram/WhatsApp)

#### 요구사항 분석

**Telegram Bot API (Bot Framework)**
- 프로토콜: HTTPS Long Polling 또는 Webhook
- 최소 대역폭: 1Mbps
- 메모리: 50-100MB (Python library)
- CPU: < 10% (대부분 대기)
- 저장소: 100MB (로그/데이터베이스)
- 라이브러리: pyTelegramBotAPI, aiogram (Python), Telegraf (Node.js)

**WhatsApp Cloud API**
- 프로토콜: HTTPS Webhook (필수)
- 최소 대역폭: 2Mbps (이미지/영상 송수신)
- 메모리: 200-300MB (미디어 처리)
- CPU: 15-20% (미디어 인코딩)
- 저장소: 500MB (미디어 캐시)
- 라이브러리: whatsapp-web.js (Node.js), python-whatsapp (Python)

#### 플랫폼별 적합성 평가

| 항목 | Lightsail | EC2 | Azure | GCP | Hetzner | Oracle |
|------|-----------|-----|-------|-----|---------|--------|
| **Telegram Long Polling** | A (우수) | A+ | A | A+ | A | A |
| **Telegram Webhook** | A | A+ | A | A+ | A | A |
| **WhatsApp Cloud API** | B+ | A+ | A+ | A+ | A | A |
| **미디어 처리 성능** | B | A | A | A | A | B |
| **HTTPS 인증서** | A (자동) | A (자동) | A (자동) | A (자동) | B (수동) | A (자동) |
| **메신저 API 신뢰도** | A | A | A | A | A | B (아시아 리전) |
| **채팅 지연** | < 200ms | < 100ms | < 100ms | < 100ms | < 150ms | < 200ms (아시아) |
| **비용 효율** | B | A (12개월) | A (12개월) | S (무제한) | S | S |

#### 상세 평가

**Telegram 메신저봇 (최고 인기)**

가장 적합한 플랫폼: **GCP** (무제한 프리 티어) > **Hetzner** (€2.99) > **AWS Lightsail** ($3.50)

```
예시: Telegram 뉴스봇 (10,000 사용자)

1. Long Polling 방식:
   - 메모리: 50MB
   - CPU: 2% (대부분 sleep)
   - 대역폭: 0.5Mbps
   - 적합성: 모든 플랫폼 (512MB 충분)

2. Webhook 방식 (권장):
   - 메모리: 100MB
   - CPU: 5%
   - HTTPS 인증서: Let's Encrypt (무료)
   - CDN: Cloudflare (무료)
   - 적합성: 모든 플랫폼

권장 설정:
  - GCP e2-micro (무료) + Cloud Run (선택)
  - Hetzner CX11 (€2.99) + Nginx 리버스 프록시
  - AWS Lightsail 512MB ($3.50) + CloudFlare
```

**WhatsApp Cloud API (기업용)**

가장 적합한 플랫폼: **AWS EC2** (프리 티어) > **Azure** (프리 티어) > **GCP** (무제한 프리 티어)

```
예시: WhatsApp 고객 지원봇 (1,000 대화/일)

1. 요구사항:
   - 메모리: 300MB (미디어 캐시)
   - CPU: 20% (이미지 처리)
   - 대역폭: 2-5Mbps
   - 저장소: 1GB (로그)

2. 적합 플랫폼:
   - AWS EC2 t3.small (1 vCPU, 2GB) - 프리 12개월
   - Azure B1 (1 vCPU, 2GB) - 프리 12개월
   - GCP e2-medium (프리 티어 불가, €8-10 필요)
   - Hetzner CX21 (2 vCPU, 4GB, €5.99)

예상 월 비용:
  - AWS/Azure: $0 (12개월), 이후 ~$10-15
  - GCP: $8-10 (프리 범위 초과)
  - Hetzner: €5.99 (일정)
```

---

### 1.6 운영 난이도 (모니터링, 트러블슈팅)

| 항목 | Lightsail | EC2 | Azure | GCP | Hetzner | Oracle |
|------|-----------|-----|-------|-----|---------|--------|
| **모니터링** | 기본 (2분 단위) | CloudWatch (1분) | Monitor (1분) | Cloud Monitoring (10초) | Grafana (수동) | OCI Monitor (1분) |
| **로깅** | 제한적 | CloudTrail | Azure Log | Cloud Logging | Syslog | Audit Logs |
| **알람 설정** | SNS 필요 | CloudWatch Alarms | Alert Rule | Monitoring Policies | 수동 | Notification |
| **자동 복구** | 없음 | Auto Scaling | Scale Sets | MIG | 없음 | 없음 |
| **대시보드** | 기본 | CloudWatch Dashboard | Azure Portal | Cloud Console | 웹 UI | OCI Console |
| **로그 저장** | 2주 | CloudWatch Logs | Log Analytics | Cloud Logging Storage | 30일 | 180일 |
| **비용 (월)** | 무료 | 무료 (기본) | 무료 (기본) | 무료 | $10+ (Grafana) | 무료 |
| **학습곡선** | 낮음 | 높음 | 중간 | 높음 | 낮음 | 중간 |

#### 모니터링 설정 (메신저봇)

**필수 메트릭**
1. CPU 사용률 > 80% (알람)
2. 메모리 사용률 > 90% (알람)
3. Disk I/O > 100% (알람)
4. 네트워크 에러 (알람)
5. API 응답 시간 > 2초 (알람)
6. 메신저 API 연결 끊김 (알람)

**각 플랫폼 설정 난이도**

```yaml
# AWS CloudWatch (복잡, 강력)
Metric:
  - CPUUtilization, MemoryUtilization
Alarm:
  - Threshold: 80% (5분, 2회 연속)
  - Action: SNS → Email/Slack
비용: 무료 (기본), 고급 메트릭 $0.10/개

# GCP Cloud Monitoring (간단, 강력)
Metric:
  - compute.googleapis.com/instance/cpu/utilization
Alarm:
  - Threshold: 80% (1분)
  - Notification: Slack/Email/SMS
비용: 무료 (기본)

# Azure Monitor (중간)
Metric:
  - Percentage CPU, Available Memory %
Alert:
  - Threshold: 80% (1분)
  - Action: Action Group (Slack, Email)
비용: 무료 (기본)

# Hetzner (수동, 어려움)
Option 1: Self-hosted Prometheus + Grafana (복잡)
Option 2: 서드파티 (New Relic, DataDog) $20+/월
비용: $10+ (자동화 시)
```

#### 트러블슈팅 용이성

**시나리오 1: 메신저봇이 응답 안 함**

```
AWS Lightsail (쉬움):
1. 웹 콘솔 → Instance → Reboot (1분)
2. SSH 접속 → systemctl status telegram-bot
3. 로그: /var/log/telegram-bot.log

AWS EC2 (중간):
1. EC2 Console → Status Checks
2. CloudWatch Logs 확인
3. Systems Manager Session Manager로 SSH 대체

GCP (쉬움):
1. Cloud Console → Compute Instances
2. Cloud Logging 에서 에러 검색
3. SSH 버튼으로 직접 접속

Hetzner (어려움):
1. 웹 UI → Server → Console
2. SSH 접속 필수
3. journalctl -u telegram-bot -f
```

**시나리오 2: 디스크 부족**

```
AWS Lightsail:
  - 스냅샷 생성 → 큰 인스턴스로 복원
  - 다운타임: 5-10분
  - 비용: 스냅샷 $0.05/GB

AWS EC2:
  - EBS 볼륨 확장 (온라인, 파일시스템 확장 필요)
  - 다운타임: 1분 (파일시스템 확장 시)
  - 비용: $0.10/GB/월 추가

GCP:
  - Persistent Disk 확장 (온라인)
  - 다운타임: 1분
  - 비용: $0.04/GB/월 추가

Hetzner:
  - 볼륨 추가 필요
  - 다운타임: 2-5분 (마운트)
  - 비용: €0.01/GB/월
```

---

## 2. 강점/약점 분석

### 2.1 AWS Lightsail

**강점**
- 가장 간단한 배포 (원클릭)
- 예측 가능한 월 비용 ($3.50-10)
- 통합된 관리 콘솔
- 자동 방화벽 관리
- CloudFlare CDN 통합 (가능)

**약점**
- 프리 티어 없음 (프로젝트 초기 비용)
- Auto Scaling 미지원 (별도 인스턴스 수동 추가)
- 용량 초과 시 인스턴스 업그레이드만 가능 (마이그레이션 필요)
- 수평 확장 비용 높음 (2대 = $7/월)

**추천 대상**
- 초보자
- 스타트업 (프로토타입)
- 고정 트래픽 애플리케이션

---

### 2.2 AWS EC2

**강점**
- 12개월 프리 티어 (t3.micro)
- 무제한 확장성 (Auto Scaling)
- 강력한 보안 (VPC, Security Group, IAM)
- 통합 생태계 (RDS, S3, CloudWatch)
- 국내 지원 (한국 리전)

**약점**
- 복잡한 설정 (보안, 네트워킹)
- 프리 티어 이후 비용 급증 ($5-8/월)
- 용량 추정 어려움
- 과금 모니터링 필수 (무한 요금 가능)

**추천 대상**
- 중규모 팀
- 확장성 필요 애플리케이션
- 엔터프라이즈 환경

---

### 2.3 Azure VMS

**강점**
- 12개월 프리 티어 (B1s)
- 강력한 네트워킹 (VNet, NSG)
- 자동 백업 옵션
- Office 365 통합
- 국내 가능 (제한적)

**약점**
- 복잡한 용어 (Resource Group, VNet, etc.)
- 프리 티어 이후 비용 높음 ($7-10/월)
- 한국 리전 미지원 (동남아 리전만)
- 로그인 절차 복잡

**추천 대상**
- Microsoft 스택 사용자
- 엔터프라이즈 IT 팀
- 초기 프로토타입

---

### 2.4 GCP Compute Engine

**강점**
- 영구 프리 티어 (e2-micro, 월 745시간)
- 최고 성능 (Graviton 없음, 그 대신 최신 프로세서)
- 간편한 CLI (gcloud)
- 최고의 모니터링 (자동, 세밀함)
- 영구 무료 제공

**약점**
- 24/7 운영 시 2개 이상의 인스턴스는 유료
- IP 주소 비용 ($0.005/시간 사용하지 않을 때)
- 리전 선택 제한 (아시아 선택지 적음)
- 크레딧 프로그램 복잡

**추천 대상**
- 스타트업 (프리 티어 최대화)
- Google Cloud 학습자
- 정확한 사용량 추적 필요 조직

---

### 2.5 Hetzner

**강점**
- 가장 저렴한 유료 옵션 (€2.99/월)
- 뛰어난 성능 (NVMe SSD)
- 유럽 및 아시아 리전
- 간결한 UI
- 빠른 프로비저닝 (3-5분)
- Terraform 지원 우수

**약점**
- 프리 티어 없음
- 자동 확장 미지원
- 한국 리전 없음 (싱가포르만)
- 영어 UI만 (대부분 영어 커뮤니티)
- 기술 지원 부족 (커뮤니티 중심)

**추천 대상**
- 개발자 (Terraform 활용)
- 유럽 기반 스타트업
- 장기 안정적 비용 필요 조직

---

### 2.6 Oracle Free Tier

**강점**
- 영구 무료 (1 OCPU, 1GB RAM)
- 20GB 블록 스토리지 무료
- 300GB 데이터 전송 무료/월
- 강력한 보안 (VCN, 방화벽)
- 진정한 "Always Free" (신용카드 필요 하지만 과금 안 함)

**약점**
- 아시아 리전 제한 (인도 리전만)
- 복잡한 UI (자동화 필수)
- 한국 지원 미흡
- 커뮤니티 작음
- Always Free 인스턴스 한 개 한정

**추천 대상**
- 교육/학습용
- 장기 프로젝트 (비용 안 내고 싶음)
- 다양한 클라우드 경험 원하는 개발자

---

## 3. 메신저 호스팅 최적 조합

### 3.1 Telegram 봇 (가장 인기)

**최고 비용 효율**: GCP + Cloud Run

```bash
# 배포 (약 5분)
gcloud run deploy telegram-bot \
  --source . \
  --platform managed \
  --region us-central1 \
  --memory 512Mi \
  --cpu 1

# 비용: $0 (프리 티어 범위, 월 200만 요청)
```

**2순위: Hetzner + Docker**

```bash
# Hetzner CX11 (€2.99/월)
# Docker 컨테이너로 배포
docker run -d \
  -e TELEGRAM_TOKEN=xxxxx \
  --restart=always \
  telegram-bot:latest

# 비용: €2.99/월 (일정)
```

**3순위: AWS Lightsail (초보자)**

```bash
# 웹 콘솔에서 Lightsail 인스턴스 생성
# SSH 접속 후 설치
apt update && apt install python3-pip
pip install pyTelegramBotAPI
python3 bot.py

# 비용: $3.50/월 (512MB 인스턴스)
```

### 3.2 WhatsApp 봇 (기업용)

**최고 비용 효율**: AWS EC2 (12개월 프리 티어)

```bash
# t3.small (1 vCPU, 2GB)
# 12개월 무료, 이후 ~$10/월

aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.small \
  --key-name my-key

# Node.js + Express 설치
npm install whatsapp-web.js express
node bot.js

# 비용: $0 (12개월), 이후 ~$10
```

**2순위: Azure VM (12개월 프리 티어)**

```bash
# B1s (1 vCPU, 1GB)
# 12개월 무료

az vm create \
  --resource-group myGroup \
  --name whatsapp-bot \
  --image UbuntuLTS

# 비용: $0 (12개월), 이후 ~$7
```

**3순위: GCP (비용 증가)**

```bash
# e2-medium 필요 (512MB 메모리로 WhatsApp 불충분)
# 월 $8-10

gcloud compute instances create whatsapp-bot \
  --machine-type e2-medium \
  --zone us-central1-a

# 비용: $8-10/월
```

---

## 4. 메신저 호스팅 권장 설정 (심층)

### 4.1 최종 추천 조합 (가성비 기준)

#### Case 1: Telegram 개인 프로젝트 (< 1,000 사용자)

**선택**: GCP e2-micro + Cloud Run (또는 Compute Engine)

```yaml
플랫폼: GCP Compute Engine
인스턴스: e2-micro (무료, 월 745시간)
운영 체제: Ubuntu 22.04 LTS
언어: Python 3.9
메신저: Telegram Bot API (Webhook)

설정 단계:
1. GCP 계정 생성 (프리 크레딧 $300)
2. gcloud CLI 설치
3. gcloud auth login
4. 인스턴스 생성 (1분)
5. SSH 접속
6. Python bot 설치
7. systemd 서비스 등록
8. HTTPS (Certbot) 설정
9. Telegram Webhook 등록

월 비용: $0 (영구)
배포 시간: 5-10분
장점:
- 진정한 무료
- 자동 모니터링
- 간단한 스케일링
- 한국 지원 가능 (asia-northeast1)

단점:
- 744시간 초과 시 과금 시작
- IP 주소 고정 비용 ($0.005/시간 미사용)
```

#### Case 2: Telegram 상업용 봇 (1,000-10,000 사용자)

**선택**: Hetzner CX11 + Docker

```yaml
플랫폼: Hetzner Cloud
인스턴스: CX11 (1 vCPU, 1GB RAM, 25GB SSD)
운영 체제: Ubuntu 22.04 LTS
컨테이너: Docker
언어: Python 3.9 (또는 Node.js)

설정 단계:
1. Hetzner 계정 생성
2. SSH 키 생성
3. 인스턴스 생성 (1-2분)
4. Docker 설치
5. Telegram 봇 Dockerfile 작성
6. 이미지 빌드
7. 컨테이너 실행 (--restart=always)
8. Nginx 리버스 프록시 설정
9. Let's Encrypt 인증서 설정
10. Telegram Webhook 등록

월 비용: €2.99 (~$3.25)
배포 시간: 10-15분
장점:
- 가장 저렴한 유료 옵션
- 뛰어난 성능 (NVMe)
- 자동 백업 ($2-3/월 추가)
- 전 세계 리전 (싱가포르 아시아)
- Terraform 지원

단점:
- 프리 티어 없음
- 자동 확장 미지원
- 기술 지원 부족
```

#### Case 3: WhatsApp 비즈니스 봇 (10,000+ 사용자)

**선택**: AWS EC2 t3.small (프리 12개월 후 Hetzner로 마이그레이션)

```yaml
초기 12개월 (AWS EC2):
  플랫폼: AWS EC2
  인스턴스: t3.small (1 vCPU, 2GB, 프리 티어)
  언어: Node.js 18 (빠른 반응)
  미들웨어: Express.js
  데이터베이스: DynamoDB (프리 25GB)
  메시징: SQS (프리 120만 요청/월)

설정:
1. AWS 계정 생성
2. VPC 생성 (1 default VPC)
3. Security Group 설정
4. t3.small 인스턴스 시작
5. Elastic IP 할당 ($0.005/시간 미사용)
6. Node.js + Express 설치
7. nginx 리버스 프록시
8. PM2 프로세스 관리자
9. CloudWatch 알람 설정
10. WhatsApp Cloud API Webhook 등록

월 비용 (초기 12개월): $0
           (이후): ~$15 (t3.small $5 + 데이터전송 $5 + 스토리지 $5)

배포 시간: 15-20분
장점:
- 12개월 무료
- 높은 신뢰도
- 자동 확장 가능
- CloudWatch 모니터링
- 한국 리전 사용 가능

단점:
- 프리 기간 후 비용 증가
- 초기 설정 복잡
- Security Group 관리 필수

---

12개월 이후 (Hetzner로 마이그레이션):
  플랫폼: Hetzner CX21
  사양: 2 vCPU, 4GB, 40GB SSD
  월 비용: €5.99 (~$6.50)

  마이그레이션:
  1. Hetzner 인스턴스 프로비저닝
  2. RDS → RDS/Managed PostgreSQL 마이그레이션
  3. 트래픽 전환 (DNS)
  4. AWS 리소스 삭제
  5. 다운타임: 2-5분 (DNS 전파)
```

---

## 5. 보안 체크리스트 (메신저봇 배포 전)

### 필수 (Security Level 1)

- [ ] SSH 키 설정 (암호 인증 비활성화)
- [ ] 방화벽 설정 (22/tcp는 특정 IP만, 443/tcp는 모두)
- [ ] OS 업데이트 (`apt update && apt upgrade`)
- [ ] 악성 소프트웨어 스캔 (ClamAV)
- [ ] 자동 백업 활성화 (스냅샷 일 1회)
- [ ] 암호화 활성화 (EBS/Disk encryption)
- [ ] 로그 수집 (CloudWatch/Azure Monitor)

### 권장 (Security Level 2)

- [ ] Fail2Ban 설치 (SSH 무차별 대입 방어)
- [ ] UFW 방화벽 강화 (OS 수준)
- [ ] 자동 보안 업데이트 활성화 (unattended-upgrades)
- [ ] 메신저 API 토큰 저장 (환경변수/Vault, 코드 제외)
- [ ] WAF 설정 (CloudFlare Free 사용)
- [ ] DDoS 보호 (플랫폼 기본값 사용)
- [ ] 감사 로깅 활성화 (auditd)

### 고급 (Security Level 3)

- [ ] WASM 샌드박스 (IronClaw 통합)
- [ ] 컨테이너 이미지 스캔 (Trivy)
- [ ] Network Policy 설정 (egress whitelist)
- [ ] Secrets Management (HashiCorp Vault)
- [ ] API Rate Limiting (메신저 API별)
- [ ] 침입 탐지 (Snort/Suricata)
- [ ] 정기적 보안 감사 (매월)

---

## 6. 배포 비용 시뮬레이션 (1년 기준)

### 시나리오 A: 개인 Telegram 봇 (500 사용자)

```
GCP e2-micro:
  월: $0 × 12 = $0
  연: $0

Hetzner CX11:
  월: €2.99 × 12 = €35.88 (~$39)
  연: ~$39

AWS Lightsail:
  월: $3.50 × 12 = $42
  연: $42

AWS EC2 (프리 이후):
  처음 12개월: $0
  이후: $5 × 12 = $60
  연: $60 (13개월 기준)

추천: GCP ($0) > Hetzner ($39) > AWS Lightsail ($42)
```

### 시나리오 B: 상업 Telegram 봇 (5,000 사용자)

```
GCP e2-small (프리 범위 초과):
  월: $15 (e2-small $3 + 데이터 $5 + 스토리지 $7)
  연: $180

Hetzner CX11:
  월: €2.99 + 백업 €2 = €4.99 (~$5.50)
  연: $66

Hetzner CX21 (스케일링):
  월: €5.99 + 백업 €3 = €8.99 (~$10)
  연: $120

AWS EC2 t3.small (프리 이후):
  처음 12개월: $0
  이후: $10 (t3.small + 네트워크)
  연: $60 (13개월 기준)

추천: AWS EC2 ($60 프리 + $60 유료) > Hetzner CX11 ($66) > GCP ($180)
```

### 시나리오 C: WhatsApp 비즈니스 봇 (10,000 사용자)

```
AWS EC2 t3.small (프리 12개월):
  처음 12개월: $0
  이후: $15/월 (t3.small $5 + 네트워크 $5 + DB $5)
  연: $60 (프리 12개월) + $180 (이후 12개월) = $240 (2년)

Hetzner CX21:
  월: €5.99 + 백업 €3 = €8.99 (~$10)
  연: $120

Azure B1 (프리 12개월):
  처음 12개월: $0
  이후: $10/월
  연: $120 (2년)

GCP e2-medium:
  월: $8 (프리 범위 초과)
  연: $96

추천: AWS EC2 프리 (처음 12개월, $0) > Hetzner CX21 ($120/년) > Azure ($120/년)
```

---

## 7. 단계별 배포 가이드

### 7.1 Telegram 봇 (GCP 기반, 5분 배포)

```bash
# Step 1: GCP 계정 및 프로젝트 설정 (2분)
gcloud init
gcloud config set project YOUR_PROJECT_ID

# Step 2: Compute Engine 인스턴스 생성 (1분)
gcloud compute instances create telegram-bot \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --machine-type=e2-micro \
  --zone=us-central1-a

# Step 3: 인스턴스 접속 (30초)
gcloud compute ssh telegram-bot --zone=us-central1-a

# Step 4: 환경 설정 (1.5분)
sudo apt update
sudo apt install python3-pip python3-venv
python3 -m venv venv
source venv/bin/activate
pip install pyTelegramBotAPI

# Step 5: 봇 코드 작성 및 실행 (30초)
cat > bot.py << 'EOF'
import telebot
import os

bot = telebot.TeleBot(os.getenv('TELEGRAM_TOKEN'))

@bot.message_handler(commands=['start'])
def send_welcome(message):
    bot.reply_to(message, "안녕하세요!")

bot.polling()
EOF

# Step 6: 서비스 등록 (30초)
sudo tee /etc/systemd/system/telegram-bot.service > /dev/null << 'EOF'
[Unit]
Description=Telegram Bot
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu
Environment="TELEGRAM_TOKEN=YOUR_TOKEN"
ExecStart=/home/ubuntu/venv/bin/python3 /home/ubuntu/bot.py
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable telegram-bot
sudo systemctl start telegram-bot

# Step 7: 확인 (30초)
sudo systemctl status telegram-bot
```

### 7.2 WhatsApp 봇 (AWS EC2 기반, 20분 배포)

```bash
# Step 1: AWS 계정 및 기본 설정
aws configure

# Step 2: VPC 및 보안 그룹 생성
aws ec2 create-security-group \
  --group-name whatsapp-bot-sg \
  --description "WhatsApp Bot Security Group"

aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxx \
  --protocol tcp --port 22 --cidr YOUR_IP/32 \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# Step 3: Key Pair 생성
aws ec2 create-key-pair --key-name whatsapp-bot-key > whatsapp-bot-key.pem
chmod 400 whatsapp-bot-key.pem

# Step 4: EC2 인스턴스 시작
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.small \
  --key-name whatsapp-bot-key \
  --security-group-ids sg-xxxxxxxx \
  --monitoring Enabled=true

# Step 5: Elastic IP 할당
aws ec2 allocate-address --domain vpc
aws ec2 associate-address --instance-id i-xxxxxxxx --allocation-id eipalloc-xxxxxxxx

# Step 6: SSH 접속 (공개 IP 확인)
INSTANCE_IP=$(aws ec2 describe-instances --instance-ids i-xxxxxxxx \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)
ssh -i whatsapp-bot-key.pem ubuntu@$INSTANCE_IP

# Step 7: 환경 설정
sudo apt update && sudo apt upgrade -y
sudo apt install nodejs npm certbot python3-certbot-nginx -y
sudo npm install -g pm2

# Step 8: WhatsApp 봇 코드
mkdir ~/whatsapp-bot && cd ~/whatsapp-bot
npm init -y
npm install express whatsapp-web.js qrcode
npm install --save-dev nodemon

cat > index.js << 'EOF'
const express = require('express');
const { Client, LocalAuth } = require('whatsapp-web.js');
const qrcode = require('qrcode');

const app = express();
const client = new Client({ authStrategy: new LocalAuth() });

client.on('qr', (qr) => {
  qrcode.toFile('qr.png', qr);
  console.log('QR code generated: qr.png');
});

client.on('ready', () => {
  console.log('Client is ready!');
});

client.on('message', (msg) => {
  if (msg.body == '!ping') {
    msg.reply('pong');
  }
});

client.initialize();

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
EOF

# Step 9: PM2 서비스 등록
pm2 start index.js --name "whatsapp-bot"
pm2 startup
pm2 save

# Step 10: Nginx + HTTPS 설정
sudo certbot certonly --standalone -d YOUR_DOMAIN.com
sudo tee /etc/nginx/sites-available/whatsapp-bot > /dev/null << 'EOF'
server {
  listen 443 ssl;
  server_name YOUR_DOMAIN.com;

  ssl_certificate /etc/letsencrypt/live/YOUR_DOMAIN.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/YOUR_DOMAIN.com/privkey.pem;

  location / {
    proxy_pass http://localhost:3000;
  }
}
EOF

sudo systemctl enable nginx
sudo systemctl restart nginx

# Step 11: CloudWatch 모니터링 설정
aws ec2 monitor-instances --instance-ids i-xxxxxxxx
```

---

## 8. 결론 및 최종 추천

### 비용별 순위

1. **무료 (영구)**: Oracle Free Tier (제한 많음) = GCP e2-micro (월 745시간)
2. **저가 ($3-5/월)**: Hetzner CX11 = AWS Lightsail 512MB
3. **중가 ($5-10/월)**: Hetzner CX21 = AWS EC2 (프리 이후)
4. **고가 ($15+/월)**: Azure = GCP (프리 범위 초과)

### 용도별 최종 추천

| 용도 | 1순위 | 2순위 | 비용 | 배포 시간 |
|------|-------|-------|------|----------|
| **Telegram 개인** | GCP e2-micro | Hetzner CX11 | $0 | 5분 |
| **Telegram 상업** | Hetzner CX11 | AWS Lightsail | €2.99 | 10분 |
| **WhatsApp 초기** | AWS EC2 프리 | Azure B1 프리 | $0 (12개월) | 20분 |
| **WhatsApp 장기** | Hetzner CX21 | AWS EC2 | €5.99 | 10분 |
| **최고 보안** | AWS EC2 + IronClaw | GCP + Armor | $5-15 | 20분 |
| **최고 확장성** | GCP + MIG | AWS + ASG | $0-10 | 2분 |

### 메신저 호스팅에 최적인 조합

**가성비 최고**: Hetzner CX11 + NanoClaw (컨테이너)
```
- 월 비용: €2.99 (~$3.25)
- 배포 시간: 10분
- 보안: 컨테이너 격리 (Tier 2)
- 확장성: 수동 (CX21로 업그레이드)
```

**초기 프로토타입**: GCP e2-micro + Telegram Bot API
```
- 월 비용: $0 (영구)
- 배포 시간: 5분
- 보안: VPC 기반 (Tier 1)
- 확장성: 자동 (추가 인스턴스 필요)
```

**장기 프로덕션**: AWS EC2 t3.small (12개월 프리) → Hetzner CX21
```
- 초기 12개월: $0
- 이후: €5.99/월 (~$6.50)
- 배포 시간: 15분
- 보안: Security Group + VPC (Tier 1)
- 확장성: Auto Scaling 지원
```

---

## 참고 자료

### 공식 문서
- AWS Lightsail: https://lightsail.aws.amazon.com/
- AWS EC2: https://docs.aws.amazon.com/ec2/
- Azure VMS: https://docs.microsoft.com/en-us/azure/virtual-machines/
- GCP Compute Engine: https://cloud.google.com/compute/docs
- Hetzner Cloud: https://docs.hetzner.cloud/
- Oracle OCI: https://docs.oracle.com/en-us/iaas/

### 메신저 API 문서
- Telegram Bot API: https://core.telegram.org/bots/api
- WhatsApp Cloud API: https://developers.facebook.com/docs/whatsapp/cloud-api/

### 보안 가이드
- AWS Security Best Practices: https://aws.amazon.com/security/best-practices/
- CIS Benchmarks: https://www.cisecurity.org/cis-benchmarks/
- OWASP Top 10: https://owasp.org/www-project-top-ten/

---

**마지막 업데이트**: 2026년 3월 14일
**대상 독자**: Claw 에이전트 프레임워크 배포자, 메신저봇 개발자
**다음 태스크**: Task #7 - 메신저봇 배포 자동화 스크립트 (Terraform)

---

## On-Premise Apple Silicon & 저비용 미니 PC (2026-03 meetup 추가)

### On-Premise Apple Silicon — M3 Ultra 사례

**소스**: 진주성 발표 (07_진주성_M3_Ultra_SNS_크롤링)

| 항목 | M3 Ultra 512GB | H100 서버 (동급) |
|------|---------------|----------------|
| **통합 메모리** | 512 GB | 7대 필요 (80GB VRAM × 7) |
| **구매 비용** | 약 1,600만 원 | 훨씬 고가 + 서버실 비용 |
| **전력 소비** | 낮음 (Mac mini 폼팩터) | 매우 높음 |
| **MLX 4-bit 양자화 후** | Qwen 122B → 약 70GB | — |
| **동시 서빙 가능** | 256GB 미만 (3개 모델 동시) | — |

**실제 서빙 모델 구성 (512GB 기준)**:

| 모델 | 크기 (MLX 4-bit) | 용도 |
|------|----------------|------|
| QWen 3.5 35B | 약 35GB | 게이트키핑 (액티브 3B MoE) |
| QWen 3.5 122B | 약 70GB | 복잡한 추론 |
| MiniMax | 약 128GB | 대용량 컨텍스트 |
| 합계 | < 256GB | 512GB 내 여유 운용 |

**이중 클러스터 구성**: 썬더볼트 풀메시 연결로 1TB+ 통합 메모리 구성 가능 (현재는 단일 Ultra 1대로 충분).

---

### 3-Tier 모델 전략 (로컬 + 클라우드 하이브리드)

**소스**: 진주성 발표

```
[Tier 1: 게이트키핑]
 QWen 3.5 35B (로컬, 빠른 속도)
 → 대량 SNS 데이터 1차 필터링
 → 의미 없는 데이터 제거

[Tier 2: 복잡한 추론]
 QWen 3.5 122B (로컬, 중간 복잡도)
 → 필터 통과한 데이터 분석

[Tier 3: 최종 의사결정]
 GPT-4.1 / Claude SOTA (클라우드)
 → 의미 있는 데이터만 전달
 → API 비용 집중 투입
```

**경제성**: 24시간 대규모 크롤링 시 클라우드 API만 사용 시 비용 감당 불가 → 로컬 LLM 1차 처리로 클라우드 API 호출 횟수를 1/10 이하로 감소.

**입문 권장 사양**: 128GB 통합 메모리 이상. QWen 3.5 35B 등 실용적 로컬 LLM 운용 가능.

---

### 17만원 중고 미니 PC — 초저비용 온프레미스

**소스**: 김우현 발표 (15_김우현_중고PC_세컨드브레인)

| 항목 | 사양/비용 |
|------|---------|
| **하드웨어** | 중고 미니 PC (RAM 16GB, 실사용 8GB 충분) |
| **구매 비용** | 17만 원 (11번가) |
| **Claude Code 구독** | 월 $20 (현재 $100 플랜) |
| **전기료** | 약 월 3,000원 |
| **결과** | TV 아래 소형 기기 1대로 24시간 AI 비서 운용 |

**소프트웨어 스택**:
- **OpenClaw + Obsidian + Discord** 삼각 구조
- **QMD 로컬 시맨틱 검색**: 자연어로 Obsidian 볼트 검색 (2~3년 후에도 활용 가능)
- **Nextcloud + WebDAV**: 멀티 디바이스 동기화 (Obsidian Sync 미사용, 비용 절감)
- **GitHub 백업**: 볼트 주기적 백업 → 마크다운 diff로 변경 이력 추적

**대비 비교 (OCI Free vs 미니 PC)**:

| 항목 | OCI Free Tier | 중고 미니 PC |
|------|--------------|------------|
| **초기 비용** | 0원 | 17만원 |
| **월 비용** | 0원 | ~3,000원 (전기료) |
| **로컬 LLM** | 불가 | 가능 (RAM 허용 범위) |
| **데이터 주권** | Oracle 서버 | 완전 온프레미스 |
| **Oracle 리스크** | 있음 | 없음 |
| **적합 용도** | 게이트웨이 서버 | 로컬 AI 비서 |
