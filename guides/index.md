---
layout: page
title: Guides / HOWTOs
---
Guides, HOWTOs or just small "look what I have done" snippets from our colleagues

<ul>
{% for p in site.pages %}
  {% if p.title contains 'guide.' %}
<li><a href="{{ p.url }}">{{ p.title | remove: 'guide.' }}</a></li>
  {% endif %}
{% endfor %}
</ul>
