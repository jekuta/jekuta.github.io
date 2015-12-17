---
layout: post
title:  "Hints on how to debug Rails"
date:   2015-12-17 21:02:58 +0100
categories: ruby, rails
---
This a short summary of the techniques presented in a RailsConf talk by Godfrey Chan.

- Find documentation on <a href="http://api.rubyonrails.org">http://api.rubyonrails.org</a><br>
Find documentation for a specific version on <a href="http://api/rubyonrails.org/v4.1.10">http://api/rubyonrails.org/<b>v4.1.10</b><a/>

- In order to see the whole stack trace, uncomment the <code>Rails.backtrace_cleaner.remove_silencers!</code> line in the <code>config/initializers/backtrace_silencers.rb</code> file. Then raise an expection on line you want to debug.

- debug with <a href="https://github.com/deivid-rodriguez/byebug"><code>byebug</code></a>

- <code>pp caller</code> returns the current execution stack

- <code>pp instance_values</code> returns a hash with all the instance variables on the object

- Use the <a href="http://ruby-doc.org/core-2.2.3/Object.html#method-i-method"><code>Object#method</code></a> that returns a <a href="http://ruby-doc.org/core-2.2.0/Method.html"><code>Method</code></a> object<br>
{% highlight ruby %}
method(:method_name).source_location
method(:method_name).owner
{% endhighlight %}<br>
For even more options on what to call to the Method object, include
{% highlight ruby %}
gem "method source"
{% endhighlight %}

- make changes to the gem files with <a href="http://bundler.io/v1.11/bundle_open.html"><code>bundle open [GEM_NAME]</code></a><br>
and restore them to their original state with <a href="http://guides.rubygems.org/command-reference/#gem-pristine"><code>gem pristine [GEM_NAME]</code></a>
Gem files aren't reloaded by Rails though, so remember to restart the server between changes.

- Include rails from a locally forked version<br>
{% highlight ruby %}
gem "rails", path "~/myfork/rails/"
{% endhighlight %}<br>
This allows you, for example, to checkout from one version to another

- git the history of the file
{% highlight ruby %}
git log file
git blame file
git show a1b2c3d4 file # Show file at specific commit
git show a1b2c3d4^ file # Show file with the commit as a parent
{% endhighlight %}

- <a href="https://github.com/rails/rails/tree/master/guides/bug_report_templates">Bug templates</a>

The whole lecture<br>
<iframe width="560" height="315" src="https://www.youtube.com/embed/IjbYhE9mWuk" frameborder="0" allowfullscreen></iframe>
