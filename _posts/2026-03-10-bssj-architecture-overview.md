---
layout: post
title: "BSSJ 아키텍처 한눈에 보기 (2026-03 기준)"
date: 2026-03-10 23:10:00 +0900
categories: [architecture]
tags: [spring-boot, vue, mysql, aws, nginx, monorepo]
---

BSSJ는 교회 공개 웹사이트(Web)와 관리자 CMS를 함께 운영하는 서비스다.  
2026년 3월 기준, 핵심 방향은 **백엔드를 단일 프로세스로 통합**하고 프론트엔드는 목적별로 분리하는 구조다.

## 1) 전체 구조

```text
사용자 브라우저
  ├─ Web 사용자      → web 도메인
  └─ CMS 관리자      → cms 도메인
          │
          ▼
        Nginx (:80)
  ├─ /api/*      → Unified Backend (:8080)
  ├─ /cms/*      → Unified Backend (:8080)
  ├─ /*          → Web Frontend 정적 파일
  └─ cms.*/*     → CMS Frontend 정적 파일
          │
          ▼
Unified Backend (Spring Boot)
  ├─ Web API 모듈   (/api/*)
  └─ CMS API 모듈   (/cms/*, JWT)
          │
          ├─ MySQL 8.0
          └─ AWS S3 (이미지)
```

포인트는 간단하다.

- FE는 `web`, `cms`를 분리해서 사용자 경험을 분명히 나눈다.
- BE는 `8080` 단일 앱으로 묶어서 운영 리소스와 배포 복잡도를 줄인다.

## 2) 모노레포 모듈 구성

`settings.gradle` 기준 현재 모듈은 다음 4개다.

```text
apps:web:backend
apps:cms:backend
apps:common:core
apps:common:mysql
```

역할은 이렇게 나뉜다.

- `apps:web:backend`: 실행 엔트리포인트(실제 부팅 앱)
- `apps:cms:backend`: CMS 기능 모듈(라이브러리처럼 포함)
- `apps:common:mysql`: 공용 Entity/Repository/DB 설정/Flyway
- `apps:common:core`: 공통 유틸리티

즉, 실행은 하나지만 기능 경계는 패키지와 모듈로 유지한다.

## 3) 백엔드 부팅 방식

현재 부팅 클래스는 `BssjWebApiApplication` 하나다.

- `com.bssj.webapi` + `com.bssj.cms`를 함께 컴포넌트 스캔
- JPA Entity/Repository도 두 패키지를 같이 스캔
- 결과적으로 `/api/*`와 `/cms/*`가 한 앱에서 동시에 제공됨

운영 관점에서 이 구조가 주는 장점:

- JVM/Tomcat/Hikari 중복 제거
- 배포 타깃 단순화(systemd 서비스 1개)
- 인프라 비용과 장애 포인트 감소

## 4) 인증/보안 전략

Web API와 CMS API의 성격이 다르기 때문에 인증 정책도 분리했다.

- Web API (`/api/*`): 공개 조회 중심
- CMS API (`/cms/*`): JWT 기반 인증
- 로그인 엔드포인트: `/cms/auth/login`

CMS JWT에는 교회 식별자(`churchId` 계열 클레임)를 담아 멀티 교회 데이터 분리를 지원한다.

## 5) 데이터와 파일

- DB: MySQL 8.0
- 스키마 관리: Flyway
- 업로드: AWS S3

구성은 고전적이지만 안정적이다.  
트랜잭션/조회는 DB, 정적 미디어는 S3로 책임을 분리해 백엔드 메모리 부담을 줄인다.

## 6) 배포 구조

EC2 단일 인스턴스(t3.small)에서 운영한다.

- Nginx 리버스 프록시
- Unified Backend (`bssj-web` 서비스)
- MySQL 로컬
- Web/CMS FE 정적 파일 서빙

작은 팀/초기 단계에서 “운영 단순성”을 우선한 현실적인 선택이다.

## 7) 다음 개선 포인트

현재 구조가 작동하는 것과, 장기적으로 더 강해지는 것은 별개다.  
다음 단계에선 아래를 우선 고려할 계획이다.

- CMS 업로드를 Presigned URL 중심으로 고도화
- API 문서 그룹(`/api`, `/cms`) 운영 규칙 표준화
- 멀티 교회 권한 모델(전역 관리자/교회 관리자) 명확화
- 배포 파이프라인의 점진적 무중단 배포 전략 도입

---

정리하면, BSSJ는 지금 **“분리된 UX + 통합된 런타임”** 전략으로 가고 있다.  
개발 속도와 운영 효율을 먼저 확보하고, 이후 확장성은 모듈 경계와 데이터 모델로 대응하는 접근이다.
