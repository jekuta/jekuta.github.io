---
layout: post
title:  "Requiring and loading files in Ruby"
date:   2015-12-12 14:00:00 +0100
categories: ruby
---

# Kernel#require

`require` allows you to load files, and make sure they only get loaded once.

As [ruby-doc.org explains](http://ruby-doc.org/core-2.3.1/Kernel.html#method-i-require):

> Loads the given name, returning true if successful and false if the feature is already loaded.
>
> If the filename does not resolve to an absolute path, it will be searched for in the directories
listed in `$LOAD_PATH` (`$:`).
>
> If the filename has the extension “.rb”, it is loaded as a source file; if the extension is “.so”,
“.o”, or “.dll”, or the default shared library extension on the current platform, Ruby loads the
shared library as a Ruby extension. Otherwise, Ruby tries adding “.rb”, “.so”, and so on to the name
until found. If the file named cannot be found, a LoadError will be raised.
>
> For Ruby extensions the filename given may use any shared library extension. For example, on Linux
the socket extension is “socket.so” and require 'socket.dll' will load the socket extension.
>
>The absolute path of the loaded file is added to `$LOADED_FEATURES` (`$"`). A file will not be loaded
again if its path already appears in $". For example, `require 'a'; require './a'` will not load a.rb
again.

First ruby checks whether the given file name resolves to an absolute path. If it does the file is required
 immediately. If it doesn’t ruby will iterate throught the directories in the `$LOAD_PATH` looking for
 a matching file name. Once the right path has been found, ruby looks into the `$LOADED_FEATURES`
 variable to see whether the file has already been required. If the path is found in $LOADED_FEATURES,
 the file has already been requird and false is returned. Otherwise, ruby will load the file and add
 the absolute path to the $LOADED_FEATURES.

In short, when using require, you need to make sure that

1. either the directory where your file is placed has been included in the `$LOAD_PATH`
{% highlight ruby %}
# file in ./my_dir/my_file.rb
$LOAD_PATH.unshift('my_dir')
require 'my_file'
{% endhighlight %}

2. or you specify an path that resolves to an absolute path
{% highlight ruby %}
# file in ./my_dir/my_file.rb
require './my_dir/my_file'
{% endhighlight %}

# Kernel#require_relative

If you specify the file name for the require method referencing the current directory, as in the previous
 example, you couple the code with place from which it gets invoked from.

Sometimes you might want the require a file referencing it from the location of the file that has the
 require. In older code, where ruby 1.8 was still used, one can find a lines similar to this:

{% highlight ruby %}
# This file in /my_dir/this.rb
# File to be required in /my_dir/my_file.rb
require File.expand_path('my_file', __FILE__)
# '/home/user/my_dir/my_file'
# or
require File.expand_path(File.dirname(__FILE__)) + '/my_file'
# '/home/user/my_dir/my_file'
{% endhighlight %}

Since 1.9.2 ruby includes a `require_relative` method, which does exactly the same as the code above.
 It creates an absolute path name, and uses the same c implementation as require method.

Obviously there is a conversation, on which of the approaches is better. `require` might in some cases
 be slightly slower than require_relative. This is because, since 1.9 ruby has built in the functionality
 of RubyGems that, overwrites the Kernel#require with its own version, that will expand the search of
 the filename to the loaded gems libraries as well.

# Kernel#load

`load` can also take an absolute or relative path. Similarly to require if for relative paths, the $LOAD_PATH
 will be searched. Load will always load the code, and it doesn’t remember which of the files it has
 already loaded.

{% highlight ruby %}
# my_file.rb
class A
  B = 'b'
end

# --irb
load('./my_file.rb')
> true
load('./my_file.rb')
my_file.rb:2: warning: already initialized constant A::B
my_file.rb:2: warning: previous definition of B was here
> true
{% endhighlight %}

What if the loaded code collides with the existing loading programs namespace? You can send a second
 parameter to the load method, that will “wrap” the code in an anonymous module.

{% highlight ruby %}
# my_file.rb
class A
  B = 'b'
end

# --irb
class A
  B = 'c'
end
load('./my_file.rb', true)
> true
A::B
> 'c'
{% endhighlight %}

# Kernel#autoload and Module#autoload
These methods allow lazy loading code as needed. They allow you too register a module and a filename,
 that will be required once called.

{% highlight ruby %}
autoload(:A, 'a')
A.some_method #=> requires the 'a' path
{% endhighlight %}

## Additional reading
[https://www.sitepoint.com/loading-code-ruby](https://www.sitepoint.com/loading-code-ruby/)

[https://www.practicingruby.com/articles/ways-to-load-code](https://www.practicingruby.com/articles/ways-to-load-code)
