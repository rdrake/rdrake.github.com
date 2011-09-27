---
layout: default
title: Posts
---
<ul class="posts"
	{% for post in site.posts %}
		<li><a href="{{ post.url }}"></a> - <span class="quiet">{{ post.date | date_to_string }}</span></li>
	{% endfor %}
</ul>
