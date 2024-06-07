---
layout: default
title: Blog
---
<div class="posts">
    {% for post in site.posts  limit:5 %}
			<article>
				<h2><a href="{{ post.url }}">{% include page_title.html target=post %}</a></h2>
					
				<div class="postdate">
					{{ post.date | date: "%B %e, %Y"  }}					
				</div>
					
				<div>{{ post.excerpt }}</div>
				<div><a href="{{ post.url}}">Read the rest of "{% include page_title.html target=post %}"</a></div>
				{% if post.commentId %}
				<div id="commentsFor{{ post.commentId }}">
					<img src="/assets/img/ajax-loader.gif" /> Loading comments...
				</div>
				{% endif %}
			</article>
    {% endfor %}
</div>

<h3>OLDER</h3>
<ul class="postArchive">
{% for post in site.posts offset:5 %}
	<li>
		<span> {{ post.date | date: "%d %b %y"  }} </span> <a href="{{ post.url }}">{% include page_title.html target=post %}</a>
	</li>
{% endfor %}

<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js"></script>
<script type="text/javascript">
function updateCommentCount(commentId, url){
	$.ajax("https://api.github.com/repos/LodewijkSioen/LodewijkSioen.github.io/issues/"+ commentId +"/comments", {
	    headers: {Accept: "application/vnd.github.full+json"},
    	success: function(msg){
    		if(msg.length > 0){
       			$("#commentsFor" + commentId).html("<a href='"+url+"#comments'>"+msg.length+" comment(s) for this post</a>.");
       		} else {
       			$("#commentsFor" + commentId).html("No comments for this post.");
       		}
		},
		error: function(){
			$("#commentsFor" + commentId).html("Error loading comments");
		}
	});
}
{% for post in site.posts  limit:5 %}
{% if post.commentId %}updateCommentCount({{ post.commentId }}, "{{ post.url }}");{% endif %}
{% endfor %}
</script>