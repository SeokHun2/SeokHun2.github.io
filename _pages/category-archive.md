---
title: "카테고리"
layout: categories
permalink: /categories/
author_profile: true
sidebar_main: true
---

전체 글을 카테고리별로 모았습니다. QA · Network 글은 상단 메뉴에서 주제별로 볼 수 있습니다.

{% assign cats = site.categories | sort %}
<ul class="taxonomy__index">
  {% for c in cats %}
    <li>
      <a href="#{{ c[0] | slugify }}">
        <strong>{{ c[0] }}</strong> <span class="taxonomy__count">{{ c[1].size }}</span>
      </a>
    </li>
  {% endfor %}
</ul>
