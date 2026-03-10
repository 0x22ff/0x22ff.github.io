---
layout: post
title: "BSSJ 아키텍처 정리 (왜 지금 이렇게 구성했는지)"
date: 2026-03-10 09:10:00 +0900
categories: [architecture]
tags: [spring-boot, vue, mysql, aws, nginx, monorepo]
---

이번 글은 `docs/architecture.md` 기준으로 다시 정리했다.  
문서 톤 말고, 실제로 왜 이렇게 짰는지 운영 관점으로 풀어보면 아래랑 같다.

## TL;DR

지금 BSSJ 구조를 한 줄로 말하면 이거다.

- FE는 `Web`/`CMS`를 분리해서 경험을 나눔
- BE는 `Spring Boot` 단일 프로세스로 통합해서 운영비를 줄임
- 데이터는 `MySQL`, 파일은 `S3`로 책임 분리
- 인증은 CMS만 JWT, 공개 API는 오픈

즉, "화면은 분리, 서버 런타임은 통합" 전략이다.

## 1) 시스템을 어떻게 쪼갰는지

서비스는 크게 두 사용자군이다.

1. 일반 방문자: 교회 소개/영상/게시판 보는 사용자
2. 관리자: CMS에서 콘텐츠 등록/수정하는 사용자

구성은 이렇게 잡았다.

```text
브라우저
  ├─ Public Web (Vue 3 + TS)
  └─ CMS        (Vue 3 + TS)
          │
          ▼
        Nginx :80
  ├─ /*             -> Web 정적 파일
  ├─ cms.*/*        -> CMS 정적 파일
  ├─ /api/*         -> Backend :8080
  └─ /cms/*         -> Backend :8080
          │
          ▼
Unified Backend (Spring Boot 3.x)
  ├─ /api/* (공개 조회)
  └─ /cms/* (JWT 인증)
          │
          ├─ MySQL 8.0
          └─ AWS S3
```

포인트는, FE는 나눴지만 BE 프로세스는 하나라는 점이다.

## 2) 모노레포 모듈 구조

백엔드는 멀티모듈인데, 실행 앱은 하나다.

```text
apps/
├── web/backend        # 실제 실행 앱 (bootRun 대상)
├── cms/backend        # java-library (단독 실행 X)
└── common/
    ├── core           # 공통 유틸
    └── mysql          # 엔티티/리포지토리/DB/Flyway
```

의존 관계는 이렇다.

- `web/backend` -> `cms/backend`
- `web/backend` -> `common/mysql` -> `common/core`
- `cms/backend` -> `common/mysql`

이렇게 둔 이유는 간단하다.

- 프로세스는 1개로 유지해서 서버 메모리/운영 복잡도 줄이고
- 코드 경계는 모듈로 나눠서 확장 여지를 남긴다

## 3) 패키지 구조는 도메인 기준

레이어(controller/service/repository) 기준으로 쪼개면 파일이 흩어져서, 지금은 도메인 기준으로 모았다.

`com.bssj.webapi` 쪽 대표 도메인:

- `settings` (메인페이지 설정)
- `video` (sermon/shorts)
- `content` (bulletin/column/gallery)
- `churchinfo` (staff/worship-time/location/history)
- `menu`, `modal`, `manager`
- `common` (보안/헬스체크/이미지 다운로드)

`com.bssj.cms` 쪽 대표 도메인:

- `auth` (로그인/JWT)
- `video`, `bulletin`, `menu`, `staff`, `worshiptime`, `mainpage`
- `upload` (S3 업로드)
- `bible` (조회/검색)
- `manager` (여기는 hexagonal 구조로 domain/port/infra 분리)

실무에서 장점은, 한 도메인 수정할 때 이동 경로가 짧다는 점이다.

## 4) API 규칙 (가장 중요한 부분)

경로 규칙은 명확하게 분리했다.

- Public API: `/api/*`
- CMS API: `/cms/*`

인증 정책도 다르게 가져간다.

- `/api/*`는 공개 조회라서 인증 없음
- `/cms/*`는 JWT 필수 (`/cms/auth/**` 제외)

대표 엔드포인트는 아래 정도만 기억해도 된다.

- `POST /cms/auth/login` (토큰 발급)
- `GET /api/sermons`, `GET /api/shorts`
- `GET /api/bulletins`, `GET /api/columns`
- `GET /api/main-page/settings`, `GET /api/menus`
- `POST /cms/upload/image` (S3 업로드)

멀티 교회 대응 때문에 JWT에 `cid`(churchId)도 포함한다.  
핵심은 "로그인은 하나로 하되, 데이터는 church 단위로 분리"다.

## 5) 데이터/스토리지

현재 데이터 레이어는 안정적인 조합으로 갔다.

- DB: MySQL 8.0 (`utf8mb4`)
- ORM: JPA/Hibernate
- 마이그레이션: Flyway (`V1 ~ V12`)
- 파일: AWS S3 (이미지)

정리하면, 정형 데이터는 DB, 바이너리 파일은 S3.  
역할을 명확히 나눠서 백엔드 메모리 부담을 줄이고 운영도 단순화했다.

## 6) 인프라와 배포

현재는 EC2 단일 인스턴스(`t3.small`) 운영이다.

- Nginx: 정적 파일 + API 프록시
- Backend: 8080 단일 앱
- MySQL: 로컬
- S3: 외부 스토리지

CI/CD는 GitHub Actions로 분리했다.

- CI (`develop` 대상 PR): 백엔드 테스트 + FE 빌드
- CD (`main` push): 빌드 -> 전송 -> EC2 배포 -> 서비스 재기동

환경 프로필은 `local`, `beta`, `release`를 잡아뒀고, 문서 기준 `release`는 아직 미사용 상태다.

## 7) 왜 굳이 "통합 백엔드"로 갔는가

초기/성장 구간에서는 이게 꽤 현실적인 선택이었다.

1. JVM/Tomcat/Hikari를 두 벌 안 띄워도 된다
2. 배포 타깃이 1개라 장애 포인트가 줄어든다
3. 서버 자원(t3.small)에서 운영하기 유리하다

대신 trade-off도 있다.

1. 코드 경계 관리 못하면 앱이 비대해질 수 있음
2. API 정책 통일이 느슨하면 `/api`와 `/cms`가 섞일 위험이 있음

그래서 지금은 "프로세스는 통합, 모듈 경계는 강하게"를 기준으로 관리 중이다.

## 8) 다음에 바로 손볼 것

지금 기준으로 우선순위는 이렇게 본다.

1. CMS 업로드를 presigned URL 기반으로 전환
2. Swagger 그룹/보안정책을 `/api`와 `/cms` 기준으로 명확히 분리
3. 멀티교회 권한 모델(전역 관리자 vs 교회 관리자) 구체화

---

정리하면, BSSJ는 "큰 구조를 빨리 고정하고, 운영비를 낮추는" 쪽으로 설계했다.  
완벽한 분산 구조를 처음부터 가기보다, 지금 필요한 속도와 안정성을 먼저 챙긴 형태라고 보면 된다.
