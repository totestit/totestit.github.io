---
title: tags
layout: page
---

<div id='tag_cloud'>
{% for tag in site.tags %}
<b><a href="#{{ tag[0] }}" title="{{ tag[0] }}" rel="{{ tag[1].size }}">{{ tag[0] }}</a></b>, 
{% endfor %}
</div>

<ul class="listing">
{% for tag in site.tags %}
  <b><li class="listing-seperator" id="{{ tag[0] }}">{{ tag[0] }}</li></b>
{% for post in tag[1] %}
  <li class="listing-item">
  <time datetime="{{ post.date | date:"%Y-%m-%d" }}">{{ post.date | date:"%Y-%m-%d" }}</time>
  <a href="{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a>
  </li>
{% endfor %}
{% endfor %}
</ul>

<script src="/media/js/jquery-1.7.1.min.js" type="text/javascript" charset="utf-8"></script> 
<script src="/media/js/jquery.tagcloud.js" type="text/javascript" charset="utf-8"></script> 
<script language="javascript">
$.fn.tagcloud.defaults = {
    size: {start: 1, end: 1, unit: 'em'},
      color: #ff3333
};

$(function () {
    $('#tag_cloud a').tagcloud();
});
</script>
