---
layout: post
title: "Simple CoffeeScript Mixins"
date: 2012-11-01 16:40
---

I was hacking around with CoffeeScript and the need to share part of some functionality between
unrelated classes came up.

<!-- more -->

{% highlight coffeescript %}
class User
  move: (somewhere) ->
    somewhere.add(this)

class Post
  move: (somewhere) ->
    somewhere.add(this)
{% endhighlight %}

These two classes are completely unrelated except for the fact that they both are movable.
So here's a simple way to introduce a mixin keyword of sorts into CoffeeScript class scope.

{% highlight coffeescript %}
Function::mixin = (functions) ->
  for name, value of functions
    @::[name] = value

Movable =
  move: (somewhere) ->
    somewhere.add(this)

class User
  @mixin Movable

class Post
  @mixin Movable
{% endhighlight %}

`Movable` can contain both functions and objects, it will work the same way as if they were defined directly in the class.
This is a very straightforward and simple implementation, it provides no inheritance mechanism to mixins like Ruby does,
but for most cases this is enough.

