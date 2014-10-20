---
layout: default
---

{% for post in site.posts %}
{% if forloop.first %}
{{ post.content }}
## All Posts
{% endif %}
{{ post.date | date: "%Y-%m-%d" }} - [{{ post.title }}]({{ post.url | prepend: site.baseurl }})
{% endfor %}
