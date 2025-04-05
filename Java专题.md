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
  <!-- 页面标题 -->
  <h1>{{ page.title }}</h1>

  <!-- 静态描述 -->
  <p>这里是关于 Java 的专题页面，涵盖相关的文章内容。</p>

  <!-- 动态显示文章列表 -->
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
