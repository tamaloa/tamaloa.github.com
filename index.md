---
layout: page
title: Recent Posts
tagline: Supporting tagline
---
{% include JB/setup %}


<ul class="posts">
  {% for post in site.posts limit:3 %}
    <h3><a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></h3>
    <span>{{ post.date | date_to_string }}</span> &raquo;
    <p>{{ post.excerpt }}</p>
  {% endfor %}
</ul>

