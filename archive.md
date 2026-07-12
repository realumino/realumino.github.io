---
layout: default
title: Archive
lang: en
ref: archive
---

<div class="posts-header">
  <h1><span data-i18n="nav_archive">Archive</span></h1>
</div>

{% assign posts_by_year = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}
{% for year in posts_by_year %}
<h2 style="font-size:1.25rem;font-weight:600;margin-bottom:16px;margin-top:32px;">{{ year.name }}</h2>
{% for post in year.items %}
<article class="post-item" data-lang="{{ post.lang | default: 'en' }}">
  <div class="post-meta">
    <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%b %d" }}</time>
    <span class="dot">·</span>
    <span>{{ post.content | number_of_words | divided_by: 200 | plus: 1 }} min read</span>
  </div>
  <h2 class="post-title"><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
</article>
{% endfor %}
{% endfor %}
