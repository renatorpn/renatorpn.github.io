---
title:  "manpages"
---
I call my digital garden the ``manpages``, because linux and stuff. I stash my notes and ideas here, feel free to wander around.

It seems that this space is yet under construction. C

{% for category in site.categories %}
  <h2>{{ category[0] }}</h2>
  <ul>
    {% for post in category[1] %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}

[teste](linux/test123)