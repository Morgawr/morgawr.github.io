---
layout: page
title: (def main :page) 
---

{% for post in site.posts limit:5 %}
  <div class="postprev">
    <div class="postprevh">
      <h2><a class="postprevlink" href="{{ post.url }}">{{ post.title }}</a></h2>
      <span class="light">&raquo; {{ post.date | date_to_string }}</span>
    </div>
    {% if post.content contains '<!--more-->' %}
      {{ post.content | split:'<!--more-->' | first }}
      <p class="readmore"><a href="{{ post.url }}">Read more...</a></p>
    {% else %}
      {{ post.content }}
    {% endif %}
  </div>
  <hr />
  <br />
{% endfor %}

{% if site.posts == empty %}
  <p>nothing to see here yet!</p>
{% endif %}
