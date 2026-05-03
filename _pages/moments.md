---
layout: archive
permalink: /moments/
title: "Moments"
author_profile: true
---

{% include base_path %}
{% assign moments = site.moments | sort: "date" | reverse %}
{% capture written_year %}'None'{% endcapture %}
{% for moment in moments %}
{% capture year %}{{ moment.date | date: '%Y' }}{% endcapture %}
{% if year != written_year %}
<h2 id="{{ year | slugify }}" class="archive__subtitle">{{ year }}</h2>
{% capture written_year %}{{ year }}{% endcapture %}
{% endif %}
{% include archive-single-moment.html moment=moment %}
{% endfor %}
