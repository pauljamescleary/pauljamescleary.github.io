---
layout: archive
permalink: /
title: "Home"
image:
  feature:
  teaser:
  thumb:
share: false
ads: false
---

![me]({{ site.url }}/images/120x120.gif)
Software Engineer.  Scala guy.

<div class="tiles">
{% for post in site.posts %}
	{% include post-grid.html %}
{% endfor %}
</div><!-- /.tiles -->
