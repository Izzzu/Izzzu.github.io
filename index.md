---
layout: page
title: Projects
---

{% for project in site.projects %}
### [{{ project.title }}]({{ project.url }})

{{ project.description }}

{% if project.tags %}**{{ project.tags | join: " · " }}**{% endif %}
{% if project.github %} · [GitHub]({{ project.github }}){% endif %}
{% if project.demo %} · [Demo]({{ project.demo }}){% endif %}

---
{% endfor %}
