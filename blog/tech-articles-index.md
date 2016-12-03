---
layout: page
permalink: tech-articles/index/
---

# Technical Articles

<ul>
 {% for post in site.posts %}
   <li>
     <a href="{{ post.url }}">{{ post.title }}</a>
     {{ post.excerpt }}
   </li>
 {% endfor %}
</ul>