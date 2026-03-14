---
layout: post
title: "BSSJ 아키텍처, 왜 지금 이 구조가 맞았는지"
date: 2026-03-10 09:10:00 +0900
categories: [architecture]
tags: [spring-boot, vue, mysql, aws, nginx, monorepo]
thumbnail: /assets/images/thumb-bssj-architecture.svg
---

서버는 많지 않고, 운영은 단순해야 하고, 화면 경험은 분리하고 싶을 때가 있다.  
BSSJ 구조는 딱 그 조건에서 출발했다.

이 글은 `docs/architecture.md`(2026-03-09 기준) 내용을 바탕으로,  
왜 지금 구조를 이렇게 가져갔는지 한 번에 보이게 정리한 버전이다.

## 30초 요약

- 사용자 화면은 `Web` / `CMS`로 분리
- 백엔드는 `Spring Boot 단일 프로세스`로 통합
- 공개 API는 `/api/*`, 관리자 API는 `/cms/*`로 경계 분리
- 데이터 저장은 `MySQL`, 파일 저장은 `S3`로 역할 분리

한 줄로 정리하면 이거다.  
**"UX는 분리하고, 런타임은 통합한다."**

## 전체 구조 먼저 보기

![BSSJ 전체 아키텍처](/assets/images/bssj-architecture-overview.svg)

요청 흐름은 단순하다.

1. Web 사용자는 공개 화면 + 공개 API 사용
2. CMS 사용자는 관리자 화면 + 인증 API 사용
3. 두 경로 모두 같은 백엔드 프로세스로 들어옴
4. 백엔드가 MySQL/S3로 책임 나눠 처리

구조를 단순하게 잡아둔 덕분에 배포 포인트, 장애 대응 포인트도 같이 줄었다.

## 왜 FE는 분리하고, BE는 합쳤을까?

이 질문을 제일 많이 받는다.

"하나의 서비스면 그냥 전부 합치면 되는 거 아닌가?"

실제로는 반대로 보는 게 더 맞다.

- **화면 목적이 다르다.**  
  방문자용 Web과 관리자용 CMS는 사용자 흐름이 다르다. 그래서 FE는 분리하는 게 맞다.
- **운영은 단순해야 버틴다.**  
  작은 팀에서 JVM을 둘로 나누면 배포/모니터링/장애 대응 비용이 빠르게 커진다.

그래서 지금은 이렇게 정리했다.

- FE: `apps/web/frontend`, `apps/cms/frontend` 분리
- BE: `apps:web:backend` 하나로 통합 실행

현재 규모에서는 이 구성이 속도와 안정성 사이 균형이 제일 좋았다.

## 요청 흐름을 실제로 보면

### 방문자(Web)

1. 사용자가 Public Web 접속
2. Nginx가 정적 리소스 응답
3. 화면에서 `/api/*` 호출
4. 백엔드가 MySQL 조회 후 JSON 반환

### 관리자(CMS)

1. 관리자가 CMS 접속
2. Nginx가 CMS 정적 리소스 응답
3. `POST /cms/auth/login`으로 JWT 발급
4. 이후 `/cms/*` 호출 시 토큰 검증
5. 필요 시 S3 파일 처리

핵심은 명확하다.  
**인증은 CMS 경로에 집중하고, 공개 조회는 가볍게 유지한다.**

## 백엔드 모듈 구조: 통합이지만 경계는 유지

![BSSJ 모듈 의존 구조](/assets/images/bssj-module-dependency.svg)

실행 프로세스는 하나지만, 내부 모듈 경계는 분명하게 나눴다.

- `apps:web:backend` (실행 엔트리)
- `apps:cms:backend` (java-library)
- `apps:common:mysql`
- `apps:common:core`

의존 관계는 아래처럼 고정돼 있다.

- `web/backend -> cms/backend`
- `web/backend -> common/mysql -> common/core`
- `cms/backend -> common/mysql`

즉 구조의 핵심은 이거다.  
**프로세스는 하나, 경계는 명확하게.**

## 패키지 구조도 도메인 중심으로 정리

초기 레이어 구조(controller/service/repository)에서는 수정 범위가 넓어질수록 이동 비용이 컸다.  
그래서 지금은 도메인 기준으로 재정렬했다.

- `com.bssj.webapi`: `video`, `content`, `churchinfo`, `settings`, `menu`, `modal`, `manager`, `common`
- `com.bssj.cms`: `auth`, `video`, `bulletin`, `menu`, `staff`, `worshiptime`, `mainpage`, `upload`, `bible`, `manager`, `common`

그리고 `cms/manager`는 `domain/port/infra`로 나눠 hexagonal 패턴을 적용했다.

## 보안/멀티 교회 관점에서 중요한 점

인증 정책은 URL 경계와 같이 간다.

- `/api/*`: 인증 없음 (공개 조회)
- `/cms/*`: JWT 필수 (`/cms/auth/**` 제외)

JWT 안에 `cid`(churchId)를 담아 교회 단위 데이터 격리를 유지한다.  
지금은 단일 운영에 가까워도, 이 경계가 있어야 멀티 교회 확장 비용이 줄어든다.

## 인프라/배포: 작게 시작하고 단단하게 유지

현재 운영 기준은 단일 EC2(t3.small)다.

- Nginx: 정적 파일 + 프록시
- Unified Backend: `:8080`
- MySQL: 로컬
- S3: 외부 스토리지

CI/CD도 목적 중심으로 가져갔다.

- CI (`develop` PR): 빌드/테스트 품질 가드
- CD (`main` push): 빌드 -> 전송 -> 서비스 재시작

대규모 분산보다, 운영 안정성을 먼저 챙긴 구성이라고 보면 된다.

## 이 구조의 장점과 리스크

### 장점

1. 운영 포인트 감소: 프로세스/배포 대상 단순화
2. 비용 효율: JVM/Tomcat/Hikari 중복 제거
3. 개발 속도: FE 분리로 작업 충돌 최소화

### 리스크

1. 경계 관리 실패 시 단일 앱 비대화
2. `/api`와 `/cms` 정책이 흐려지면 보안/권한 리스크 증가

그래서 운영 원칙은 계속 이대로 가져간다.

- 런타임 통합 유지
- 모듈/도메인 경계 강화
- 인증 정책 URL 규칙 강제

## 다음 단계 우선순위

1. CMS 업로드를 presigned URL 중심으로 전환
2. Swagger를 `/api`와 `/cms` 기준으로 명확히 분리
3. 멀티 교회 권한 모델(전역/교회 관리자) 구체화

---

정리하면 BSSJ 구조는 "완벽한 이상형"보다,  
**지금 팀이 운영 가능한 최적점**을 목표로 잡은 아키텍처다.

그리고 확장을 위한 경계(모듈/인증)는 이미 깔아둔 상태다.
