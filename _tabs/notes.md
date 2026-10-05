---
# the default layout is 'page'
title: Notes
icon: fas fa-pen-to-square
order: 1
---

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url | relative_url }}) <span style="color: var(--text-muted-color); font-size: 0.85em;">{{ post.date | date: "%b %d, %Y" }}</span>
{% endfor %}
