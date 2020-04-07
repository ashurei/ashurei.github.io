---
layout: page
title: Documentation
permalink: /docs/
location: home
comments: false
---

# Documentation

Archive of Documentation.

<div class="section-index">
	{% for post in site.docs  %}        
	{% if post.location == 'content' %}
	<div class="entry">
	<h5><a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a></h5>
	<p>{{ post.description }}</p>
	</div>{% endif %}{% endfor %}
</div>
