---
layout: default
title: Home
redirect_from:
  - /home/
relative_stylesheets:
  - home
---

<div class="showcase">
	{% for p in site.showcase %}
		<figure>
		{{ p.content }}
		</figure>
	{% endfor %}
</div>

# Home Page
Sit tight, I'm still working on it!
