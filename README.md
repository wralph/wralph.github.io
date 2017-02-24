---
title: Ralph on technology...
---

This is a blog about technical stuff I am dealing with every day. 
It hopefully helps others to get things done.

## Articles
<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.date | date_to_string  }} - {{ post.title }}</a>      
    </li>
  {% endfor %}
</ul>
