---
layout: default 
title: simple site
tagline: Easy websites with GitHub Pages
description: Minimal tutorial on making a simple website with GitHub Pages
---

{% for post in site.posts %}
### [{{ post.title }}]({{ post.url }})
_lang: {{ post.language }}_

{{ post.description }}
{% endfor %}
