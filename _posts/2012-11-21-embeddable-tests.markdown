---
layout: post
title: "Embeddable tests"
date: 2012-11-21 20:07
---

Why not embed unit tests right into the code?

<!-- more -->

{% highlight ruby %}
module Testable
  def specs &block
    @specs = block
  end

  def validate(output: $stdout)
    require 'rspec'
    @specs.call
    RSpec::Core::Runner.run [], nil, output
  end

  def valid?
    code = validate(output: nil)
    code == 0
  end

  def run_specs!
    validate
  end
end

Object.extend Testable

class Foo
  specs do
    describe Foo do
      it 'should return bar 1' do
        subject.bar.should == 1
      end
    end
  end

  def bar
    1
  end
end

Foo.valid?
Foo.run_specs!

{% endhighlight %}

There, want to know whether a class passes its specs? Just run `Foo.valid?`.

This isn't very convenient if you think of the use cases for this, but the
idea itself is fun. :)
