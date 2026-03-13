# 보안 분석 보고서 요약
## 중국 AI 에이전트 서비스 6개 vs 기존 Claw 프레임워크

**분석 완료**: 2026-03-14

---

## 빠른 참조: 보안 Tier

### Tier 1 (엔터프라이즈급)
- IronClaw, ZeroClaw
- 중국 서비스: **없음**

### Tier 2 (고급 보안)
- Alibaba OpenClaw
- Z.ai OpenClaw
- OpenClawD

### Tier 2-3 (중간 보안 - 진행 중)
- Baidu DuClaw (가장 진전, 하지만 미완성)

### Tier 3 (기본 보안)
- Kimi Claw
- Zhipu AutoClaw

---

## 5개 보안 영역 성적표

```
자격증명 관리      암호화    OS Keychain
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Baidu         [O]       [X]
Alibaba       [O] 선택   [X]
Z.ai          [X]       [X]
Kimi          [X]       [X]
Zhipu         [X]       [X]
OpenClawD     부분      [X]

샌드박싱       Docker    WASM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Baidu         [O]       [O] 평가
Alibaba       [O]       [X]
Z.ai          [O] 선택   [X]
OpenClawD     [O]       [X]
Kimi          [X]       [X]
Zhipu         [X]       [X]

권한 제어       도구별    데이터 흐름
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Baidu         [O] High/Mid/Low  [X]
Alibaba       [O] ACL         [O] 제한
Z.ai          [O] Whitelist   [X]
OpenClawD     [O] 감사 기반   [X]
Kimi          [X]            [X]
Zhipu         [O] 앱 수준    [X]

프롬프트 인젝션   전용 탐지   위협 분류
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Baidu         [O] 진행       [O] 부분
Z.ai          기본 필터      [X]
Alibaba       마킹           [X]
OpenClawD     입력 검증      [X]
Kimi          문자 이스케이프 [X]
Zhipu         입력 검증      [X]

HITL            구현        긴급 정지
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Baidu         [O] 워크플로우  [O] 즉시
Alibaba       [O] 검토 모드   [X]
Z.ai          [X]            [O] 신호
OpenClawD     [X]            [O] 로깅
Kimi          [X]            [X]
Zhipu         [X]            [X]
```

---

## 6가지 핵심 보안 갭

### [CRITICAL] 암호화 자격증명 저장소

**문제**: 5개 서비스가 평문 환경변수 또는 config 파일 사용
**영향**: 컨테이너 탈취 시 API 키 노출
**해결**: ChaCha20-Poly1305 암호화 + OS Keychain (Baidu 패턴 참고)
**소요 시간**: 1주일 (모든 서비스)

### [HIGH] 프롬프트 인젝션 탐지 레이어

**문제**: 전용 탐지 레이어 없음 (Baidu만 시작)
**영향**: 프롬프트 인젝션 공격 우회 용이
**해결**: regex 기반 Scanner (15+ 패턴) + 4개 위협 수준 분류
**소요 시간**: 1-2주 (Baidu, Alibaba, Z.ai)

### [HIGH] Taint Tracking (데이터 흐름 추적)

**문제**: 모든 6개 서비스 미구현
**영향**: 민감 데이터 (PII/SECRET) 유출 경로 차단 불가
**해결**: 4-label 추적 + SINK_POLICY (OpenJarvis 패턴)
**소요 시간**: 4-8주 (Baidu, Alibaba, OpenClawD)

### [HIGH] HITL (Human-in-the-Loop)

**문제**: Baidu만 부분, Alibaba는 검토 모드만
**영향**: 폭주 에이전트 제어 불가
**해결**: 3단계 AutonomyLevel + E-Stop (ZeroClaw 패턴)
**소요 시간**: 2-4주 (모든 서비스)

### [HIGH] WASM 샌드박스

**문제**: Baidu만 평가 단계, 나머지 없음
**영향**: 컨테이너 탈취 시 호스트 공격 가능
**해결**: WASM 격리 (10MB 메모리, 10억 fuel 제한)
**소요 시간**: 8-16주 (Baidu, Alibaba, Z.ai)

### [MEDIUM] 비용 하드 한도

**문제**: 모든 6개 서비스 구현 없음
**영향**: 24시간 에이전트에서 비용 폭발 위험
**해결**: 일별 하드 한도 ($5-10) + HITL 조합
**소요 시간**: 2주 (모든 서비스)

---

## 우선순위별 개선 로드맵

### Phase 1: 즉시 (1-2주)
1. 모든 서비스: 자격증명 암호화 (ChaCha20-Poly1305)
2. Baidu, Alibaba, Z.ai: 프롬프트 인젝션 Scanner 추가

### Phase 2: 단기 (2-4주)
1. Baidu, Alibaba: AutonomyLevel 도입
2. 모든 서비스: 비용 하드 한도 구현
3. 모든 서비스: 도구별 위험도 분류

### Phase 3: 중기 (4-8주)
1. Baidu, Alibaba, OpenClawD: Taint Tracking 도입
2. Alibaba, Z.ai: 자격증명 암호화 기본화

### Phase 4: 장기 (8-16주)
1. Baidu: WASM 샌드박스 완성
2. Alibaba, Z.ai: WASM 도입

---

## 각 서비스별 추천 사용 시나리오

| 서비스 | 현재 Tier | 추천 용도 | 주의사항 |
|--------|---------|--------|--------|
| **Baidu DuClaw** | 2-3 | 엔터프라이즈, 규제 산업 | WASM 완성 대기, 비용 한도 추가 필요 |
| **Alibaba OpenClaw** | 2 | 팀 협업, 채널 기반 | 자격증명 암호화 필수 활성화 |
| **Z.ai OpenClaw** | 2 | 통제된 도구 집합 (5-10) | 자격증명 암호화, HITL 추가 필요 |
| **OpenClawD** | 2-3 | 팀 협업 + 감사 추적 | 자격증명 암호화, HITL 추가 필요 |
| **Kimi Claw** | 3 | 개인 프로토타이핑만 | 보안 불충분 (엔터프라이즈 부적절) |
| **Zhipu AutoClaw** | 3 | 신뢰 환경 프로토타이핑 | 보안 불충분 (엔터프라이즈 부적절) |

---

## 기존 Claw 프레임워크와의 격차

### IronClaw / ZeroClaw (Tier 1)
- 암호화 자격증명 (AES-256-GCM 또는 ChaCha20-Poly1305)
- WASM + Docker 이중 샌드박스
- SafetyLayer / PromptGuard (다중 인젝션 방어)
- 정교한 HITL (3단계 AutonomyLevel, E-Stop)
- 비용 하드 한도 ($5/일)

### 중국 서비스의 격차
- Baidu: 암호화 있음, 하지만 WASM 미완, Taint Tracking 없음
- Alibaba: 채널별 정책 우수, 하지만 암호화 선택적
- 나머지: 기본 보안만 (Tier 3)

---

## 실행 가능한 개선 코드 샘플

### 1. 자격증명 암호화 (Rust)
```rust
use chacha20poly1305::ChaCha20Poly1305;

fn encrypt_credential(password: &str, api_key: &str) -> String {
    let key = derive_key_from_password(password);
    let cipher = ChaCha20Poly1305::new(Key::from(key));
    let nonce = Nonce::from_slice(&random_96_bits());
    cipher.encrypt(nonce, api_key.as_bytes()).unwrap()
}
```

### 2. 프롬프트 인젝션 Scanner (Python)
```python
import re
from enum import Enum

class ThreatLevel(Enum):
    LOW, MEDIUM, HIGH, CRITICAL = 1, 2, 3, 4

PATTERNS = [
    (r"ignore.*previous.*instructions", ThreatLevel.CRITICAL),
    (r"(password|api_key|secret).*show", ThreatLevel.CRITICAL),
]

def scan(user_input: str) -> ThreatLevel | None:
    for pattern, level in PATTERNS:
        if re.search(pattern, user_input, re.IGNORECASE):
            return level
    return None
```

### 3. 도구별 위험도 분류 (TypeScript)
```typescript
enum ToolRiskLevel { Low = "low", Medium = "medium", High = "high" }

async function execute_tool(tool: Tool, autonomy: string) {
    const policy = get_policy(tool.id);
    if (autonomy === "full" && policy.riskLevel === "high") {
        const approval = await request_approval();
        if (!approval) return error("Rejected");
    }
    return await tool.execute();
}
```

---

## 다음 단계

1. **이 보고서 공유**: Baidu, Alibaba, Z.ai, Zhipu 보안 담당자
2. **Phase 1 우선순위 수립**: 자격증명 암호화 + 프롬프트 인젝션 Scanner
3. **30일 후 재평가**: 각 서비스의 개선 진행 상황 추적
4. **90일 후 재분류**: 보안 Tier 업데이트

---

**문서 생성**: 2026-03-14
**마지막 검토**: 2026-03-14
**다음 리뷰**: 2026-04-14

상세 분석은 `chinese_ai_services_security_analysis.md` 참고.
