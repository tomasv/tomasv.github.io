---
layout: post
title: "Closed Classes in Ruby"
date: 2012-10-05 00:05
published: true
---

Don't you just hate it when someone reopens your perfect class?
Well let's do something about that.

<!--more-->

So let's say we have a class:
{% highlight ruby %}
class Monkey
  def see
    'banana'
  end
end
{% endhighlight %}

And then we try to reopen it:
{% highlight ruby %}
class Monkey
  def see
    'banana'
  end
end

class Monkey
  def see
    'dirty patch'
  end
end
{% endhighlight %}

And now our class behaves differently.

{% highlight ruby %}
Monkey.new.see # => 'dirty_patch'
{% endhighlight %}

But what if we could do something like this:

{% highlight ruby %}
class Monkey
  extend NotMonkeyPatchable
end
{% endhighlight %}

And when we look at the same scenario again:
{% highlight ruby %}
class Monkey
  extend NotMonkeyPatchable

  def see
    'banana'
  end
end

class Monkey
  def see
    'dirty patches'
  end
end

Monkey.new.see # => 'banana'
{% endhighlight %}

Cool, our class resisted being reopened. But how?
The module we extend our class with looks like this:

{% highlight ruby %}
module NotMonkeyPatchable
  def method_added(name)
    @methods ||= {}
    if @methods[name]
      unpatch_method(name)
    else
      @methods[name] = instance_method(name)
    end
  end

  def unpatch_method(name)
    return if @unpatching
    @unpatching = true
    method = @methods[name] 
    define_method(name) { |*args, &block|
      method.bind(self).call(*args, &block)
    }
    @unpatching = false
  end
end

{% endhighlight %}

We utilize the `method_added` hook Ruby gives us.
For each method you define on a class, it's `method_added` method is called with the name of the method that was defined.

{% highlight ruby %}
def method_added(name)
  @methods ||= {}
  if @methods[name]
    unpatch_method(name)
  else
    @methods[name] = instance_method(name)
  end
end
{% endhighlight %}

We store the first implementation for a name. If it already exists then it means someone is trying to redefine the method.
And we don't want that, so we unpatch it back to what it was.

{% highlight ruby %}
def unpatch_method(name)
  return if @unpatching
  @unpatching = true
  method = @methods[name]
  define_method(name) { |*args, &block|
    method.bind(self).call(*args, &block)
  }
  @unpatching = false
end
{% endhighlight %}

The method we got through `instance_method` is unbound. In order to call it, it must first be bound to an object.
So that's what we do, we redefine the method again after it was patched, and inside we bind the original method object to self and call it.
Since we are defining a method the `method_added` hook will be invoked again, and to avoid dropping into an infinite loop we
guard ourselves with the `@unpatching` flag.

We probably need to add a mutex on this whole process of redefinition to be threadsafe, but I omitted it.

In fact, this even works as expected for subclasses.

{% highlight ruby %}
class Gorilla < Monkey
  def see
    'jungle'
  end
end

Gorilla.new.see # => 'jungle'

class Gorilla
  def see
    'ketchup'
  end
end

Gorilla.new.see # => 'jungle'
{% endhighlight %}

And there you have it, no more monkey patches are going to mess up your precious code. :)
