---
layout: page
title: Blog
relative_stylesheets:
  - blog
---
<h1 id="title">Post Archives <a href="{{ site.baseurl }}/feed.xml">Atom Feed</a></h1>

<dl>
{% for post in site.posts %}
	{% capture date %}{{ post.date | date: "%B %Y" }}{% endcapture %}
	{% if date != current_date %}
		{% assign current_date = date %}
		<dt>{{ current_date }}</dt>
	{% endif %}
	<dd>
		<p>{{ post.date | date: "%-d" }}</p>
		<a href="{{ post.url }}">{{ post.title }}</a>
	</dd>
{% endfor %}
</dl>
