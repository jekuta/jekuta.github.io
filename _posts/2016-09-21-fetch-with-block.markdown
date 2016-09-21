---
layout: post
title:  Fetch with block
date:   2016-09-21 6:00:00 +0100
categories: ruby
---

Provide default value for `#fetch` method with block instead of an argument.

{% highlight ruby %}
# expensive_computation is evaluated immediately
[].fetch(1, expensive_computation)
# expensive_computation is evaluated if the key is no found
[].fetch(1) { expensive_computation}
{% endhighlight %}
