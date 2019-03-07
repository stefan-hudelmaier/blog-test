---
layout: default
---

[Device Insight](https://device-insight.com) has been founded in 2003 and has realized
more than 150 IoT projects across all industries. Our projects and solutions are all
based on our CENTERSIGHT cloud platform.

{% for post in site.posts %}
  <h1><a href="{{ post.url }}">{{ post.title }}</a></h1>
  <p>{{ post.date | date_to_string }} - {{ post.author }}</p>
  <p>{{ post.content }}</p>
{% endfor %}

