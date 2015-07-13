---
layout: post
title: "Movie night with Basil"
date: 2012-08-18 01:28
published: true
---

[Basil](https://github.com/pbrisbin/basil) is a Skype bot with a flexible plugin system.
I'm not going to go over the basics of basil, it's all there on its homepage.
Instead I'll try to demonstrate how easy it is to add custom plugins to it.

<!--more-->

I've had basil running in a chat room for some while now. And as time passed I added various
plugins to it that I thought might be fun: google translate, random memorized phrase replies,
url minifier, Torvalds quotes.

The latest fad was people asking for movie recommendations several evenings in a row.
So I thought, what the heck, this is a job for basil.

Luckily there's an [imdb](https://github.com/ariejan/imdb/) gem available.
The API is simple:

{% highlight ruby %}
require 'imdb'

Imdb::Top250.new.movies
{% endhighlight %}

Which gives as a list of the top 250 IMDb movies. Now we only need to hook it into basil.
Basil stores plugins in its plugins subdirectory by default. All of the ruby files in there
are loaded up and evaluated.

So all we have to do is add another one, so we just add imdb to our basil project `Gemfile` and
then create a plugin file in the plugins subdirectory.
Let's call it `imdb.rb`:

{% highlight ruby %}
require 'imdb'

Basil.respond_to(/what should I watch.*/i) {
  movie = Imdb::Top250.new.movies.sample
  title = "#{movie.title} (#{movie.url})"
  possible_replies = [
    "try #{title}, its cool",
    "how about #{title}?",
    "I heard #{title} is pretty good"
  ]
  replies possible_replies.sample
}
{% endhighlight %}
Basil plugins in general expect you to define such pattern guards to catch
user messages which are relevant to a particular plugin and react inside the block.

Here we use `Basil.respond_to`. It only reacts when someone addresses basil directly, as in:
> basil, what should i watch today?

There's also `Basil.watch_for`, it screens all the messages in a conversation, not just those directed to basil.

Inside the block we can use `say` or `replies` methods to make basil say something.
`replies` will prepend the senders name, and `says` simply sends the string passed to it.

And that's it, fire up basil, and enjoy insightful movie recommendations. :)

