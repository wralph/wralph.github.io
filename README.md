This is a blog about technical stuff I am dealing with every day. 
It hopefully helps others to get things done.

Cheers


<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>
