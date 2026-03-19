---
layout: default
title: "0x22ff Journal | Java Backend Engineering Notes"
description: "9년차 자바 백엔드 개발자의 운영 기록. Spring Boot, WebFlux, SRE, Resilience4j, 모니터링/장애 대응 실전 적용기를 다룹니다."
---

<section class="v-tabs" aria-label="navigation tabs">
  <a class="active" href="/">글</a>
  <a href="/worklog/">작업일지</a>
</section>

<section class="v-search" aria-label="글 검색">
  <label class="search-label" for="post-search-input">글 검색</label>
  <div class="search-row">
    <input id="post-search-input" class="search-input" type="search" placeholder="제목, 요약, 태그로 검색" autocomplete="off">
    <button class="search-clear" type="button" data-post-search-clear hidden>지우기</button>
  </div>
</section>

{% if site.posts.size > 0 %}
<section class="v-feed">
  {% for post in site.posts %}
  <article class="v-card" data-post-item>
    <a class="v-card-link" href="{{ post.url | relative_url }}">
      {% if post.thumbnail %}
      <img class="v-thumb" src="{{ post.thumbnail | relative_url }}" alt="{{ post.title | escape }} thumbnail" loading="lazy">
      {% else %}
      <div class="v-thumb v-thumb-placeholder" aria-hidden="true"></div>
      {% endif %}
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

      </div>
    </a>
  </article>
  {% endfor %}
</section>
<p class="v-search-empty" data-post-empty hidden>검색 결과가 없습니다.</p>
{% else %}
<div class="v-empty">아직 작성된 글이 없습니다. 첫 포스트를 만들어보세요.</div>
{% endif %}

<script>
  (function () {
    const input = document.getElementById('post-search-input');
    const clear = document.querySelector('[data-post-search-clear]');
    const items = document.querySelectorAll('[data-post-item]');
    const empty = document.querySelector('[data-post-empty]');
    if (!input || !items.length) return;

    function applyFilter() {
      const query = input.value.trim().toLowerCase();
      let visibleCount = 0;

      items.forEach((item) => {
        const text = item.textContent.toLowerCase();
        const matches = !query || text.includes(query);
        item.hidden = !matches;
        if (matches) visibleCount += 1;
      });

      if (clear) clear.hidden = query.length === 0;
      if (empty) empty.hidden = visibleCount !== 0;
    }

    input.addEventListener('input', applyFilter);
    if (clear) {
      clear.addEventListener('click', function () {
        input.value = '';
        applyFilter();
        input.focus();
      });
    }

    applyFilter();
  })();
</script>
