---
layout: default
title: linux
location: home
comments: false
---

# Linux

<div class="section-index">
<!--<hr class="panel-line" />-->
{% for document in site.docs  %}
	{% if document.url contains page.title and document.location == 'content' %}
	<div class="entry">
	<h5><a href="{{ document.url | prepend: site.baseurl }}">{{ document.title }}</a></h5>
	<p>{{ document.description }}</p>
	</div>
	{% endif %}
{% endfor %}
</div>

