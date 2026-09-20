---
title: "分类"
description: "按分类浏览文章"
permalink: /categories/
layout: page
---

{% assign categories = site.categories | sort %}
{% for category in categories %}
## {{ category[0] }}

{% for post in category[1] %}
- [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}

{% endfor %}
