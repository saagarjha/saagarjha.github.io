---
layout: page
title: Blog
relative_stylesheets:
  - blog
---

<header>
	<h1 class="title">
		<span>Post Archives</span>
		<span><a href="{{ site.baseurl }}/feed.xml">Atom Feed</a></span>
	</h1>
	<p class="subtitle">Semi-coherent rambling on topics I find interesting.</p>
</header>


<div id="karmdown-sucks">
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
		<h3 class="post">
			<time datetime="{{ post.date | date: "%Y-%m-%d" }}">{{ post.date | date: "%-d" }}</time>
			<a href="{{ post.url }}">{{ post.title }}</a>
		</h3>
	{% endfor %}
	</section>
</div>
