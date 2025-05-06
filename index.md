---
layout: home
title: "Diego Navarro's personal blog"
---

# Welcome to my personal blog

Here are my latest posts:

{% for post in site.posts %}
 - {{post.date | date: "%B %Y"}}
   {{post.title}}
   {{post.excerpt}}
   [Go to art]({{post.url}})
{% endfor %}