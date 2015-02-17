---
layout: post
title: Joining the Disqusion
---

I have finally got around to adding commenting to this blog. In the end it turned out to be quite simple.

I chose to use Disqus as it was such an easy option to embed, and has support for many different blogging platforms. To embed it in Jekyll was as simple as copying the sample HTML from the Disqus website.

There are some subtleties to getting this all working properly with Jekyll though. If articles can appear on more than one page of the site you'll need to make sure you set up a unique ID for each article and you'll probably want to add a permalink to the article's page as well.

{% raw %}
Luckily with Jekyll this was quite simple. Each post already has a unique ID `{{ post.id }}` and you're permalink URL can be found by adding the URL of the site to the `{{ post.url }}`. Putting this all together you get something like this:
{% endraw %}

{% highlight html %}
{% raw %}
  <!-- DISQUS -->
  <script type="text/javascript">
    /* * * CONFIGURATION VARIABLES: EDIT BEFORE PASTING INTO YOUR WEBPAGE * * */
    var disqus_shortname = '{{ site.disqus_shortname }}'; // required: set your short name in config.yml
	var disqus_identifier = '{{ post.id }}';
	var disqus_title = '{{ post.title }}';
	var disqus_url = '{{ post.url | prepend: site.url }}';

    /* * * DON'T EDIT BELOW THIS LINE * * */
    (function() {
    var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
    dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
    (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    })();
  </script>
  <div id="disqus_thread"></div>
{% endraw %}
{% endhighlight %}

Feel free to use this to embed Disqus into your site.