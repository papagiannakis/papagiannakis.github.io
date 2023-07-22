---
layout: archive
title: "Publications"
permalink: /publications/
author_profile: true
---

## Latest published research work
For a complete and most updated list of my scientific publications please visit:
* [Google Scholar](https://scholar.google.com/citations?user=rUfyI3MAAAAJ&hl=en)
* [Arxiv](https://arxiv.org/search/?query=George%20Papagiannakis&searchtype=all&source=header)
* [DBLP](https://dblp.uni-trier.de/pers/hd/p/Papagiannakis:George)
* [OrcID](https://orcid.org/0000-0002-2977-9850)
* [ORamaVR publications](https://oramavr.com/publications/)


{% if author.googlescholar %}
  You can also find my articles on <u><a href="{{author.googlescholar}}">my Google Scholar profile</a>.</u>
{% endif %}

{% include base_path %}

{% for post in site.publications reversed %}
  {% include archive-single.html %}
{% endfor %}
