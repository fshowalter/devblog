---
layout: post
title: Auditing in Rails
---

au·dit·ing:
_noun_

1. a misnomer for entity-change logging.
2. a requirement that will surface in most any line-of-business app.
3. something that is kind of a pain to implement in Rails.

If you're building any kind of CRUD app, chances are your customer will eventually ask for some sort of auditing. Now, before you nod, spin around in your sweet Herman Miller chair (lucky bastard) and start coding, take a moment to ask some questions and suss out what your customer is _really_ asking for. Even assuming that they understand that they're really talking about entity-change logging, you still need to know, at minimum:

1. Do they really mean versioning? If so, don't reinvent the wheel. Use [Vestal Versions](https://github.com/laserlemon/vestal_versions)  or [Paper Trail](https://github.com/airblade/paper_trail).
2. Do they need to log before and after values?

For the sake of this post, let's assume their entity model is evolving rapidly and thus versioning isn't really practical, but they do want some kind of log reflecting changes, including before and after values.

Given those requirements, we need a way to know what changed on an entity after an update, including what the "old" value was. Now, lets look at how we might implement that in Rails.

## ActiveModel::Dirty

Rails actually provides a concept of 'dirty' attributes, including before/after values, that we can tap into: [ActiveModel::Dirty](http://api.rubyonrails.org/classes/ActiveModel/Dirty.html). Assume we're using the following model:

```ruby
# == Schema Information
#
# Table name: movies
#
#  title          :string(255)      not null
#  release_date   :date
class Movie < ActiveRecord::Base
end
```

This just works:

```ruby
movie = Movie.new
movie.title = 'Rio Bravo'
movie.release_date = Date.parse('1959-03-18')

movie.changed_attributes #=> {"title"=>nil, "release_date" =>nil}
movie.save

movie.changed_attributes #=> {}
movie.previous_changes #=> {"title"=>nil, "release_date" =>nil}
```

Sweet. We can get access to the changed values both before and after save, as well as the old values.

But what happens when things get a little more complicated?

## Aggregate Objects

Lets add in another couple of tables:

```ruby
# == Schema Information
#
# Table name: performers
#
#  name          :string(255)      not null
class Performer < ActiveRecord::Base
end

# == Schema Information
#
# Table name: movies_performers
#
#  performer_id          :integer      not null
#  movie_id              :integer      not null
```

And create a relation:

```ruby
# == Schema Information
#
# Table name: movies
#
#  title          :string(255)      not null
#  release_date   :date
class Movie < ActiveRecord::Base
  has_and_belongs_to_many :performers
end
```

The `has_and_belongs_to_many` association adds a `performer_ids` reader and writer allowing us do:

```ruby
movie = Movie.new
movie.performer_ids = [1, 2, 3]
movie.performer_ids #=> [1, 2, 3]
```

This is nice because it allows us to update the aggregate (the Movie) as well as it's children (the Performers) in a single call by passing a single params hash.

```ruby
movie = Movie.create(title: 'Rio Bravo', performer_ids: [1,2,3])
movie.performer_ids #=> [1, 2, 3]
```

But what about capturing these changes?

```ruby
chance = Performer.create(name: 'John Wayne')
dude = Performer.create(name: 'Dean Martin')
colorado = Performer.create(name: 'Ricky Nelson')

movie = Movie.new
movie.title = 'Rio Bravo'
movie.performer_ids = [chance.id, dude.id, colorado.id]
movie.performer_ids #=> [1, 2, 3]
movie.changed_attributes #=> { title: nil }
movie.save
movie.previous_changes #=> { title: nil }
```

Huston, we have a problem.

### Fixing the ids_writer Method

It seems that while Rails provides convince methods for updating an aggregate, those methods don't integrate with the `ActiveModel::Dirty` concern.

The method in question is generated here:

```ruby
module ActiveRecord
  module Associations
    class CollectionAssociation < Association
      # Implements the ids writer method, e.g.
      # foo.item_ids= for Foo.has_many :items
      def ids_writer(ids)
        pk_column = reflection.primary_key_column
        ids = Array(ids).reject { |id| id.blank? }
        ids.map! { |i| pk_column.type_cast(i) }
        replace(klass.find(ids).index_by { |r| r.id }.values_at(*ids))
      end
  end
end
```

Unfortunately, there's no easy fix. Instead, we have to chose the lesser of several evils.

1. We could fix this by simply avoiding the `ids_writer` method altogether and writing our own mapper, maybe utilizing a form-object, but what about API calls? Well, we could further abstract the form object into a generic presenter, but now we're writing our own mapping logic. But aren't we already representing that logic in our `ActiveRecord` models?

2. We could create our own association type, subclassing `CollectionAssociation` and override `ids_writer(ids)`, but we'd need to make sure we always use the custom macro instead of the traditional `has_many`, etc..

3. We could open up `CollectionAssociation` and fix `ids_writer`.

I always consider opening classes a last resort, but given that our fix here is a one-line, targeted fix, I'd be okay with it. If it makes you uncomfortable, you could try fix #2, but that's a lot of extra code for the same result.

```ruby
module ActiveRecord
  module Associations
    class CollectionAssociation < Association
      def ids_writer_with_changed_attributes(ids)
        old_ids = load_target.map(&:id)
        if ids_will_change(old_ids, ids)
          name = reflection.name.to_s.singularize
          owner.changed_attributes.merge!("#{name}_ids" => old_ids)
        end

        ids_writer_without_changed_attributes(ids)
      end

      alias_method_chain :ids_writer, :changed_attributes

      private

      def ids_will_change(old_ids, new_ids)
        (normalize_ids(old_ids) & normalize_ids(new_ids)).any?
      end

      def normalize_ids(ids)
         pk_column = reflection.primary_key_column
         ids = Array(ids).reject { |id| id.blank? }
         ids.map! { |i| pk_column.type_cast(i) }
         ids
      end
    end
  end
end
```

Add that to an initializer and restart our console:

```ruby
chance = Performer.create(name: 'John Wayne')
dude = Performer.create(name: 'Dean Martin')
colorado = Performer.create(name: 'Ricky Nelson')

movie = Movie.new
movie.title = 'Rio Bravo'
movie.performer_ids = [chance.id, dude.id, colorado.id]
movie.performer_ids #=> [1, 2, 3]
movie.changed_attributes #=> {"title" => nil, "performer_ids" => []}
movie.save
movie.previous_changes #=> {"title" => nil, "performer_ids" => []}
```

Sweet, now we've got changed attributes working for associations, right? Not quite.

### Nested Attributes

Lets add in another table:

```ruby
# == Schema Information
#
# Table name: premiers
#
# venue           :string(255)      not null
# date            :date             not null
class Premier < ActiveRecord::Base
  has_one :movie
end
```

And add a new relation:

```ruby
# == Schema Information
#
# Table name: movies
#
#  title          :string(255)      not null
# premier_id      :integer
#  year   :integer
class Movie < ActiveRecord::Base
  belongs_to :premier

  accepts_nested_attributes_for :premier
end
```

(Yes, this isn't the optimal way to structure these tables, but I'm trying to point out a Rails gotcha, so work with me.)

Once again Rails, provides us some nice methods to update the aggregate object and all its children with a single params hash. In this case:

```ruby
movie = Movie.new
movie.update_attributes(
  title: 'Rio Bravo',
  premier_attributes: {
    venue: "Grauman's Chinese Theatre", date: '1959-03-18' })
movie.premier
#=> { venue: "Grauman's Chinese Theatre", date: Wed, 18 Mar 1959 }
```

Now, how about those changed attributes?

##### The Autosave Gotcha

```ruby
movie = Movie.new
movie.title = 'Rio Bravo'
movie.premier_attributes =
  { venue: "Grauman's Chinese Theatre", date: '1959-03-18' }
movie.changed_attributes #=> { title: nil }
movie.save
movie.previous_changes #=> {}
```

Say what? I mean, forget the premier attributes, what happened to `title`? That's a little gotcha. To understand what's going on I had to dig into the Rails source. Basically, when you execute save the following occurs:

1. The `previous_changes` hash is replaced by the `changed_attributes` hash.
2. The `changed_attributes` hash is cleared.

So what's happening here? Well, save is getting called twice. When we call `movie.save`, Rails first calls save on the premier child-object so it can have an id to put in the movie's `premier_id` column. Due to Premier's `has_one :movie` association, the premier child-object calls save on it's associated movie, which in turn replaces the movie's `previous_changes` hash with the movie's `changed_attributes` hash and then clears the movie's `changed_attributes` hash.

At this point, Rails has a saved Premier object with an id. The movie's `changed_attributes` hash is empty and it's `previous_changes` hash is `{ "title" => nil }`.

Now that the child object is saved, Rails finishes saving our object, which _once again_ replaces the `previous_changes` hash, except now our `changed_attributes` hash is empty, thus making the `previous_changes` hash empty.

To change this behavior, we can tell our association not to auto save, since we know that Movie is our aggregate, and not Premier, by doing:

```ruby
# == Schema Information
#
# Table name: premiers
#
# venue           :string(255)      not null
# date            :date             not null
class Premier < ActiveRecord::Base
  has_one :movie, auto_save: false
end
```

Now, we get:

```ruby
movie = Movie.new
movie.title = 'Rio Bravo'
movie.premier_attributes =
  { venue: "Grauman's Chinese Theatre", date: '1959-03-18' }
movie.changed_attributes #=> { "title" => nil }
movie.save
movie.previous_changes #=> { "title" => nil }
```

But what about the changed attributes?

#### Fixing Nested Attributes

The code for nested attributes lives in the `ActiveRecord::NestedAttributes` module. The `accepts_nested_attributes_for` macro adds helper reader and writer methods to your model, similar to the `performer_ids` methods we saw earlier, only this time they're called `premier_attributes`.

So how do we fix it?

Well, again there's no easy fix, and again we've got to chose the lesser of several evils.

1. Once again, we could simply avoid nested attributes all together and write our own mapper, but again, aren't we already representing that logic in our `ActiveRecord` models?

2. Since this is a module and not a class, we can't subclass the methods we want directly, but we can alias the `accepts_nested_attributes_for` method to generate helpers that call aliased versions of the `premier_attributes` methods, but that's a lot of aliasing making for a confusing implementation.

3. We could override save and destroy in the child object and add the appropriate logic to append to the parent `changed_attributes`, but then changes would only reflected after a save, unlike the other attributes, resulting in inconsistent behavior.

3. We could open up `ActiveRecord::NestedAttributes` and fix the attribute writers.

Again, I always consider opening classes a last resort, but option is the simplest, with the smallest amount of code required. In this case it's a matter of:

```ruby
module ActiveRecord
  module NestedAttributes
    def assign_nested_attributes_for_one_to_one_association_with_changed_attributes(association_name, attributes)
      assign_nested_attributes_for_one_to_one_association_without_changed_attributes(association_name, attributes)
      altered_attributes = send(association_name).changed_attributes
      changed_attributes.merge!("#{association_name}_attributes" =>
        altered_attributes if altered_attributes.any?
    end

    alias_method_chain(
      :assign_nested_attributes_for_one_to_one_association,
      :changed_attributes)

    def assign_nested_attributes_for_collection_association_with_changed_attributes(association_name, attributes)
      assign_nested_attributes_for_collection_association_without_changed_attributes(association_name, attributes)
      altered_attributes = {}.with_indifferent_access
      attributes.each do |id, params|
        child_object = send(association_name).find(id)
        if child_object.marked_for_destruction?
          altered_attributes[id] => child_object.attributes
        elsif child_object.changed_attributes.any?
          altered_attributes[id] => child_object.changed_attributes
        end
      end

      changed_attributes.merge!("#{association_name}_attributes" =>
        altered_attributes if altered_attributes.any?
    end
  end
end
```

Ugh. Not so pretty, but it's the simplest thing that works. The end result is that we can do:

```ruby
movie = Movie.new
movie.title = 'Rio Bravo'
movie.premier_attributes =
  { venue: "Grauman's Chinese Theatre", date: '1959-03-18' }
movie.changed_attributes
#=> { "title" => nil,
#     "premier_attributes" => { "venue" => nil, "date" => nil }}
movie.save
movie.previous_changes
#=> { "title" => nil
#     "premier_attributes" => { "venue" => nil, "date" => nil } }
```

And if we update our model slightly we can even support deletes:

```ruby
# == Schema Information
#
# Table name: movies
#
#  title          :string(255)      not null
# premier_id      :integer
#  year   :integer
class Movie < ActiveRecord::Base
  belongs_to :premier

  accepts_nested_attributes_for :premier, allow_destroy: true
end

movie = Movie.new
movie.update_attributes(
  title: 'Rio Bravo',
  premier_attributes: {
    venue: "Grauman's Chinese Theatre", date: '1959-03-18' })

movie.premier
#=> <Premier, venue: "Grauman's Chinese Theatre", date: '1959-03-18'>

movie.update_attributes { premier_attributes: { _destroy: true } }

movie.premier #=> nil
movie.previous_changes
#=> { "premier_attributes" => {
# "venue" =>"Grauman's Chinese Theatre", "date" => '1959-03-18' }}
```

Now we've got an atomic way to update an aggregate object and get the original values for any changes attributes, even associations and child objects. Since we're opening classes, we're vulnerable breaking changes with each version of Rails, but a few simple unit tests should prevent unwanted surprises.
