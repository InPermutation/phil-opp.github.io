---
layout: page
title: Archive
redirect_from: "/archive/"
---

## Blog Posts

{% for post in site.posts reversed %}
  * {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
{% endfor %}
