---
layout: default
title: PHP专题
permalink: /PHP专题/
category: PHP
tags:
  - PHP
---

<div class="content">
  <h1>{{ page.title }}</h1>
<h2>相关文章：</h2>
  <ul>
    {% for post in site.posts %}
      {% if post.category == page.category %}
        <li>
          <a href="{{ post.url }}">{{ post.title }}</a> - <small>{{ post.date | date: "%Y-%m-%d" }}</small>
        </li>
      {% endif %}
    {% endfor %}
  </ul>
</div>
