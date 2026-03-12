---
layout: default
title: Worklog
permalink: /worklog/
---

<section class="v-worklog">
  <section class="v-tabs" aria-label="navigation tabs">
    <a href="/">글</a>
    <a class="active" href="/worklog/">작업일지</a>
  </section>

  <header class="wl-header">
    <h1>작업일지 Timeline</h1>
    <p>프로젝트별로 묶어서 흐름을 볼 수 있게 정리한 작업일지입니다. BSSJ 개발 기록과 블로그 운영 기록을 탭으로 구분했습니다.</p>
    <div class="wl-stats"><span>총 항목 42개</span><span>BSSJ 37개</span><span>기타 5개</span><span>기간 2024-12-25 ~ 2026-03-12</span></div>
  </header>

  <section class="wl-project">
    <p class="wl-project-label">프로젝트</p>
    <div class="wl-project-tabs" role="tablist" aria-label="project tabs">
      <button class="active" type="button" role="tab" aria-selected="true" data-project="bssj">BSSJ</button>
      <button type="button" role="tab" aria-selected="false" data-project="misc">기타</button>
    </div>
  </section>

  <ol class="wl-timeline">
    <li class="wl-item" data-project="misc">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-03-12</p>
        <span class="wl-project-chip">GitHub Blog</span>
        <h3>GitHub Blog · 작업일지 탭/필터 적용</h3>
        <p>작업일지 페이지를 프로젝트 탭 구조로 개편했다. BSSJ와 기타 탭으로 전환할 수 있게 만들고, 탭 클릭 시 타임라인이 즉시 필터링되도록 적용했다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="misc">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-03-11</p>
        <span class="wl-project-chip">GitHub Blog</span>
        <h3>GitHub Blog · 작업일지 전용 페이지 오픈</h3>
        <p>일반 포스트 본문에서 분리해 /worklog 전용 페이지를 만들고, 타임라인 UI로 가독성을 개선했다. 날짜 순서와 카드 구조를 통일해서 흐름을 읽기 쉽게 맞췄다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="misc">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-03-11</p>
        <span class="wl-project-chip">GitHub Blog</span>
        <h3>GitHub Blog · BSSJ 아키텍처 글/다이어그램 작성</h3>
        <p>architecture 문서를 기준으로 아키텍처 글을 다시 작성하고, 전체 구성도와 모듈 의존도 SVG 다이어그램을 추가해 이해가 쉬운 형태로 정리했다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="misc">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-03-10</p>
        <span class="wl-project-chip">GitHub Blog</span>
        <h3>GitHub Blog · 라이트 테마 가독성 개선</h3>
        <p>다크 톤을 제거하고 눈이 편한 라이트 팔레트로 전체 스타일을 재정비했다. 배경/카드/코드 블록 대비를 낮춰 장문을 읽기 좋은 상태로 조정했다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="misc">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-03-10</p>
        <span class="wl-project-chip">GitHub Blog</span>
        <h3>GitHub Blog · GitHub Pages 초기 세팅</h3>
        <p>블로그 저장소를 정비하고 Pages 배포 경로를 점검했다. 포스트 날짜/경로 이슈를 조정해 실제 공개 URL에서 글이 노출되도록 안정화했다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-24</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0048 · AWS EC2 서버 세팅 및 애플리케이션 배포</h3>
        <p>EC2 단일 인스턴스 기준으로 Web/CMS/Backend를 운영 가능한 상태까지 배포했다. 실제 서비스 운영을 위한 인프라, 배포 스크립트, 서비스 재시작 흐름이 이 단계에서 완성됐다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-19</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0047 · GitHub Actions CI — PR 빌드/테스트 체크</h3>
        <p>PR 단계에서 자동 빌드/테스트가 돌도록 CI를 붙였다. 수동 확인 의존도를 줄이고 머지 품질을 시스템으로 보장하기 시작한 시점이다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-17</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0046 · Web Backend 패키지 구조 도메인별 전환</h3>
        <p>레이어 중심 패키지를 도메인 중심으로 재구성했다. 파일 탐색 경로를 줄이고 도메인 응집도를 높여, 이후 유지보수 속도를 올린 리팩터링이다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-13</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0044 · 성경 구절 검색 (개역개정 DB + CMS API)</h3>
        <p>성경 본문 데이터를 DB에 적재하고 검색 API를 제공해 CMS 작성 효율을 높였다. 콘텐츠 입력 과정에서 복사/붙여넣기 부담을 크게 줄인 작업이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-07</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0040 · Web 메인 페이지 리디자인</h3>
        <p>메인 화면의 인상과 정보 전달력을 동시에 개선했다. 단순 시각 변경이 아니라 첫 방문자가 핵심 콘텐츠로 빠르게 진입하도록 흐름을 다시 잡았다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-07</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0039 · CMS 이벤트 모달 관리 페이지</h3>
        <p>이벤트 모달을 CMS에서 생성/수정/삭제할 수 있게 관리 화면을 구현했다. 운영이 원하는 시점에 바로 공지/이벤트를 반영할 수 있게 된 작업이다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-05</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0038 · Review Fixes (Menu/Staff/FE Navigation)</h3>
        <p>리뷰에서 지적된 Staff 조회 쿼리, 메뉴 누락 조건, FE 하드코딩 문제를 정리했다. 기능 추가보다 신뢰도를 올리는 정합성 보완 작업이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-04</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0036 · 이벤트 모달 팝업 기능</h3>
        <p>메인 진입 시 노출되는 이벤트 모달 기능을 추가했다. 노출 기간/활성 상태 같은 운영 파라미터를 고려해 반복 사용 가능한 구조로 설계했다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-04</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0035 · 쇼츠 API 구현</h3>
        <p>쇼츠 목록 API를 별도로 구현해 설교와 쇼츠의 사용 맥락을 분리했다. 콘텐츠 타입에 맞는 조회 흐름을 따로 둔 게 포인트였다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-04</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0034 · 설교 API 구현</h3>
        <p>설교 목록 페이징 API를 구현해 콘텐츠가 많아져도 응답과 렌더링이 버티도록 했다. 운영에서 가장 자주 쓰이는 콘텐츠 영역 중 하나를 안정화한 작업이다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-03</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0033 · Frontend API Integration</h3>
        <p>프론트에서 API 설정(BASE_URL, CHURCH_ID 등)을 표준화하고 실제 연동을 본격 적용했다. 화면 더미 데이터 중심에서 실데이터 중심으로 전환한 단계다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-03</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0032 · Church ID UUID v7 Migration</h3>
        <p>교회 식별자 체계를 UUID v7로 정리해 정렬성과 확장성을 개선했다. 멀티 교회 대응을 고려한 데이터 식별 전략 정비 작업이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-03</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0031 · Web API 통합 테스트 추가</h3>
        <p>교회 정보 API를 중심으로 통합 테스트를 작성했다. 개발 속도만 보던 단계에서 회귀 안정성까지 챙기는 구간으로 넘어간 의미가 있다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-02</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0030 · Swagger API 문서화</h3>
        <p>OpenAPI/Swagger를 붙여 API 계약을 문서로 공개했다. 프론트-백 협업에서 "말로 맞추는" 비용을 줄인 실용적인 개선이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-02-01</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0029 · 섬기는 사람들 API</h3>
        <p>카테고리별 스태프 조회 API를 추가해 프론트와 데이터 연결을 완료했다. 공개 페이지가 CMS 데이터와 실제로 맞물리기 시작한 단계다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-29</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0025 · SubMenu content_table 필드 추가</h3>
        <p>서브메뉴가 단순 링크가 아니라 실제 콘텐츠 테이블과 연결되도록 확장했다. 메뉴 구조를 운영 데이터와 직접 연결하는 중요한 전환점이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-28</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0024 · Database Schema 설계 및 구현</h3>
        <p>본격 API 개발 전에 스키마를 먼저 확정했다. 데이터 관계를 선제적으로 정리하면서 기능 추가 시 발생할 수 있는 구조적 충돌을 줄였다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-25</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0023 · Sermons View Cleanup</h3>
        <p>설교 뷰의 불필요한 상태값과 복잡한 부분을 정리했다. 기능 추가가 아니라 유지보수성을 올리는 리팩터링 성격의 작업이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-24</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0022 · Web API Spring Profiles 분리</h3>
        <p>local / beta / release 프로필을 분리해서 환경별 설정 충돌을 줄였다. 배포 단계가 늘어날수록 필요한 기본 정리 작업을 초기에 반영한 케이스다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-23</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0021 · Backend API Bootstrap</h3>
        <p>Spring Boot 3.x + Java 21 기준으로 백엔드 기본 구조와 필수 엔드포인트를 세팅했다. 이후 API 작업의 출발점이 되는 표준 뼈대를 만든 작업이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-23</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0020 · 메인페이지 관리 기능</h3>
        <p>메인 페이지 노출 요소를 CMS 데이터로 제어하도록 전환했다. 배너/블록 같은 핵심 영역을 코드 수정 없이 운영에서 조정할 수 있게 된 점이 컸다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-20</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0019 · 섬기는 사람들 관리 기능 구현</h3>
        <p>교역자/리더십 정보를 CMS에서 직접 관리할 수 있게 CRUD 흐름을 구축했다. 소개 페이지를 운영 데이터 기반으로 전환한 핵심 작업이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-19</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0017 · CMS 메뉴 관리 기능 구현</h3>
        <p>하드코딩 메뉴를 CMS에서 관리 가능한 구조로 바꿨다. 운영자가 개발자 도움 없이 메뉴를 변경할 수 있게 되면서 유지보수 속도가 확 올라갔다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-17</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0014 · 쇼츠영상 페이지 구현</h3>
        <p>짧은 영상 콘텐츠를 빠르게 탐색할 수 있는 쇼츠 페이지를 만들었다. 검색/필터를 고려해 운영자가 계속 쌓아도 사용자 탐색 비용이 커지지 않게 설계했다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-13</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0011 · 목사님 칼럼 페이지</h3>
        <p>칼럼은 읽기 경험이 핵심이라 타이포그래피와 줄 간격, 문단 리듬에 집중했다. 콘텐츠 자체가 돋보이도록 장식보다 가독성을 우선한 페이지로 정리했다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-12</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0010 · 설교 페이지 개선</h3>
        <p>텍스트/이미지 배치가 어색했던 부분을 정리했다. 특히 흐름이 끊기던 레이아웃을 자연스럽게 바꿔 읽는 경험 자체를 개선했다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-12</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0009 · 모바일 메뉴 버그 수정</h3>
        <p>모바일에서 메뉴 오버레이가 열리지 않는 문제를 수정했다. 작은 버그였지만 실제 사용성에 바로 영향이 가는 구간이라 우선순위를 높게 가져간 작업이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-12</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0008 · 교회소개 페이지 통합 개편</h3>
        <p>흩어져 있던 교회소개 관련 메뉴를 사용자 관점으로 다시 묶었다. 정보 구조를 재정렬해서 방문자가 필요한 내용을 한 흐름 안에서 찾을 수 있게 개선했다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-11</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0007 · Design Migration (Figma AI -> Vue)</h3>
        <p>디자인 산출물을 실제 Vue 컴포넌트로 옮기는 마이그레이션 작업을 진행했다. 결과적으로 화면은 예쁘게 보이는 수준을 넘어, 이후 재사용 가능한 코드 구조로 바뀌었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-11</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0006 · Sermons Page (설교 페이지)</h3>
        <p>설교 콘텐츠를 카드/메타 중심으로 정리해서 탐색성이 좋게 구성했다. 단순 리스트가 아니라 제목-본문-설교자 같은 핵심 정보를 빠르게 읽도록 구조를 잡았다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2026-01-11</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0005 · 라이트 테마 + 콘텐츠 페이지 완성</h3>
        <p>전체 톤을 라이트 컨셉으로 맞추고 주요 콘텐츠 페이지의 시각 일관성을 맞췄다. 단순 색상 변경이 아니라 텍스트 가독성과 정보 밀도를 같이 조정한 작업이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2025-01-19</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0013 · 메뉴 한국어화 및 주보 게시판 구현</h3>
        <p>공개 사이트의 메뉴를 한국어 중심으로 정리하고 주보 게시판을 붙였다. 실제 교회 사용 환경에 맞춘 로컬라이징 성격이 강한 작업이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2025-01-10</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0004 · 메인 레이아웃 및 네비게이션</h3>
        <p>헤더/메뉴/레이아웃의 기본 틀을 확정했다. 이 시점에 네비게이션 흐름을 먼저 고정해둔 덕분에 이후 페이지 추가 작업이 훨씬 덜 흔들렸다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2025-01-09</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0003 · Splash Intro</h3>
        <p>첫 진입 경험을 위해 스플래시 인트로를 넣었다. 과한 기술 대신 SVG/CSS 애니메이션으로 구현해서 유지보수성을 챙기고, prefers-reduced-motion까지 고려해 접근성도 같이 챙긴 작업이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2025-01-09</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0002 · Web Frontend 프로젝트 초기화</h3>
        <p>Public Web 프론트엔드 뼈대를 Vite + Vue 3 + TypeScript로 세팅했다. 라우터와 Pinia까지 초기 연결해두면서 이후 화면 작업이 바로 들어갈 수 있는 기반을 마련했다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2025-01-09</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0001 · Bootstrap</h3>
        <p>프로젝트를 시작하면서 코드보다 먼저 일하는 방식을 정했다. agents / tasks / docs / apps 구조를 고정하고, Planning -> Design -> Implementation -> Review -> QA 흐름을 문서로 명확히 잡아 팀 기준점을 만든 작업이었다.</p>
      </article>
    </li>
    <li class="wl-item" data-project="bssj">
      <div class="wl-dot" aria-hidden="true"></div>
      <article class="wl-card">
        <p class="wl-date">2024-12-25</p>
        <span class="wl-project-chip">BSSJ</span>
        <h3>Task 0012 · 사진첩 갤러리</h3>
        <p>사진첩을 별도 섹션으로 분리해 운영 관점에서 확장 가능하게 설계했다. 카테고리/앨범 구조를 미리 잡아두면서 이후 CMS 연동 전환이 쉬워진 작업이었다.</p>
      </article>
    </li>
  </ol>
</section>

<script>
  (function () {
    const buttons = document.querySelectorAll('.wl-project-tabs button[data-project]');
    const items = document.querySelectorAll('.wl-item[data-project]');
    if (!buttons.length || !items.length) return;

    function applyFilter(project) {
      buttons.forEach((btn) => {
        const active = btn.dataset.project === project;
        btn.classList.toggle('active', active);
        btn.setAttribute('aria-selected', active ? 'true' : 'false');
      });

      items.forEach((item) => {
        item.hidden = item.dataset.project !== project;
      });
    }

    buttons.forEach((btn) => {
      btn.addEventListener('click', function () {
        applyFilter(btn.dataset.project);
      });
    });

    applyFilter('bssj');
  })();
</script>
