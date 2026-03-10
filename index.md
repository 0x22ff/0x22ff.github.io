---
layout: default
title: Home
---

<section class="intro-hero reveal">
  <p class="hero-kicker">ENGINEERING JOURNAL</p>
  <h1>실무에서 부딪힌 문제를<br />정확하게 정리하는 개발 로그</h1>
  <p class="hero-lead">
    배포, 백엔드 설계, 운영 이슈, 트러블슈팅 기록을 구조적으로 남깁니다.
    빠르게 만들고, 정확하게 개선하는 과정을 공유합니다.
  </p>
  <div class="hero-cta-row">
    <a class="hero-cta primary" href="#latest-posts">최신 글 보기</a>
    <a class="hero-cta secondary" href="/about/">블로그 소개</a>
  </div>
</section>

<section class="stats-grid reveal">
  <article class="stat-card">
    <h2>Build</h2>
    <p>아키텍처 선택과 구현 전략</p>
  </article>
  <article class="stat-card">
    <h2>Ship</h2>
    <p>배포 파이프라인과 운영 기록</p>
  </article>
  <article class="stat-card">
    <h2>Fix</h2>
    <p>장애 대응과 회고 중심 트러블슈팅</p>
  </article>
</section>

<section id="latest-posts" class="section-title reveal">
  <h2>Latest Posts</h2>
  <p>최근에 정리한 글입니다.</p>
</section>

{% if site.posts.size > 0 %}
<ul class="post-grid">
  {% for post in site.posts limit: 9 %}
  <li class="post-card reveal">
    <a class="post-link-card" href="{{ post.url | relative_url }}">
      <p class="post-date">{{ post.date | date: "%Y.%m.%d" }}</p>
      <h3>{{ post.title | escape }}</h3>
      {% if post.excerpt %}
      <p class="post-excerpt">{{ post.excerpt | strip_html | truncate: 120 }}</p>
      {% endif %}
      <p class="post-more">Read article</p>
    </a>
  </li>
  {% endfor %}
</ul>
{% else %}
<div class="empty-state reveal">
  <p>아직 등록된 글이 없습니다. 첫 글을 작성해보세요.</p>
</div>
{% endif %}
