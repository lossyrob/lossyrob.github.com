---
layout: page
title: Pretending Shapes
tagline: Lossy compression of a decendant to pirates.
---
{% include JB/setup %}

We do exist to seek out all the things that make the dream stay.

Posts:

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>


