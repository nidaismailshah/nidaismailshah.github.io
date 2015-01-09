---
layout: blogpage
title: Blog
permalink: /blog/
---
<div class="tech-posts">
	<ul class="posts">
	    {% for post in site.posts %}
	    	{% if post.tag == 1 %}
		      <li>
		        <!--<span class="post-date">{{ post.date | date: "%b %-d, %Y" }}</span>-->
		        <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
		         {{post.excerpt}}
		      </li>
		     {% endif %}
	    {% endfor %}
	 </ul>
 </div>
 <div class="poetry-posts">
	<ul class="posts">
	    {% for post in site.posts %}
	    	{% if post.tag == 2 %}
		      <li>
		        <!--<span class="post-date">{{ post.date | date: "%b %-d, %Y" }}</span>-->
		        <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
		         {{post.excerpt}}
		      </li>
		     {% endif %}
	    {% endfor %}
	 </ul>
 </div>