---
layout: default
title: 글
---

<section class="v-profile">
  <div class="v-avatar">0x</div>
  <div>
    <h1>0x22ff</h1>
    <p>백엔드 개발, 배포 자동화, 운영 트러블슈팅 기록</p>
  </div>
</section>

<section class="v-tabs" aria-label="navigation tabs">
  <a class="active" href="/">글</a>
  <a href="/worklog/">작업일지</a>
</section>

{% if site.posts.size > 0 %}
<section class="v-feed">
  {% for post in site.posts %}
  <article class="v-card">
    <a class="v-card-link" href="{{ post.url | relative_url }}">
      <div class="v-thumb" aria-hidden="true"></div>
      <div class="v-body">
        <h2>{{ post.title | escape }}</h2>
        <p class="v-excerpt">{{ post.excerpt | strip_html | truncate: 140 }}</p>

        {% if post.tags and post.tags.size > 0 %}
        <ul class="v-tags">
          {% for tag in post.tags limit: 4 %}
          <li>#{{ tag }}</li>
          {% endfor %}
        </ul>
        {% endif %}

        <div class="v-meta">
          <span>{{ post.date | date: "%Y년 %-m월 %-d일" }}</span>
          <span>·</span>
          <span>1 min read</span>
        </div>
      </div>
    </a>
  </article>
  {% endfor %}
</section>
{% else %}
<div class="v-empty">아직 작성된 글이 없습니다. 첫 포스트를 만들어보세요.</div>
{% endif %}
