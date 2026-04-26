---
layout: page
title: Notes
permalink: /notes/
---

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }}) <span style="color: #999; font-size: 0.85em;">{{ post.date | date: "%b %d, %Y" }}</span>
{% endfor %}