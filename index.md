---
layout: default
title: Home
---

<section class="hero-shell fade-in">
  <p class="micro-label">0x22ff Journal</p>
  <h1>
    Build fast.<br />
    Fix deep.<br />
    Document clearly.
  </h1>
  <p class="hero-summary">
    백엔드 설계, 배포 자동화, 운영 이슈 대응 과정을 실무 중심으로 기록합니다.
  </p>
  <div class="hero-actions">
    <a class="btn-solid" href="#posts">최근 글 보기</a>
    <a class="btn-ghost" href="/about/">소개 보기</a>
  </div>
</section>

<section class="band-grid fade-in">
  <article>
    <h2>Architecture</h2>
    <p>구조 선택의 이유와 트레이드오프를 남깁니다.</p>
  </article>
  <article>
    <h2>Delivery</h2>
    <p>CI/CD, 배포 흐름, 운영 환경 개선 내역을 정리합니다.</p>
  </article>
  <article>
    <h2>Reliability</h2>
    <p>장애 원인과 재발 방지 체크리스트를 공유합니다.</p>
  </article>
</section>

<section id="posts" class="section-head fade-in">
  <h2>Recent Posts</h2>
  <p>가장 최근에 정리한 기록입니다.</p>
</section>

{% if site.posts.size > 0 %}
<ul class="post-stack">
  {% for post in site.posts limit: 10 %}
  <li class="post-row fade-in">
    <a href="{{ post.url | relative_url }}">
      <span class="post-row-date">{{ post.date | date: "%Y.%m.%d" }}</span>
      <strong class="post-row-title">{{ post.title | escape }}</strong>
      <span class="post-row-arrow">→</span>
    </a>
  </li>
  {% endfor %}
</ul>
{% else %}
<div class="empty-box fade-in">
  아직 등록된 글이 없습니다. 첫 글을 작성해보세요.
</div>
{% endif %}
