---
layout: default 
title: Ilya 'Elcamlost' Rassadin 
tagline: My blog about modern perl and devops 
description: My blog about modern perl and devops
---

I'm a perl programmer and devops engineer living in [Yaroslavl, Russia](https://en.wikipedia.org/wiki/Yaroslavl).
You can contact me through [telegram](https://t.me/elcamlost).

## Recent posts

{% for post in site.posts %}
### [{{ post.title }}]({{ post.url }})

{{ post.description }}
{% endfor %}
