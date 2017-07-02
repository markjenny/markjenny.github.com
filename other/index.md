---
layout: archive
title: "Latest Posts in Other"
image:
 feature: cover.jpg
 credit: Gratisography, Free high-resolution photos
 creditlink: http://gratisography.com
---

<div class="tiles">
{% for post in site.posts %}
   {% if post.title == '小站的第一篇博客' %}
		{% include post-grid.html %}
   {% endif %}
{% endfor %}
</div><!-- /.tiles -->
