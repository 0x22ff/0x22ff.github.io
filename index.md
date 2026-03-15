---
layout: default
title: 글
---

<section class="v-tabs" aria-label="navigation tabs">
  <a class="active" href="/">글</a>
  <a href="/worklog/">작업일지</a>
</section>

{% if site.posts.size > 0 %}
<section class="v-feed">
  {% for post in site.posts %}
  <article class="v-card">
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
{% else %}
<div class="v-empty">아직 작성된 글이 없습니다. 첫 포스트를 만들어보세요.</div>
{% endif %}
