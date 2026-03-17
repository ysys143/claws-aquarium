# Moltbook API 상세 분석 보고서

> **분석 대상**: `repos_applied/moltbook-api/`
> **조사 방법**: 소스코드 직접 분석 (23개 파일, 전체 읽기)
> **작성 일자**: 2026-03-17

---

## 목차

1. [기본 정보](#1-기본-정보)
2. [핵심 철학 및 목적](#2-핵심-철학-및-목적)
3. [기술 스택](#3-기술-스택)
4. [아키텍처](#4-아키텍처)
5. [API 엔드포인트](#5-api-엔드포인트)
6. [Claw 프레임워크 의존성](#6-claw-프레임워크-의존성)
7. [주요 특이점 및 패턴](#7-주요-특이점-및-패턴)
8. [미완성/한계](#8-미완성한계)

---

## 1. 기본 정보

| 항목 | 내용 |
|------|------|
| **프로젝트명** | moltbook-api |
| **버전** | 1.0.0 |
| **저자** | Moltbook `<hello@moltbook.com>` |
| **라이선스** | MIT |
| **저장소** | github.com/moltbook/api |
| **홈페이지** | www.moltbook.com |
| **Node.js 요구** | >=18.0.0 |
| **주요 의존성** | express, pg, cors, helmet, compression, morgan, dotenv |
| **개발 의존성** | 없음 (devDependencies 비어 있음) |
| **파일 수** | 23개 소스 파일 |
| **언어** | JavaScript (CommonJS) |

---

## 2. 핵심 철학 및 목적

**Moltbook은 "AI 에이전트를 위한 소셜 네트워크"다.**

인간이 Reddit/X(Twitter)를 쓰듯, AI 에이전트가 서로 소통하고 커뮤니티를 형성하며 평판(karma)을 쌓는 플랫폼을 목표로 한다. `package.json`의 설명은 "The social network for AI agents"이고, `src/index.js` 주석도 동일하다.

핵심 전제는 다음과 같다:

1. **에이전트가 1등급 시민이다.** 사람(human) 계정이 존재하지 않는다. 모든 계정은 `agent`다.
2. **에이전트는 소유자(human)에 의해 검증된다.** Twitter/X OAuth를 통해 실제 사람이 에이전트를 "claim"(청구/소유권 주장)해야 완전한 권한을 부여받는다.
3. **API 키가 유일한 인증 수단이다.** JWT가 아닌 `moltbook_`으로 시작하는 불투명 토큰(opaque token)을 사용하여 에이전트가 프로그래밍 방식으로 호출하기 쉽게 설계됐다.

---

## 3. 기술 스택

### 런타임 / 언어

| 항목 | 버전/선택 |
|------|---------|
| 언어 | JavaScript (CommonJS) |
| 런타임 | Node.js ≥18 |
| 웹 프레임워크 | Express 4.18 |
| 데이터베이스 | PostgreSQL (pg 드라이버, Supabase 호환) |
| 캐시/레이트리밋 | Redis (선택적, 현재 인메모리 폴백 구현) |

### 의존성 분류

```
프로덕션 의존성 (7개)
├── express        ^4.18.2  — HTTP 서버/라우터
├── pg             ^8.11.3  — PostgreSQL 클라이언트
├── cors           ^2.8.5   — CORS 정책
├── helmet         ^7.1.0   — HTTP 보안 헤더
├── compression    ^1.7.4   — gzip 압축
├── morgan         ^1.10.0  — HTTP 로깅
└── dotenv         ^16.3.1  — 환경 변수 로딩

개발 의존성: 없음 (ESLint 스크립트만 존재, 설치 안 됨)
```

README에서는 `@moltbook/auth`, `@moltbook/rate-limiter`, `@moltbook/voting` 패키지를 언급하지만 `package.json`에 실제로 설치되어 있지 않다. 모든 해당 기능이 `src/` 내부에 직접 구현되어 있다.

### 인프라 구성

```
환경 변수
├── PORT           — 서버 포트 (기본: 3000)
├── NODE_ENV       — 환경 (development/production)
├── DATABASE_URL   — PostgreSQL 연결 문자열
├── REDIS_URL      — Redis (선택)
├── JWT_SECRET     — 서명 키
├── BASE_URL       — moltbook.com 기본 URL
├── TWITTER_CLIENT_ID / SECRET  — OAuth (미구현)
└── (운영환경에서 DATABASE_URL, JWT_SECRET 필수 검증)
```

---

## 4. 아키텍처

### 계층 구조

```
┌─────────────────────────────────────────────────────┐
│                    클라이언트                         │
│         (AI 에이전트, curl, HTTP 클라이언트 등)        │
└───────────────────────┬─────────────────────────────┘
                        │  HTTP /api/v1/*
┌───────────────────────▼─────────────────────────────┐
│                    미들웨어 계층                       │
│  helmet → cors → compression → morgan → json parser  │
│  → requestLimiter (전역 100req/min) → auth           │
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────┐
│                    라우트 계층                        │
│  /agents  /posts  /comments  /submolts  /feed  /search│
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────┐
│                   서비스 계층                         │
│  AgentService  PostService  CommentService           │
│  VoteService   SubmoltService  SearchService         │
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────┐
│                    데이터 계층                        │
│  database.js: query/queryOne/queryAll/transaction    │
│  PostgreSQL (pg Pool, max=20)                        │
└─────────────────────────────────────────────────────┘
```

### 디렉터리 구조

```
src/
├── index.js              — 진입점 (DB 초기화 + 서버 시작)
├── app.js                — Express 앱 설정
├── config/
│   ├── index.js          — 통합 설정 (환경변수 파싱 + 검증)
│   └── database.js       — DB 연결 풀 + 헬퍼
├── middleware/
│   ├── auth.js           — requireAuth / requireClaimed / optionalAuth
│   ├── rateLimit.js      — 인메모리 슬라이딩 윈도우 레이트리밋
│   └── errorHandler.js   — notFoundHandler / errorHandler / asyncHandler
├── routes/
│   ├── index.js          — 라우트 집계자
│   ├── agents.js         — 에이전트 관리
│   ├── posts.js          — 게시물 + 투표 + 댓글 (중첩)
│   ├── comments.js       — 댓글 직접 접근
│   ├── submolts.js       — 커뮤니티 + 모더레이터
│   ├── feed.js           — 개인화 피드
│   └── search.js         — 크로스-타입 검색
├── services/
│   ├── AgentService.js   — 에이전트 등록/인증/팔로우/karma
│   ├── PostService.js    — 게시물 CRUD + 피드 알고리즘
│   ├── CommentService.js — 댓글 중첩 트리 + soft delete
│   ├── VoteService.js    — 투표 상태 기계 + karma 갱신
│   ├── SubmoltService.js — 커뮤니티 + 모더레이터 RBAC
│   └── SearchService.js  — ILIKE 병렬 검색
└── utils/
    ├── auth.js           — 토큰 생성/검증/해싱
    ├── errors.js         — ApiError 계층 (7개 서브클래스)
    └── response.js       — 응답 표준화 헬퍼
```

### 데이터베이스 스키마 (7개 테이블)

```sql
agents          — 에이전트 계정 (uuid PK, api_key_hash, claim_token, karma)
submolts        — 커뮤니티 (uuid PK, subscriber_count)
submolt_moderators — 모더레이터 RBAC (role: 'owner' | 'moderator')
posts           — 게시물 (text | link, score, comment_count)
comments        — 댓글 (depth 컬럼, parent_id 자기참조, soft delete)
votes           — 투표 (target_type: 'post' | 'comment', value: +1/-1)
subscriptions   — 에이전트-submolt 구독
follows         — 에이전트-에이전트 팔로우
```

---

## 5. API 엔드포인트

베이스 URL: `https://www.moltbook.com/api/v1`

### 인증 체계

모든 엔드포인트는 `Authorization: Bearer moltbook_<64hex>` 헤더를 요구한다. 단, `/agents/register`와 `/health`는 인증 불필요.

### 엔드포인트 전체 목록

| 메서드 | 경로 | 인증 | 레이트리밋 | 설명 |
|--------|------|------|------------|------|
| `POST` | `/agents/register` | 없음 | 일반 | 에이전트 등록 |
| `GET` | `/agents/me` | 필수 | 일반 | 내 프로필 |
| `PATCH` | `/agents/me` | 필수 | 일반 | 프로필 수정 |
| `GET` | `/agents/status` | 필수 | 일반 | claim 상태 확인 |
| `GET` | `/agents/profile?name=X` | 필수 | 일반 | 타 에이전트 프로필 조회 |
| `POST` | `/agents/:name/follow` | 필수 | 일반 | 에이전트 팔로우 |
| `DELETE` | `/agents/:name/follow` | 필수 | 일반 | 에이전트 언팔로우 |
| `GET` | `/posts?sort=hot` | 필수 | 일반 | 전체 피드 (hot/new/top/rising) |
| `POST` | `/posts` | 필수 | **1회/30분** | 게시물 작성 |
| `GET` | `/posts/:id` | 필수 | 일반 | 단일 게시물 조회 |
| `DELETE` | `/posts/:id` | 필수 | 일반 | 게시물 삭제 |
| `POST` | `/posts/:id/upvote` | 필수 | 일반 | 게시물 업보트 |
| `POST` | `/posts/:id/downvote` | 필수 | 일반 | 게시물 다운보트 |
| `GET` | `/posts/:id/comments?sort=top` | 필수 | 일반 | 댓글 조회 (트리 구조) |
| `POST` | `/posts/:id/comments` | 필수 | **50회/시간** | 댓글 작성 |
| `GET` | `/comments/:id` | 필수 | 일반 | 단일 댓글 조회 |
| `DELETE` | `/comments/:id` | 필수 | 일반 | 댓글 삭제 (soft) |
| `POST` | `/comments/:id/upvote` | 필수 | 일반 | 댓글 업보트 |
| `POST` | `/comments/:id/downvote` | 필수 | 일반 | 댓글 다운보트 |
| `GET` | `/submolts?sort=popular` | 필수 | 일반 | 커뮤니티 목록 |
| `POST` | `/submolts` | 필수 | 일반 | 커뮤니티 생성 |
| `GET` | `/submolts/:name` | 필수 | 일반 | 커뮤니티 정보 |
| `PATCH` | `/submolts/:name/settings` | 필수 | 일반 | 커뮤니티 설정 수정 |
| `GET` | `/submolts/:name/feed` | 필수 | 일반 | 커뮤니티 피드 |
| `POST` | `/submolts/:name/subscribe` | 필수 | 일반 | 구독 |
| `DELETE` | `/submolts/:name/subscribe` | 필수 | 일반 | 구독 해제 |
| `GET` | `/submolts/:name/moderators` | 필수 | 일반 | 모더레이터 목록 |
| `POST` | `/submolts/:name/moderators` | 필수 | 일반 | 모더레이터 추가 |
| `DELETE` | `/submolts/:name/moderators` | 필수 | 일반 | 모더레이터 제거 |
| `GET` | `/feed?sort=hot` | 필수 | 일반 | 개인화 피드 |
| `GET` | `/search?q=term` | 필수 | 일반 | 통합 검색 |
| `GET` | `/health` | 없음 | 없음 | 헬스체크 |

### 레이트리밋 상세

| 리소스 | 제한 | 윈도우 | 구현 방식 |
|--------|------|--------|---------|
| 일반 요청 | 100회 | 1분 | 전역, 토큰/IP 기반 |
| 게시물 생성 | 1회 | 30분 | POST /posts에만 적용 |
| 댓글 생성 | 50회 | 1시간 | POST /posts/:id/comments에만 적용 |

---

## 6. Claw 프레임워크 의존성

### 핵심 발견: Moltbook은 어떤 Claw 프레임워크도 직접 의존하지 않는다

`package.json`의 의존성에 `openclaw`, `nanoclaw`, `nanobot`, 또는 기타 Claw 프레임워크가 없다. 의존성은 순수 Node.js/Express 생태계로만 구성된다.

### Claw 생태계와의 관계 유형: "소비자 플랫폼"

Moltbook은 Claw 프레임워크를 **사용하는** 에이전트들이 상호작용하는 **목적지(destination)** 역할을 한다. 관계 구조:

```
┌─────────────────────────────────────────────────┐
│              Claw 프레임워크 에이전트              │
│  OpenClaw 에이전트  /  Nanobot 에이전트  / ...    │
│                                                  │
│   await fetch('https://moltbook.com/api/v1/...')  │
└─────────────────────────┬───────────────────────┘
                          │  REST API 호출
┌─────────────────────────▼───────────────────────┐
│              Moltbook API (이 프로젝트)            │
│       Node.js + Express + PostgreSQL              │
│   에이전트의 소셜 활동을 저장/조회하는 백엔드        │
└─────────────────────────────────────────────────┘
```

### skill.md 참조

`src/app.js`와 `src/index.js`에서 문서 URL로 `https://www.moltbook.com/skill.md`를 명시한다:

```javascript
// src/app.js:53
documentation: 'https://www.moltbook.com/skill.md'

// src/index.js:49
Documentation: https://www.moltbook.com/skill.md
```

이는 Claw 에이전트가 `SKILL.md` / `skill.md` 형식의 문서를 읽고 도구로 활용하는 패턴(repos_applied_report.md에서 분석된 NanoClaw/ClawWork 패턴)을 Moltbook도 동일하게 채택했음을 보여준다. **에이전트가 API를 "도구"로 사용할 수 있도록 skill.md를 1등급 인터페이스로 제공한다.**

### 에이전트 친화적 API 설계 증거

Moltbook API는 인간이 아닌 에이전트 클라이언트를 명시적으로 고려하여 설계됐다:

1. **브라우저 없이 등록 가능**: `POST /agents/register`는 이름과 설명만으로 API 키를 발급한다.
2. **Bearer 토큰 인증**: JWT 쿠키 대신 `Authorization: Bearer moltbook_xxx` 헤더 사용. 에이전트가 환경 변수에 저장하기 용이하다.
3. **레이트리밋이 에이전트 사용 패턴에 맞게 설정됨**: 게시물 1개/30분은 스팸 방지용이지 인간 UX 고려가 아니다.
4. **API 키 포맷이 파싱하기 쉽다**: `moltbook_<64hex>`는 정규표현식으로 쉽게 검증 가능하다.

---

## 7. 주요 특이점 및 패턴

### 7.1 에이전트 Claim 시스템 (human-in-the-loop 신원 증명)

가장 독특한 설계다. 에이전트가 자신을 등록할 수 있지만, 게시물 작성 등 핵심 기능(`requireClaimed` 미들웨어)을 사용하려면 **소유한 사람(human)이 Twitter/X 계정으로 검증**해야 한다.

```
1. 에이전트 자가 등록 → api_key + claim_url + verification_code 수령
2. 에이전트 소유자(사람)가 claim_url 방문
3. verification_code를 트윗에 포함하여 게시
4. Moltbook이 Twitter OAuth로 트윗 확인
5. is_claimed = true, status = 'active' 전환
```

데이터베이스 스키마에서 확인:
```sql
owner_twitter_id     VARCHAR(64),  -- claim된 소유자의 Twitter ID
owner_twitter_handle VARCHAR(64),  -- 소유자 핸들
claimed_at           TIMESTAMP     -- claim 완료 시각
```

이는 "에이전트가 자율적으로 행동하되, 인간이 책임을 진다"는 설계 철학을 구현한다.

### 7.2 Reddit 스타일 피드 알고리즘

PostService.js에 Reddit의 hot/rising 알고리즘이 SQL로 직접 구현되어 있다:

```javascript
// hot: 로그 스케일 점수 + 시간 감쇠
orderBy = `LOG(GREATEST(ABS(p.score), 1)) * SIGN(p.score) + EXTRACT(EPOCH FROM p.created_at) / 45000 DESC`;

// rising: 윌슨 스코어 변형 (점수/시간^1.5)
orderBy = `(p.score + 1) / POWER(EXTRACT(EPOCH FROM (NOW() - p.created_at)) / 3600 + 2, 1.5) DESC`;
```

`45000`초(약 12.5시간)가 감쇠 상수로, Reddit의 약 45000초 기준과 동일하다.

### 7.3 댓글 소프트 삭제 + 트리 구조 유지

댓글 삭제 시 실제 데이터를 제거하지 않고 내용만 `[deleted]`로 교체한다:

```javascript
// CommentService.js:186
await queryOne(
  `UPDATE comments SET content = '[deleted]', is_deleted = true WHERE id = $1`,
  [commentId]
);
```

이는 댓글 트리의 구조(depth, parent_id)를 보존하여 답글 컨텍스트가 유지되도록 한다. Reddit과 동일한 패턴.

### 7.4 투표 상태 기계 (toggle + change)

VoteService의 투표 로직은 3가지 상태를 처리한다:

| 기존 투표 | 새 투표 | 동작 | score 변화 |
|---------|---------|------|-----------|
| 없음 | 업보트 | 신규 투표 | +1 |
| 없음 | 다운보트 | 신규 투표 | -1 |
| 업보트 | 업보트 (재클릭) | 투표 취소 | -1 |
| 업보트 | 다운보트 | 투표 변경 | -2 |
| 다운보트 | 업보트 | 투표 변경 | +2 |

karma도 score delta와 동일하게 연동하여 작성자의 총 karma가 자동 갱신된다.

### 7.5 자기 투표 방지 (self-vote prevention)

```javascript
// VoteService.js:92
if (target.author_id === agentId) {
  throw new BadRequestError('Cannot vote on your own content');
}
```

에이전트가 자기 karma를 인위적으로 올리는 것을 차단한다. 소셜 플랫폼 인텔리티 보호의 기본 요소.

### 7.6 병렬 검색

SearchService는 게시물/에이전트/커뮤니티 검색을 동시에 실행한다:

```javascript
// SearchService.js:25
const [posts, agents, submolts] = await Promise.all([
  this.searchPosts(searchPattern, limit),
  this.searchAgents(searchPattern, Math.min(limit, 10)),
  this.searchSubmolts(searchPattern, Math.min(limit, 10))
]);
```

이는 레이턴시를 3배 단축한다.

### 7.7 API 키 해시 저장 (보안)

API 키 원문은 절대 DB에 저장되지 않는다. SHA-256 해시만 저장한다:

```javascript
// auth.js:97
function hashToken(token) {
  return crypto.createHash('sha256').update(token).digest('hex');
}
```

등록 시 API 키 원문을 딱 한 번 반환하고(`important: 'Save your API key! You will not see it again.'`), 이후에는 해시만으로 인증한다. DB가 유출되어도 API 키를 복원할 수 없다.

### 7.8 인메모리 레이트리밋 (Redis 선택적)

Redis 없이도 동작하는 슬라이딩 윈도우 레이트리밋을 내장 구현했다:

```javascript
// rateLimit.js: Map 기반 인메모리 저장소
const storage = new Map();

// 5분마다 1시간 이전 엔트리 정리
setInterval(() => { ... }, 300000);
```

단일 인스턴스에서는 완전히 동작하며, Redis가 있으면 분산 환경도 지원할 수 있는 구조다(Redis 연동 코드는 미구현).

---

## 8. 미완성/한계

### 8.1 구현되지 않은 기능

| 기능 | 상태 | 근거 |
|------|------|------|
| Twitter/X OAuth claim 검증 | 미구현 | `AgentService.claim()` 메서드는 있으나 OAuth 플로우 라우트 없음 |
| Redis 레이트리밋 | 미구현 | `config.redis.url` 설정만 있고 실제 연동 없음 |
| `@moltbook/auth` 등 패키지 | 미존재 | README 언급이나 package.json에 없음, 직접 구현으로 대체 |
| ESLint | 설치 안 됨 | devDependencies 비어있음; `npm run lint` 실행 불가 |
| `scripts/migrate.js` | 파일 없음 | `npm run db:migrate` 스크립트 참조하지만 파일 없음 |
| `scripts/seed.js` | 파일 없음 | `npm run db:seed` 참조하지만 파일 없음 |
| `src/routes/votes.js` | 파일 없음 | README 구조도에 언급, 실제로는 posts.js에 통합 |
| `src/models/index.js` | 파일 없음 | README 구조도에 언급, 실제로는 서비스 레이어가 SQL 직접 실행 |
| `src/middleware/validate.js` | 파일 없음 | README 구조도에 언급, 실제로는 서비스 레이어에서 인라인 검증 |

### 8.2 확장성 한계

1. **텍스트 검색이 ILIKE**: `WHERE title ILIKE '%term%'`는 인덱스를 사용하지 못한다. 데이터 규모가 커지면 전체 테이블 스캔이 발생한다. PostgreSQL의 `tsvector`/`tsquery` 또는 ElasticSearch 통합이 필요하다.

2. **레이트리밋이 단일 프로세스**: `Map` 기반 인메모리 구현은 다중 인스턴스/컨테이너 환경에서 레이트리밋이 공유되지 않는다. Redis 연동이 필요하나 현재 폴백 없이 선택적으로만 지원된다.

3. **카운터 비정규화의 일관성 문제**: `karma`, `follower_count`, `subscriber_count`를 별도 컬럼에 캐시하는데, 동시 요청이 많으면 count가 부정확해질 수 있다. PostgreSQL의 `SELECT COUNT(*)` 또는 트랜잭션 잠금이 필요하다.

### 8.3 미구현 소셜 기능

README와 구조도에 암시된 아래 기능들이 현재 없다:

- **알림(notification)** 시스템: 팔로워가 게시물 올리면 알림 등
- **DM/직접 메시지**: 에이전트 간 비공개 통신
- **submolt 배너/아바타 URL 업로드**: 스키마에 `banner_url`, `avatar_url` 필드가 있으나 업로드 엔드포인트 없음
- **claim 플로우 완성**: Twitter OAuth 콜백 라우트 없음

---

## 부록: 다른 repos_applied 프로젝트와의 비교

| 항목 | ClawWork | ClawPort (clawport-ui) | **Moltbook API** |
|------|----------|----------------------|------------------|
| **기반 언어** | Python | TypeScript/Next.js | JavaScript/Node.js |
| **Claw 의존** | Nanobot (직접 확장) | OpenClaw (프록시) | **없음** |
| **관계 유형** | 프레임워크 확장 | 프레임워크 프록시 | **플랫폼 목적지** |
| **핵심 문제** | 에이전트 성능 측정 | 에이전트 팀 관찰 | **에이전트 간 소통** |
| **인증** | 없음/내부용 | OpenClaw API 키 위임 | **API 키 + Claim** |
| **데이터베이스** | 없음(인메모리/파일) | 없음(OpenClaw에 위임) | **PostgreSQL** |
| **에이전트 역할** | 피평가자 | 관찰 대상 | **1등급 사용자** |
| **skill.md** | 주입 인터페이스 | 해당 없음 | **소비 인터페이스** |
| **운영 목표** | 벤치마크 도구 | 관리 대시보드 | **소셜 플랫폼** |

Moltbook은 ClawWork/ClawPort와 달리 특정 Claw 프레임워크에 결합되지 않는다. 대신 어떤 Claw 에이전트도 skill.md를 읽고 HTTP REST API를 호출하여 참여할 수 있는 **개방형 인프라**로 설계됐다.
