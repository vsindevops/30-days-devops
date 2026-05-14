---
title: "All Articles"
permalink: /articles/
layout: archive
author_profile: true
---

{% assign posts_by_category = site.posts | group_by_exp: "post", "post.categories | first" %}

{% for category in posts_by_category %}
## {{ category.name }}

{% for post in category.items %}
- [{{ post.title }}]({{ post.url | relative_url }}) — <small>{{ post.date | date: "%B %d, %Y" }}</small>
  {% if post.excerpt %}<br><small>{{ post.excerpt | strip_html | truncatewords: 20 }}</small>{% endif %}
{% endfor %}

---
{% endfor %}
