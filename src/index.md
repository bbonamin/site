---
# Feel free to add content and custom Front Matter to this file.

layout: default
---

# Landing Code

<ul>
  {% for post in collections.posts.resources %}
    <li>
      <a href="{{ post.relative_url }}">{{ post.data.title }}</a>
    </li>
  {% endfor %}
</ul>

---
