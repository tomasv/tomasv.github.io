---
layout: post
title: "Manipulating the database in Rails console"
date: 2012-09-10 21:41
published: true
---

A neat thing I learned while hacking on legacy schema support is that you can
easily test various migration/schema methods in the Rails console.

Just type this in the Rails console:
{% highlight ruby %}
ActiveRecord::Schema.define { create_table :posts }
{% endhighlight %}
And the table will be created right there and then.
Abuse this at your own risk. ;)
