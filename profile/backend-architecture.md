# 백엔드 아키텍처 초안

> **상태**: 초안 — 초기 설계 참고용
> **작성일**: 2026-02-04

---

## 1. 아키텍처 결정: 모놀리틱 + 모듈 분리

### 1.1 결정 배경

기존 시스템 아키텍처 문서(lifeplan-docs)에서는 Auth/API/Policy/Notification을 별도 서비스로 나누는 MSA 형태로 설계되어 있었으나, 현재 상황을 고려하여 **모놀리틱 + 모듈 분리** 구조를 채택한다.

### 1.2 MSA를 채택하지 않는 이유

| MSA 이점 | 현재 상황에서 불필요한 이유 |
|----------|--------------------------|
| 서비스별 독립 배포 | 2명이 동시 배포 충돌할 일이 없음 |
| 서비스별 독립 스케일링 | MVP 목표 1,000 MAU, 단일 서버로 충분 |
| 장애 격리 | 서비스 간 네트워크 호출이 오히려 장애 포인트 추가 |
| 팀 간 독립 개발 | 팀이 1개(2명) |

MSA로 인해 추가되는 운영 부담:

- 서비스 간 통신 관리 (REST/gRPC)
- API Gateway 설정 및 유지
- 서비스별 Docker 이미지 빌드/배포
- 분산 로깅 (서비스 4개의 로그 추적)
- 서비스 간 인증 전파
- 분산 트랜잭션 처리
- 헬스체크 4배

2명이 MVP를 만들면서 이 인프라를 동시에 관리하면 제품 개발보다 인프라에 시간을 더 쓰게 된다.

---

## 2. 백엔드 구조

### 2.1 모놀리틱 + 모듈 분리

```
backend/
├── src/
│   ├── modules/
│   │   ├── auth/              인증/회원
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   └── auth.types.ts
│   │   ├── transaction/       가계부
│   │   │   ├── transaction.controller.ts
│   │   │   ├── transaction.service.ts
│   │   │   └── transaction.types.ts
│   │   ├── policy/            정책 매칭/조회
│   │   │   ├── policy.controller.ts
│   │   │   ├── policy.service.ts
│   │   │   ├── matching.service.ts
│   │   │   └── policy.types.ts
│   │   ├── chat/              챗봇/대화
│   │   │   ├── chat.controller.ts
│   │   │   ├── chat.service.ts
│   │   │   ├── intent.service.ts
│   │   │   ├── prompt.service.ts
│   │   │   └── chat.types.ts
│   │   ├── profile/           사용자 프로필
│   │   │   ├── profile.controller.ts
│   │   │   ├── profile.service.ts
│   │   │   └── profile.types.ts
│   │   └── notification/      알림
│   │       ├── notification.controller.ts
│   │       ├── notification.service.ts
│   │       └── notification.types.ts
│   ├── shared/
│   │   ├── db/                하나의 DB 커넥션
│   │   ├── middleware/        인증, 로깅, 에러 핸들링
│   │   └── utils/
│   └── main.ts                단일 진입점
├── package.json
└── Dockerfile                 이미지 1개
```

핵심 원칙:

- **모듈 간 의존은 service 레이어를 통해서만** — controller가 다른 모듈의 service를 직접 import하지 않음
- **DB는 하나** — 모든 모듈이 같은 PostgreSQL을 사용
- **배포는 하나** — 서버 1대, Docker 이미지 1개

### 2.2 전체 시스템 구성

```
┌────────────┐
│ Flutter App │
└──────┬─────┘
       │ HTTPS
       ▼
┌──────────────────────────────────────┐
│         백엔드 API (모놀리틱)          │
│                                       │
│  auth / transaction / policy / chat   │
│  profile / notification               │
│                                       │
└────────┬──────────────┬──────────────┘
         │              │
         ▼              ▼
  ┌────────────┐  ┌────────────┐
  │ PostgreSQL │  │  Redis     │
  │            │  │  (캐시/    │
  │ - users    │  │   세션)    │
  │ - txns     │  └────────────┘
  │ - policies │
  │ - chats    │        ┌────────────┐
  │ - profiles │        │  LLM API   │
  └─────┬──────┘        └────────────┘
        │
        │ (DB 공유, 읽기 전용)
        │
┌───────┴──────┐
│  collector   │  별도 프로세스 (cron/CLI)
│  정책 수집    │  policies 테이블에 쓰기
└──────────────┘
```

collector는 이미 별도 프로세스로 분리되어 있으며,
백엔드 API와 DB만 공유한다 (collector가 쓰기, 백엔드가 읽기).

---

## 3. 데이터베이스 설계

단일 PostgreSQL에 모든 테이블을 배치한다.

### 3.1 테이블 그룹

```
[가계부 영역]
  users              사용자 계정
  user_profiles      프로필 (직접 입력 + 자동 추론)
  transactions       거래 내역 (수입/지출)
  budgets            예산 설정

[정책 영역 — collector가 관리]
  policies           정책 데이터
  change_logs        변경 이력
  collect_logs       수집 로그

[챗봇 영역]
  chat_sessions      대화 세션
  chat_messages      대화 메시지

[기타]
  user_policies      정책 즐겨찾기
  notifications      알림
```

### 3.2 Redis 용도

- 사용자 세션 (JWT refresh token)
- 정책 매칭 결과 캐시 (프로필 변경 전까지 유효)
- API 응답 캐시 (정책 목록 등)

---

## 4. API 엔드포인트 구조

모든 엔드포인트가 단일 서버에서 제공된다.

```
/api/v1
├── /auth
│   ├── POST   /register
│   ├── POST   /login
│   └── POST   /refresh
├── /transactions
│   ├── GET    /               (월별 조회)
│   ├── POST   /               (생성)
│   ├── PUT    /:id            (수정)
│   └── DELETE /:id            (삭제)
├── /budgets
│   ├── GET    /               (월별 예산 조회)
│   └── POST   /               (설정)
├── /profile
│   ├── GET    /               (프로필 조회)
│   └── PUT    /               (프로필 수정)
├── /policies
│   ├── GET    /               (목록/검색)
│   ├── GET    /:id            (상세)
│   ├── GET    /recommend      (프로필 기반 매칭)
│   ├── POST   /:id/favorite   (즐겨찾기 추가)
│   └── DELETE /:id/favorite   (즐겨찾기 해제)
├── /chat
│   ├── GET    /sessions       (세션 목록)
│   ├── POST   /sessions       (새 세션)
│   ├── GET    /sessions/:id   (메시지 목록)
│   └── POST   /sessions/:id   (메시지 전송 → 답변)
└── /notifications
    ├── GET    /               (알림 목록)
    └── PUT    /:id/read       (읽음 처리)
```

---

## 5. MSA 전환 조건

다음 조건 중 2개 이상 해당될 때 부분 전환을 검토한다:

- MAU 10,000 이상으로 특정 모듈에 부하 집중
- 팀이 3명 이상으로 성장하여 동시 개발 병목 발생
- 챗봇(LLM 호출)의 응답시간이 다른 API에 영향을 줌
- 특정 모듈의 배포 주기가 나머지와 크게 다름

전환 시에는 부하가 큰 모듈(예: chat)부터 하나씩 분리한다.
모듈 구조로 코드를 짜두면 이 전환은 service 레이어를 API 호출로 교체하는 수준이다.

---

## 6. 기술 스택 (미확정)

| 항목 | 후보 | 비고 |
|------|------|------|
| 런타임 | Node.js (TypeScript) | collector와 언어 통일, 프론트(Flutter)와 분리 |
| 프레임워크 | NestJS / Fastify / Hono | 모듈 구조 지원 여부 기준으로 선택 |
| ORM | Drizzle | collector와 동일, 스키마 공유 용이 |
| DB | PostgreSQL | 이미 사용 중 |
| 캐시 | Redis | 세션 + API 캐시 |
| LLM | Claude / GPT | 챗봇 아키텍처 문서 참고 |
| 클라이언트 | Flutter | 크로스플랫폼 |

---

## 7. 미결정 사항

| 항목 | 선택지 | 비고 |
|------|--------|------|
| 백엔드 프레임워크 | NestJS vs Fastify vs Hono | 모듈 구조 + 러닝커브 기준 |
| 클라우드 플랫폼 | AWS vs GCP | 비용 + 무료 티어 비교 필요 |
| 인증 방식 | JWT 자체 구현 vs Firebase Auth | 개발 속도 vs 커스터마이징 |
| 파일 저장 | S3 / GCS | 영수증 이미지 등 (추후) |
