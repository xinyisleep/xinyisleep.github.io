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

  <!-- 判断 category -->
{% if page.category == "Java" %}
<p>这篇文章属于 Java 分类。</p>
{% endif %}

  <!-- 判断 tags -->
{% if page.tags contains 'Java' %}
<p>这篇文章有 Java 标签。</p>
{% endif %}

  <!-- 判断 permalink -->
{% if page.permalink == '/Java专题/' %}
<p>这是 Java专题的永久链接。</p>
{% endif %}

  <!-- 判断是否用默认布局 -->
{% if page.layout == 'default' %}
<p>使用的是默认布局。</p>
{% endif %}
</div>
