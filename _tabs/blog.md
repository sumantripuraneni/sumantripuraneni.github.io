---
layout: page
title: Blog
icon: fas fa-pen-nib
order: 1
---

{% for post in site.posts %}
<div class="mb-4">
  <h3 class="mb-1"><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
  <small class="text-muted">{{ post.date | date: "%b %d, %Y" }}</small>
  {% if post.description %}<p class="mt-1">{{ post.description }}</p>{% endif %}
</div>
{% endfor %}