---
layout: default
title: Java专题
permalink: /Java专题/
category: Java
tags:
  - programming
  - Java
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
