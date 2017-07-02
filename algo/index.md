---
layout: archive
title: "Latest Posts in Algo"
image:
 feature: cover.jpg
 credit: Gratisography, Free high-resolution photos
 creditlink: http://gratisography.com
---

<div class="tiles">
{% for post in site.categories.algo %}
	{% include post-grid.html %}
{% endfor %}
</div><!-- /.tiles -->
