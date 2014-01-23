---
layout: page
title: Lemon's Blog
tagline:  
---
{% include JB/setup %}

{% for post in site.posts %}
<div class="row">
<div class="col-md-12">
<h2><a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></h2>
<p><span class="glyphicon glyphicon-calendar"></span> <span class="label label-info">{{ post.date | date: "%Y年%m月%d日" }}</span>&nbsp;&nbsp;<span class="glyphicon glyphicon-tags"></span> <span class="label label-default">{{ post.tags }}</span>&nbsp;&nbsp;<span class="glyphicon glyphicon-folder-open"></span> <span class="label label-primary">{{ post.categories }}</span></p>
<p>{{ post.content | strip_html | truncate : 300 | prepend : "&nbsp;&nbsp;&nbsp;&nbsp;" }}</p>  

<p><a class="btn btn-default" href="{{ BASE_PATH }}{{ post.url }}" role="button">阅读全文 &raquo;</a></p>
</div>
</div>
{% endfor %}