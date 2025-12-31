---
title: Tech
layout: default
permalink: /tech/
description: Notes and technical writing. Posts tagged 'tech'.
---

# Technical Notes
Posts with the `tech` tag will appear below.

<ul class="post-list">
  {% assign tech_posts = site.posts | where_exp: "p", "p.tags contains 'tech'" %}
  {% if tech_posts and tech_posts.size > 0 %}
    {% for post in tech_posts %}
      <li>
        <a class="title" href="{{ post.url | relative_url }}">{{ post.title }}</a>
        <div class="post-excerpt">{{ post.excerpt | strip_html | truncate: 180 }}</div>
        <div class="post-meta">
          <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d, %Y" }}</time>
        </div>
      </li>
    {% endfor %}
  {% else %}
    <li class="muted">No technical posts yet. Tag posts with <code>tech</code> to list them here.</li>
  {% endif %}
</ul>


