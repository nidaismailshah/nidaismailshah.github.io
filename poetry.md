---
layout: blogpage
title: Poems
permalink: /poetry/
---
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