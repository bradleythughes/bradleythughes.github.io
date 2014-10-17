---
layout: default
---

# Introduction
This is my little corner, something I've been wanting to make for a while but never got around to doing. I chose GitHub Pages because it allowed me to do everything in [Markdown](https://help.github.com/articles/github-flavored-markdown/), which appealed to the minimalist in me.

### Posts
{% for post in site.posts %}
{{ post.date | date: "%Y-%m-%d" }} - [{{ post.title }}]({{ post.url | prepend: site.baseurl }})
{% endfor %}

