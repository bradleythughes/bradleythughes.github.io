---
layout: default
---

{% for post in site.posts %}
{% if forloop.first %}
{% assign page = post %}
{% assign content = post.content %}
{% include post.html %}
{% endif %}
{% endfor %}
