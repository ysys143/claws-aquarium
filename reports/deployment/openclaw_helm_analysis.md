# OpenClaw Helm Chart 분석 보고서
## serhanekicii vs Chrisbattarbee

### 분석 대상
- **serhanekicii/openclaw-helm** (v1.4.9, AppVersion 2026.3.12)
  - 기반: bjw-s app-template (v4.6.2)
  - 저장소: https://github.com/serhanekicii/openclaw-helm

- **Chrisbattarbee/openclaw-helm** (v0.1.13, AppVersion 2026.3.2)
  - 커스텀 Helm 구현
  - 저장소: https://github.com/Chrisbattarbee/openclaw-helm

---

## 1. Helm 차트 구조 및 설정 복잡도

### serhanekicii (app-template 기반)

**복잡도: 높음** (640+ 라인 values.yaml)

**장점:**
- 라이브러리 차트(app-template) 활용으로 메인테넌스 부담 감소
- 깊이 있는 커스터마이제이션 가능
- 상세한 주석과 스키마 선언 (@schema)
- CLI 기반 설치 흐름 (devices approve, devices list)

**단점:**
- 값 구조가 `app-template:` 네스팅으로 복잡
- 의존성 관리 필요 (app-template 버전 고정)
- 초보자 입장에서 가파른 학습곡선

**값 구조 깊이:**
```yaml
app-template:
  openclawVersion: "2026.3.12"
  configMode: merge
  controllers:
    main:
      containers:
        main:
          resources:
            requests: {cpu: 200m, memory: 512Mi}
```
-> 최대 5단계 네스팅

---

### Chrisbattarbee (커스텀 구현)

**복잡도: 낮음** (270라인 values.yaml)

**장점:**
- 평탄한 값 구조 (image, openclaw, credentials 최상위)
- 의존성 없음 (순수 Helm 차트)
- Quick Start 3줄 명령어
- 초보자 친화적

**단점:**
- 커스텀 코드 유지보수 필요
- 기능 추가/변경 시 직접 템플릿 수정
- 동적 초기화 스크립트 관리 복잡

**값 구조 깊이:**
```yaml
image:
  repository: ghcr.io/openclaw/openclaw
  tag: "2026.3.2"
openclaw:
  agents:
    defaults:
      model: "anthropic/claude-sonnet-4-20250514"
credentials:
  anthropicApiKey: ""
```
-> 최대 3단계 네스팅

---

## 2. 프로덕션 준비도

### PersistentVolume & PVC

**serhanekicii:**
- 5Gi 기본값 (PVC 관리 강화)
- init-config, init-skills도 PVC 마운트하여 상태 보존 강력
- 액세스 모드: ReadWriteOnce (기술적으로만 가능, 단일 인스턴스)

**Chrisbattarbee:**
- 동일한 5Gi 기본값
- 커스텀 Storage Class 지정 가능

**프로덕션 권장:**
- 크기: 최소 20-50Gi (skills, sessions, memory 고려)
- Storage Class: SSD 권장 (EBS gp3, Azure Premium)
- 백업: PVC 스냅샷 일일 자동화

---

### Ingress & 네트워크 노출

**serhanekicii:**
```yaml
ingress:
  main:
    enabled: false
    annotations:
      nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```
- WebSocket 3600초 타임아웃 명시
- TLS 인증서 매니저 지원

**Chrisbattarbee:**
```yaml
ingress:
  enabled: false
  hosts:
    - host: openclaw.local
```
- 기본 예제만 제공
- WebSocket 타임아웃 미명시 (위험)

**평가:**
| 항목 | serhanekicii | Chrisbattarbee |
|------|-------------|----------------|
| WSS 타임아웃 | 명시 [O] | 미명시 [X] |
| cert-manager | 지원 [O] | 미제공 [X] |

---

### Network Policy

**serhanekicii (고급 지원):**
- 기본 거부(deny-all) + 명시적 허용 규칙
- DNS 트래픽 허용
- 공인 IP 허용, 사설 범위(RFC1918) 차단
- Calico/Cilium 필요

**Chrisbattarbee:**
- **NetworkPolicy 지원 안 함** (보안 갭)

**차이 분석:**
- serhanekicii: 감염 시 피해 범위 제한 가능 (SCORE: 4/5)
- Chrisbattarbee: 전체 클러스터 접근 가능 (SCORE: 2/5)

---

## 3. 확장성 (HPA, 상태 저장)

### 수평 스케일링

**양쪽 차트 모두: 불가능**

```yaml
replicas: 1
strategy: Recreate
```

**이유:** OpenClaw는 상태 머신 -> 단일 인스턴스만 지원

**확장 전략:**
1. **수직 확장(Vertical)**: 리소스 제한 증가
2. **멀티 에이전트 배포**: 별도 네임스페이스에 독립 인스턴스
3. **Redis 기반 세션 공유** (미지원, 커스텀 필요)

---

### 상태 저장 정보 관리

**두 차트 모두 PVC에 저장:**
- `/home/node/.openclaw/openclaw.json` - 설정
- `/home/node/.openclaw/workspace/skills/` - 설치된 스킬
- `/home/node/.openclaw/sessions/` - 세션 상태
- `/home/node/.openclaw/devices/` - 페어링 정보

**권장 백업 전략:**
```bash
# PVC 스냅샷 (일일)
kubectl create volumesnapshot openclaw-backup-$(date +%Y%m%d) \
  --source=pvc/openclaw-pvc

# 또는 Velero
velero backup create openclaw-$(date +%Y%m%d) --include-namespaces openclaw
```

---

## 4. 보안 설정

### 컨테이너 보안 컨텍스트

**serhanekicii:**
```yaml
securityContext:
  runAsUser: 1000
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```
**평가:** EXCELLENT (모든 컨테이너 hardened)

**Chrisbattarbee:**
```yaml
securityContext:
  runAsUser: 1000
  runAsNonRoot: true
  readOnlyRootFilesystem: false  # [WARN] 쓰기 가능 = 위험
  seccompProfile:
    type: RuntimeDefault
```
**평가:** GOOD (seccompProfile 강화, 하지만 readOnlyFS 약함)

**주요 차이:**
| 항목 | serhanekicii | Chrisbattarbee |
|------|-------------|----------------|
| readOnlyRootFS (main) | [O] true | [X] false |
| seccompProfile | (미명시) | RuntimeDefault |

**보안 위협:**
- Chrisbattarbee의 readOnlyRootFilesystem: false -> Pod 손상 후 파일 수정 가능

---

### 네트워크 격리

**serhanekicii:** SCORE 4/5
- NetworkPolicy 템플릿 제공 (기본 비활성화)
- ingress 규칙, egress 규칙 상세함

**Chrisbattarbee:** SCORE 2/5
- NetworkPolicy 미지원

**피해 범위 비교:**
- serhanekicii: 클러스터 내 격리 가능
- Chrisbattarbee: 전부 허용 (감염 시 피해 확산)

---

## 5. 메시징 채널 통합

### Telegram/WhatsApp 설정

**serhanekicii:**
```yaml
# values.yaml에 주석 템플릿
"channels": {
  "telegram": {
    "botToken": "${TELEGRAM_BOT_TOKEN}",
    "enabled": true
  }
}
```
- 모든 채널 선택사항 미리 정의
- 복잡도: 중간 (언커멘트 + 환경변수 생성)

**Chrisbattarbee:**
```yaml
openclaw:
  configOverrides: {}
```
- 사용자가 JSON 직접 작성
- 복잡도: 낮음 (유연성 높음)

**평가:**
| 채널 | serhanekicii | Chrisbattarbee |
|------|-------------|----------------|
| Telegram | 템플릿 제공 | configOverrides |
| Discord | 템플릿 제공 | configOverrides |
| WhatsApp | 없음 | configOverrides (유연) |
| 커스텀 | 수동 추가 | 편함 |

---

## 6. 24/7 운영 (가용성)

### Liveness Probe 설정

**serhanekicii:**
```yaml
liveness:
  enabled: true
  type: TCP
  spec:
    initialDelaySeconds: 30
    periodSeconds: 30  # 30초마다 체크
    failureThreshold: 3  # 90초 미응답 후 재시작
```

**Chrisbattarbee:**
```yaml
liveness:
  enabled: true
  initialDelaySeconds: 30
  periodSeconds: 10  # 10초마다 체크 (더 빠름)
  failureThreshold: 3  # 60초 미응답 후 재시작
```

**비교:**
| 메트릭 | serhanekicii | Chrisbattarbee |
|------|-------------|----------------|
| 체크 간격 | 30초 | 10초 |
| 감지 시간 | ~120초 | ~60초 |
| SLA 관점 | 낮음 | 높음 |

---

### 리소스 제한

**serhanekicii:**
```yaml
main:
  requests: {cpu: 200m, memory: 512Mi}
  limits: {cpu: 2000m, memory: 2Gi}
chromium:
  requests: {cpu: 100m, memory: 256Mi}
  limits: {cpu: 1000m, memory: 1Gi}
```

**Chrisbattarbee:**
```yaml
main:
  requests: {cpu: 100m, memory: 512Mi}  # CPU 요청 낮음
  limits: {cpu: 2000m, memory: 2Gi}
chromium:
  requests: {cpu: 100m, memory: 256Mi}
  limits: {cpu: 1000m, memory: 1Gi}
```

**평가:**
- serhanekicii CPU 요청 높음 -> 노드 리소스 부족 가능성 낮음
- Chrisbattarbee CPU 요청 낮음 -> 스케줄링 용이하지만 경합 시 성능 저하

**프로덕션 권장:**
```yaml
requests: {cpu: 300m, memory: 768Mi}
limits: {cpu: 3000m, memory: 3Gi}
```

---

### 모니터링 & 가시성

**serhanekicii:**
- 로깅 설정 ConfigMap에 포함
- 디버그 모드 가능 (level: debug)

**Chrisbattarbee:**
- 로깅 설정 미제공

**권장 모니터링 스택:**
```yaml
# Prometheus 알람
- alert: OpenClawPodDown
  expr: up{job="openclaw"} == 0
- alert: OpenClawMemoryHigh
  expr: container_memory_usage_bytes > 1.8e9
- alert: OpenClawDiskSpace
  expr: usage / capacity > 0.8

# Loki 로그 집계
# Grafana 대시보드 (CPU, Memory, Session 활성 수)
```

---

## 7. 종합 평가 매트릭스

| 평가항목 | serhanekicii | Chrisbattarbee | 우위 |
|---------|-------------|----------------|------|
| **Helm 구조** | | |
| 학습곡선 | 가파름 | 완만함 | Chris [O] |
| 커스터마이제이션 | 매우 높음 | 중간 | Serhan [O] |
| 유지보수성 | 자동화 | 수동 | Serhan [O] |
| **프로덕션** | | |
| Ingress/TLS | SCORE 4 | SCORE 3 | Serhan [O] |
| NetworkPolicy | SCORE 4 | NOT SUPPORTED | Serhan [O] |
| **보안** | | |
| readOnlyRootFS | [O] ALL | [X] main/chromium | Serhan [O] |
| seccompProfile | default | RuntimeDefault | Chris [O] |
| **운영** | | |
| Liveness 빈도 | 30초 | 10초 | Chris [O] |
| 다운타임 | 120초 | 60초 | Chris [O] |
| **메시징** | | |
| Telegram 설정 | 템플릿 | configOverrides | Serhan [O] |
| WhatsApp 설정 | 없음 | configOverrides | Chris [O] |
| **종합점수** | **8.5/10** | **7.5/10** | **Serhan [O]** |

---

## 8. 권장사항

### serhanekicii 선택 권장
**적합한 조직:**
- 프로덕션 환경 (엔터프라이즈)
- 보안 감사 필수 (금융, 헬스케어)
- 자동화된 배포 (GitOps + ArgoCD)

**설정 체크리스트:**
- [ ] PVC 크기: 50Gi 이상
- [ ] Ingress: 활성화 + TLS
- [ ] NetworkPolicy: 활성화
- [ ] 모니터링: Prometheus + Grafana
- [ ] 백업: 일일 스냅샷 자동화

---

### Chrisbattarbee 선택 권장
**적합한 조직:**
- 빠른 프로토타이핑 (PoC)
- 소규모 팀 (운영 리소스 제한)
- 비용 최적화 필요

**추가 작업:**
- [ ] NetworkPolicy 수동 추가 필요
- [ ] readOnlyRootFilesystem 수정 권장
- [ ] 모니터링 구성 필요
- [ ] Liveness probe periodSeconds 조정 (30초 -> 10초)

---

## 9. 배포 전 체크리스트

### 배포 전
- [ ] NetworkPolicy 활성화 (Calico/Cilium)
- [ ] PVC 스토리지 클래스 확인 (SSD)
- [ ] 시크릿 저장소 설정 (Vault/Sealed Secrets)
- [ ] 백업 정책 수립
- [ ] RBAC 권한 최소화
- [ ] Pod Security Policy 설정

### 배포 후
- [ ] Pod 상태 확인: kubectl get pods
- [ ] 로그 확인: kubectl logs deployment/openclaw
- [ ] 게이트웨이 테스트: port-forward + 웹 UI
- [ ] PVC 사용량 모니터링
- [ ] 백업 자동화 테스트
- [ ] 장애 조치 테스트

---

## 10. 프로덕션 배포 샘플

### serhanekicii (권장)
```bash
helm repo add openclaw https://serhanekicii.github.io/openclaw-helm
helm repo update

kubectl create namespace openclaw
kubectl create secret generic openclaw-env \
  --from-literal=ANTHROPIC_API_KEY=sk-ant-xxx \
  -n openclaw

helm install openclaw openclaw/openclaw \
  --namespace openclaw \
  --set app-template.persistence.data.size=50Gi \
  --set app-template.networkpolicies.main.enabled=true \
  --set app-template.ingress.main.enabled=true \
  --wait
```

---

## 결론

**프로덕션 환경**: **serhanekicii/openclaw-helm** 강력 권장
- 보안, 네트워크 정책, 모니터링 가이드 완비
- app-template 기반으로 장기 유지보수성 우수
- 엔터프라이즈 수준의 구성 가능

**PoC/개발 환경**: **Chrisbattarbee/openclaw-helm** 권장
- 빠른 배포, 낮은 학습곡선
- 단, 보안 정책은 수동 강화 필요
- 기업 배포 전 업그레이드 권장

**하이브리드 권장:**
serhanekicii 차트 사용 + Chrisbattarbee의 간단한 설정 철학 결합

---

**작성일**: 2026-03-14
**분석 버전**: serhanekicii v1.4.9, Chrisbattarbee v0.1.13
**분석 초점**: 24/7 운영, 보안, 메시징 통합, Kubernetes 배포 모델
