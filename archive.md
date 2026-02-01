---
layout: page
title: Blog Archive
---

{% for tag in site.tags %}
  <h3 class="archive-tag-heading">{{ tag[0] }}</h3>
  <ul class="archive-list">
    {% for post in tag[1] %}
      <li class="archive-item">
        <span class="archive-date">{{ post.date | date: "%B %Y" }}</span>
        <a href="{{ post.url }}">{{ post.title }}</a>
      </li>
    {% endfor %}
  </ul>
{% endfor %}
