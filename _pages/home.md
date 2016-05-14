---
layout: splash
permalink: /
date:
header:
  overlay_color: "#5e616c"
  overlay_image: mm-home-page-feature.jpg
  cta_label: "<i class='fa fa-download'></i> Install Now"
  cta_url: "/docs/quick-start-guide/"
  caption:
excerpt: 'A flexible two-column Jekyll theme. Perfect for personal sites, blogs, and portfolios hosted on GitHub or your own server.<br /> <small>Currently at version 3.1.6</small><br /><br /> {::nomarkdown}<iframe style="display: inline-block;" src="https://ghbtns.com/github-btn.html?user=mmistakes&repo=minimal-mistakes&type=star&count=true&size=large" frameborder="0" scrolling="0" width="160px" height="30px"></iframe> <iframe style="display: inline-block;" src="https://ghbtns.com/github-btn.html?user=mmistakes&repo=minimal-mistakes&type=fork&count=true&size=large" frameborder="0" scrolling="0" width="158px" height="30px"></iframe>{:/nomarkdown}'
feature_row:
  - image_path: mm-customizable-feature.png
    alt: "customizable"
    title: "Super Customizable"
    excerpt: "Everything from the menus, sidebars, comments, and more can be configured or set with YAML Front Matter."
    url: "/docs/configuration/"
    btn_label: "Learn More"
  - image_path: mm-responsive-feature.png
    alt: "fully responsive"
    title: "Responsive Layouts"
    excerpt: "Built on HTML5 + CSS3. All layouts are fully responsive with helpers to augment your content."
    url: "/docs/layouts/"
    btn_label: "Learn More"
  - image_path: mm-free-feature.png
    alt: "100% free"
    title: "100% Free"
    excerpt: "Free to use however you want under the MIT License."
    url: "/docs/license/"
    btn_label: "Learn More"
github:
  - excerpt: '{::nomarkdown}<iframe style="display: inline-block;" src="https://ghbtns.com/github-btn.html?user=mmistakes&repo=minimal-mistakes&type=star&count=true&size=large" frameborder="0" scrolling="0" width="160px" height="30px"></iframe> <iframe style="display: inline-block;" src="https://ghbtns.com/github-btn.html?user=mmistakes&repo=minimal-mistakes&type=fork&count=true&size=large" frameborder="0" scrolling="0" width="158px" height="30px"></iframe>{:/nomarkdown}'
intro:
  - excerpt: 'Get notified when I add new stuff &nbsp; [<i class="fa fa-twitter"></i> @mmistakes](https://twitter.com/mmistakes){: .btn .btn--twitter}'
---

{% include feature_row id="intro" type="center" %}

{% include feature_row %}

<div id="index">
  <h3><a href="{{ site.url}}/posts/">Recent Posts</a></h3>
  {% for post in site.posts limit:5 %}
  <article>
    {% if post.link %}
      <h2 class="link-post"><a href="{{ site.url }}{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a> <a href="{{ post.link }}" target="_blank" title="{{ post.title }}"><i class="fa fa-link"></i></h2>
    {% else %}
      <h2><a href="{{ site.url }}{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a></h2>
      <p>{{ post.excerpt | strip_html | truncate: 160 }}</p>
    {% endif %}
  </article>
  {% endfor %}
</div><!-- /#index -->
