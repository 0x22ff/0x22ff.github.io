---
layout: post
title: "BSSJ 아키텍처, 지금 구조를 이렇게 잡은 이유"
date: 2026-03-10 09:10:00 +0900
categories: [architecture]
tags: [spring-boot, vue, mysql, aws, nginx, monorepo]
---

이번 글은 `docs/architecture.md`(최종 수정일 2026-03-09) 기준으로, 현재 BSSJ 구조를 실제 운영 관점에서 정리한 내용이다.

문서 요약만 복붙한 게 아니라, 왜 이렇게 설계했는지까지 같이 적어본다.

## 한 번에 보는 전체 구조

![BSSJ 전체 아키텍처](/assets/images/bssj-architecture-overview.svg)

핵심은 딱 두 줄이다.

- 화면(Web/CMS)은 분리해서 UX를 명확히 가져간다.
- 서버 런타임은 통합해서 운영 비용과 복잡도를 줄인다.

즉, "프론트는 분리, 백엔드는 통합" 전략이다.

## 먼저 요구사항부터

서비스 관점에서 사용자 성격이 다르다.

1. 방문자: 교회 정보/영상/게시판을 보는 사용자
2. 관리자: CMS에서 콘텐츠를 등록/수정하는 사용자

이 둘의 화면 경험은 완전히 달라야 해서 FE는 분리(`apps/web/frontend`, `apps/cms/frontend`)했다.

반대로 BE는 운영 단순성이 더 중요해서, Spring Boot를 하나로 통합했다.

- 공개 API: `/api/*`
- CMS API: `/cms/*`
- 단일 포트: `:8080`

## 요청 흐름은 이렇게 돈다

### 방문자 요청

1. 사용자가 Public Web 접속
2. Nginx가 정적 리소스 제공
3. 화면에서 `/api/*` 호출
4. Unified Backend가 MySQL 조회 후 JSON 응답

### 관리자 요청

1. 관리자가 CMS 접속
2. Nginx가 CMS 정적 리소스 제공
3. 로그인 시 `POST /cms/auth/login`으로 JWT 발급
4. 이후 `/cms/*` 호출 시 Bearer 토큰 검증
5. 데이터는 MySQL, 파일은 S3에 저장

여기서 중요한 건, 관리자 API만 인증이 걸려 있다는 점이다.

## 백엔드 모듈 구조 (실제로는 이게 핵심)

![BSSJ 모듈 의존 구조](/assets/images/bssj-module-dependency.svg)

문서 기준 모듈은 아래 4개다.

- `apps:web:backend`
- `apps:cms:backend`
- `apps:common:mysql`
- `apps:common:core`

의존 관계는 이렇게 정리된다.

- `web/backend` -> `cms/backend`
- `web/backend` -> `common/mysql` -> `common/core`
- `cms/backend` -> `common/mysql`

중요 포인트:

- `web/backend`만 실행 모듈이다.
- `cms/backend`는 `java-library`라 단독 실행하지 않는다.
- `./gradlew :apps:web:backend:bootRun`으로 기동하면 `/api`와 `/cms`가 같이 열린다.

처음 보면 "왜 굳이 합쳤지?" 싶지만, 현재 운영 리소스(단일 EC2) 기준에선 꽤 합리적이다.

## 패키지 구조는 레이어가 아니라 도메인 기준

예전처럼 controller/service/repository를 전역으로 나누면, 수정할 때 파일 점프가 너무 많아진다.

그래서 2026-03 기준으로 도메인 중심 구조로 전환했다.

### `com.bssj.webapi`

- `video`, `content`, `churchinfo`, `settings`, `menu`, `modal`, `manager`, `common`

### `com.bssj.cms`

- `auth`, `video`, `bulletin`, `menu`, `staff`, `worshiptime`, `mainpage`, `upload`, `bible`, `manager`, `common`

특히 `cms/manager`는 `domain/port/infra`로 나눠서 hexagonal 아키텍처를 적용했다.

## 보안/권한 설계

문서 기준 정책은 명확하다.

- Web API (`/api/*`): 인증 없음(공개 조회)
- CMS API (`/cms/*`): JWT 인증 필수 (`/cms/auth/**` 제외)

그리고 JWT에 `cid`(churchId)를 넣어서 교회 단위 데이터 격리를 가져간다.

이 부분이 중요한 이유는, BSSJ가 단일 교회 사이트를 넘어서 멀티 교회 확장 가능성을 고려하기 때문이다.

## 데이터 계층

- DB: MySQL 8.0 (`utf8mb4`, ngram 전문검색)
- 마이그레이션: Flyway (`V1 ~ V12`)
- 파일: AWS S3 (ap-northeast-2)

정리하면,

- 정형 데이터 -> MySQL
- 이미지/파일 -> S3

이렇게 책임을 나눠서 앱 메모리 부담과 운영 리스크를 낮췄다.

## 인프라/배포

현재 운영 기준은 단일 EC2(t3.small)다.

- Nginx: 정적 파일 서빙 + 리버스 프록시
- Backend: Unified Spring Boot `:8080`
- MySQL: 로컬
- S3: 외부 스토리지

CI/CD는 GitHub Actions로 분리했다.

- CI(`develop` PR): 백엔드 테스트 + FE 빌드
- CD(`main` push): 빌드 -> SCP 전송 -> EC2 반영 -> `systemctl restart`

환경 프로필은 `local`, `beta`, `release`이고, 문서 기준으로 `release`는 아직 미사용 상태다.

## 이 구조의 장단점

### 장점

1. 운영 단순성: 프로세스 하나로 배포/모니터링 포인트 감소
2. 비용 절감: JVM/Tomcat/Hikari 이중화 제거
3. 개발 속도: FE 분리로 팀 작업 충돌 감소

### 단점(주의할 점)

1. 통합 앱이라 경계 관리 실패 시 빠르게 비대화됨
2. `/api`와 `/cms` 정책 분리를 느슨하게 하면 보안 사고 위험

그래서 지금 기준 운영 원칙은 이거다.

- 프로세스는 통합
- 모듈 경계는 강하게
- 인증 정책은 URL 규칙으로 강제

## 다음 단계

문서와 현재 운영 상황 기준으로, 다음 우선순위는 아래가 맞다.

1. CMS 업로드를 presigned URL 중심으로 전환
2. Swagger를 `/api`와 `/cms` 기준으로 그룹 명확화
3. 멀티교회 권한 모델(전역 관리자/교회 관리자) 구체화

---

한 줄 정리하면,

지금 BSSJ 아키텍처는 "처음부터 거대한 분산 시스템"을 노린 구조가 아니라, **작은 팀이 실제로 운영 가능한 구조를 먼저 고정한 설계**다.

그리고 그 위에 확장을 얹을 수 있게, 모듈 경계와 도메인 경계는 계속 지키는 방향으로 가고 있다.
