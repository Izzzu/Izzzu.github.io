---
# the default layout is 'page'
title: Projects
icon: fas fa-diagram-project
order: 2
---

{% for project in site.projects %}
### [{{ project.title }}]({{ project.github | default: project.url }})

{{ project.description }}

{% if project.tags %}<span class="post-tags">{% for tag in project.tags %}<span class="post-tag">{{ tag }}</span>{% endfor %}</span>{% endif %}
{% if project.demo %}

[Demo]({{ project.demo }}){% endif %}

---
{% endfor %}
