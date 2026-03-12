---
layout: post
title: "BSSJ 아키텍처, 왜 지금 이 구조가 맞았는지"
date: 2026-03-10 09:10:00 +0900
categories: [architecture]
tags: [spring-boot, vue, mysql, aws, nginx, monorepo]
thumbnail: /assets/images/thumb-bssj-architecture.svg
---

서버 한 대로 서비스는 띄워야 하고, 운영 복잡도는 낮춰야 하고, 화면 경험은 분리해야 할 때가 있죠.  
BSSJ 아키텍처는 정확히 그 조건에서 출발했습니다.

이번 글은 `docs/architecture.md`(2026-03-09 기준)를 바탕으로, **왜 이 구조를 선택했는지**를 한 번에 이해되게 정리한 버전입니다.

## 30초 요약

- 사용자 화면은 `Web`과 `CMS`로 분리했습니다.
- 백엔드는 `Spring Boot 단일 프로세스`로 통합했습니다.
- 공개 API는 `/api/*`, 관리자 API는 `/cms/*`로 경계를 분리했습니다.
- 데이터는 `MySQL`, 파일은 `S3`로 역할을 나눴습니다.

한 줄로 말하면, **"UX는 분리하고, 런타임은 통합"** 전략입니다.

## 전체 구조 먼저 보기

![BSSJ 전체 아키텍처](/assets/images/bssj-architecture-overview.svg)

핵심 흐름은 단순합니다.

1. Web 사용자는 공개 화면과 공개 API를 사용
2. CMS 사용자는 관리자 화면과 인증 API를 사용
3. 두 API 모두 결국 같은 백엔드 프로세스로 들어옴
4. 백엔드는 MySQL/S3로 책임 분리

구조를 단순하게 유지한 덕분에, 배포와 장애 대응 포인트도 줄었습니다.

## 왜 FE는 분리하고, BE는 합쳤을까?

여기서 제일 많이 받는 질문이 이겁니다.

"어차피 하나의 서비스면 다 합치면 되는 거 아닌가?"

반대로 보면 더 명확합니다.

- **화면은 성격이 완전히 다릅니다.**
  방문자용 Web과 관리자용 CMS는 목적과 인터랙션이 다르기 때문에 분리하는 게 맞습니다.
- **운영은 단순할수록 유리합니다.**
  작은 팀에서 JVM을 두 벌로 나누면 배포/모니터링/장애 대응 비용이 빠르게 커집니다.

그래서 BSSJ는 아래처럼 정리했습니다.

- FE: `apps/web/frontend`, `apps/cms/frontend` 분리
- BE: `apps:web:backend` 실행 앱 하나로 통합

이 선택 덕분에, 지금 규모에서 필요한 속도와 안정성을 동시에 가져갈 수 있었습니다.

## 요청이 실제로 어떻게 흐르는지

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

포인트는 하나입니다.  
**인증은 CMS에만 강하게, 공개 조회는 가볍게** 가져간다는 점입니다.

## 백엔드 모듈 구조: 통합이지만 무질서는 아님

![BSSJ 모듈 의존 구조](/assets/images/bssj-module-dependency.svg)

실행은 하나지만, 내부 경계는 모듈로 분명히 나눴습니다.

- `apps:web:backend` (실행 엔트리)
- `apps:cms:backend` (java-library)
- `apps:common:mysql`
- `apps:common:core`

의존 관계는 아래처럼 고정됩니다.

- `web/backend -> cms/backend`
- `web/backend -> common/mysql -> common/core`
- `cms/backend -> common/mysql`

즉, **프로세스는 하나, 경계는 유지**가 이 구조의 핵심입니다.

## 패키지 구조도 도메인 기준으로 정리

초기에는 레이어별 구조(controller/service/repository)였지만, 수정 범위가 커질수록 파일 이동 비용이 컸습니다.  
그래서 현재는 도메인 기준으로 재정렬했습니다.

- `com.bssj.webapi`: `video`, `content`, `churchinfo`, `settings`, `menu`, `modal`, `manager`, `common`
- `com.bssj.cms`: `auth`, `video`, `bulletin`, `menu`, `staff`, `worshiptime`, `mainpage`, `upload`, `bible`, `manager`, `common`

특히 `cms/manager`는 `domain/port/infra`로 나눠서 hexagonal 패턴을 적용했습니다.

## 보안과 멀티 교회 관점에서 중요한 점

BSSJ의 인증 정책은 URL 경계와 같이 갑니다.

- `/api/*`: 인증 없음 (공개 조회)
- `/cms/*`: JWT 필수 (`/cms/auth/**` 제외)

그리고 JWT에는 `cid`(churchId)를 포함해 교회 단위 데이터 격리를 유지합니다.  
지금은 단일 운영에 가까워 보여도, 이 경계가 있어야 멀티 교회 확장 시 비용이 줄어듭니다.

## 인프라/배포는 "작게, 단단하게"

현재 운영 기준은 단일 EC2(t3.small)입니다.

- Nginx: 정적 파일 + 프록시
- Unified Backend: `:8080`
- MySQL: 로컬
- S3: 외부 스토리지

CI/CD도 복잡하게 늘리지 않고 목적 중심으로 구성했습니다.

- CI (`develop` PR): 빌드/테스트로 품질 가드
- CD (`main` push): 빌드 -> 전송 -> 서비스 재시작

"대규모 분산"보다 "빠르고 안정적인 운영"을 우선한 구조라고 보면 됩니다.

## 이 구조의 장점과 리스크

### 장점

1. 운영 포인트 감소: 프로세스/배포 대상 단순화
2. 비용 효율: JVM/Tomcat/Hikari 중복 제거
3. 개발 속도: FE 분리로 작업 충돌 최소화

### 리스크

1. 경계 관리 실패 시 단일 앱 비대화
2. `/api`와 `/cms` 정책이 흐려지면 보안/권한 리스크 증가

그래서 지금 운영 원칙은 명확합니다.

- 런타임 통합은 유지
- 모듈/도메인 경계는 더 강하게
- 인증 정책은 URL 규칙으로 지속 강제

## 다음 단계(우선순위)

1. CMS 업로드를 presigned URL 중심으로 전환
2. Swagger를 `/api`와 `/cms` 기준으로 명확히 분리
3. 멀티 교회 권한 모델(전역/교회 관리자) 구체화

---

정리하면, BSSJ 아키텍처는 "완벽한 이상형"보다 "지금 운영 가능한 최적점"에 맞춘 설계입니다.  
그리고 그 위에서 확장할 준비를 모듈 경계와 인증 경계로 차근히 깔아두고 있습니다.
