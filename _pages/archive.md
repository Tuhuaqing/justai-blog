---
title: "文章归档"
description: "按年份归档的全部文章"
permalink: /archive/
layout: page
---

{% assign posts_by_year = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}
{% for year in posts_by_year %}
## {{ year.name }}

{% for post in year.items %}
- {{ post.date | date: "%m-%d" }} · [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}

{% endfor %}
