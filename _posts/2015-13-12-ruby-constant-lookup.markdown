---
layout: post
title:  "Ruby constant lookup"
date:   2015-16-12 21:02:58 +0100
categories: ruby
---

# Ruby constant lookup

When you reference a constant in ruby code, how is the right definition found?

First ruby checks the lexical scope of where the constant was referenced. That is the enclosing class or module. If no match is found ruby will continue to look for the reference in the next enclosing modules and classes. <code>Module.nesting</code> can be used to investigate which modules and classes are taken into account.
{% highlight ruby %}
module A
  class B
     Module.nesting #=> [A::B, A]
  end
end
{% endhighlight %}
{% highlight ruby %}
module A; end
class A::B
  Module.nesting #=> [A::B]
  # A is not part of the lexical scope here
end
{% endhighlight %}
If the definition is not found in the lexically enclosing scope, ruby will look into the inheritance hierarchy of the module or class that referenced the constant.
{% highlight ruby %}
module A
  Z = 'zorro'
end
Object.const_defined? 'A::Z' #=> true
class A::B
  ancestors #=> [A::B, Object, Kernel, BasicObject]
  Z #=> Will result in NameError: uninitialized constant A::B::Z
  A::Z #=> 'zorro' found on Object
end
{% endhighlight %}
Constants that are defined on Object class are so called "top-level" constants. It is worthwile noting that since ruby class inherit from Object, they eventually have access to the constants defined by it.
{% highlight ruby %}
Z = 'zebra'
module A
  Z = 'zorro'
end
Object.const_defined? 'A::Z' #=> true
class A::B
  ancestors #=> [A::B, Object, Kernel, BasicObject]
  Z #=> 'zebra', since Z is a top-level constant
end
# Modules do not inherit from Object
module C
  ancestors #=> [C]
  Z #=> NameError: uninitialized constant C::Z
end
{% endhighlight %}
If the definition still hasn't been found ruby will envoke the <code>const_missing</code> method of the class or module referencing to the constant.
