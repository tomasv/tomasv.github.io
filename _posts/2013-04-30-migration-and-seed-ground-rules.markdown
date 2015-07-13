---
layout: post
title: "Migration and Seed Ground Rules"
date: 2013-04-30 21:08
---

I found the proper seeding and migration practices somewhat scattered all over the web,
so here's a writeup of what I think are the best practices for migrations and
seeding in Rails (3.2 as of writing this).

<!-- more -->

Migrations should contain:

* schema DSL
* raw SQL
* properly isolated ActiveRecord classes

Migrations must not contain:

* any dependence on code that will change in the future
* seed data

Application code grows and changes as your app evolves, migrations should be timeless and always valid.
Using your domain code in migrations makes them brittle and prone to becoming invalid.

`seeds.rb` should contain:

* data creation statements using application code

How `seeds.rb` should behave:

* it should be idempotent (it can be run many times and it will ensure everything is present in the database without breaking)
* it must not destroy existing data
* it must break loudly if some of the data failed to be created

How does a good migration look?

{% highlight ruby %}
class AddStuffProperly < ActiveRecord::Migration
  class PearForMigration < ActiveRecord::Base
    self.table_name = 'pears'
  end

  def up
    # schema DSL
    create_table :apples do |t|
      t.string :color
      t.references :owner
      t.timestamps
    end

    rename_column :pears, :size, :height
    
    # raw SQL
    execute <<-SQL
      UPDATE pears SET height = height * 2;
    SQL

    # or

    # model defined inside migration, never used outside
    PearForMigration.find_each do |pear|
      pear.height = pear.height * 2
      pear.save(validate: false) # the validate: false part is optional here, might be faster
    end
  end

  def down
    # ... opposite actions
  end
end
{% endhighlight %}

Having an isolated ActiveRecord model inside the migration is convenient if the migration is not trivial and we can benefit
from having a true ActiveRecord model at hand.

How does a good `seeds.rb` look?

{% highlight ruby %}

Gardener.find_or_create_by_email!(email: 'jack@example.com',
                                  first_name: 'Jack',
                                  last_name: 'Flowers')

Apple.find_or_create_by_color!(color: 'red')

Pear.find_or_create_by_size!(height: 5)
Pear.find_or_create_by_size!(height: 6)
Pear.find_or_create_by_size!(height: 3)
{% endhighlight %}

Using bang methods ensures we don't miss validation errors.
Using `find_or_create_by`  ensures we create the records only if they are not present.

If seeds change we can rerun the seed task.
If already existing seeds must be changed, it should be handled in migrations through update or delete statements.

All of this ensures that:

* migrations will be completely independent from the rest of the application code, and will not break even if run from the very start
* seeds are all in one place
* `db:seed` can always be run safely (e.g. as part of deployment)

This implies that when deploying both `db:migrate` and `db:seed` must be run.
I think invoking `db:seed` from a migration only when necessary might be a good solution too.

This covers topics that I struggled to find coherent answers to,
and this certainly isn't the one true way of doing this.
