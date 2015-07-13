---
layout: post
title: "ActiveRecord and legacy schema"
date: 2012-08-27 23:47
published: true
---

Rails, and ActiveRecord in particular, is pretty possessive in how it handles your database.
But once in a while you need to play nice with strangers as well.
So let's look at a situation where we must develop on top of an existing schema with existing data in Rails.

<!--more-->

As usual we have our own tables we develop and create ourselves through migrations.
And in production there are already pre-existing tables with data that we must use in our code as well.

Once we do `db:schema:load`, all the tables mentioned in `db/schema.rb` will be dropped if they exist and then recreated.
So we can't really do a schema load in production if we have pre-existing tables in our `db/schema.rb`.
We could avoid mentioning pre-existing tables in migrations, but once you invoke `db:migrate` it dumps the current schema into `db/schema.rb`.
If we keep the tables we do not own from `db/schema.rb` it becomes tricky to set up the test database, since it does
`db:schema:load` for the test environment.

My initial approach was to check out what features `ActiveRecord::SchemaDumper` provides. 
And luckily there is an `ignored_tables` class attribute accessor defined on `SchemaDumper` which does exactly what it says: it ignores
specified tables while doing schema dumps.

So to keep pre-existing tables out of `db/schema.rb` we need to add them to `ignored_tables`.
For that I created an initializer called `schema_dumper.rb` containing:
{% highlight ruby %}
ActiveRecord::SchemaDumper.ignored_tables = %w(not_my_table1 not_my_table2)
{% endhighlight %}

Just to be sure we can do `rake db:schema:dump` and check whether `db/schema.rb` does not mention the ignored tables.
This means we can safely load our own schema in production to bootstrap things
(afterwards migrations are fine).

But our tests still can't run. The test environment has no knowledge of the pre-existing tables we might need.
I chose to make a separate schema file for them, and load it separately.
Let's call it `db/foreign_schema.rb`:
{% highlight ruby %}
ActiveRecord::Schema.define do
  create_table :not_my_table1, :force => true do |t|
    t.string "foo"
    t.string "bar"
  end

  create_table :not_my_table2, :force => true do |t|
    t.string "baz"
    t.string "qaz"
  end
end
{% endhighlight %}

Of course writing out schema definitions by hand is tedious, I exported schema definitions from production,
ran them locally, dumped the schema with `ignored_tables` temporarily empty,
then moved the definitions out of `db/schema.rb` into `db/foreign_schema.rb`.

To load this we need a new rake task. So let's do `rails g task load_foreign_schema` and define it like so:
{% highlight ruby %}
namespace :db do
  namespace :schema do
    task :load_foreign => :environment do
      load("#{Rails.root}/db/foreign_schema.rb")
    end
  end
end
{% endhighlight %}

Now we need to hook this up with `db:test:prepare`, so we add this to `Rakefile`:
{% highlight ruby %}
namespace :db do
  namespace :test do
    task :prepare => 'db:schema:load_foreign'
  end
end
{% endhighlight %}

Now once we run `rake spec` it will as usual invoke `db:schema:load`, which in turn will invoke `db:schema:load_foreign`. Thus
our test environment will have everything it needs.

For development we can run `db:schema:load_foreign` manually (not so elegant).
Tests are taken care of automatically by hooking into `db:test:prepare`.
`db:schema:load` can safely be run on production, and migrations used afterwards.
All the pre-existing tables are added to `db/foreign_schema.rb` and appended to `ignored_tables`.

By doing this we insure that `db:schema:load` can be run in production without fear of dropping tables that do not belongs to us
(assuming there are no table name collisions).
