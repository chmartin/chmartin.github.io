---
layout: page
title: Blog
active: blog
---

<div>
<ul>
    {% for post in site.posts %}
      <li><span>{{ post.date | date: "%Y-%m-%d" }} &raquo; </span><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
</ul>
</div>

<h3>Coming soon</h3>
<ul>
  <li>Series: AI development &amp; baseball</li>
</ul>
