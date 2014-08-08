---
layout: page
title: Repositories
---

Github repositories by [{{ site.noita.author.name }}][0]:

<ul>
{% for repo in site.github.public_repositories %}
<li><a href="{{ repo.html_url }}">{{ repo.name }}</a></li>
{% endfor %}
</ul>

[0]: https://github.com/mgruener
