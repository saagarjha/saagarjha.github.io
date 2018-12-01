---
layout: page
title: Blog
relative_stylesheets:
  - blog
---
<h1 id="title">Post Archives <a href="{{ site.baseurl }}/feed.xml">Atom Feed</a></h1>

<div id="jekyll-needs-this-for-some-reason">
	<section>
	{% for post in site.posts %}
		{% capture date %}{{ post.date | date: "%B %Y" }}{% endcapture %}
		{% if date != current_date %}
			{% if current_date %}
				</section>
				<section>
			{% endif %}
			{% assign current_date = date %}
			<h2><time datetime="{{ post.date | date: "%Y-%m" }}">{{ current_date }}</time></h2>
		{% endif %}
		<h3>
			<time datetime="{{ post.date | date: "%d" }}">{{ post.date | date: "%-d" }}</time>
			<a href="{{ post.url }}">{{ post.title }}</a>
		</h3>
	{% endfor %}
	</section>
</div>
