---
layout: post
title: "Creating a Hash from a Collection"
date:   2014-03-12 16:35:45 -0400
excerpt: Learn about Ruby's ability to create hash lookups from Collections.
categories: code
---

I recently needed to create a hash lookup from an object using the ID as the lookup key and the full object as the value in our rails app. It wasn’t a difficult concept and there were several ways this could be accomplished; however, we just couldn’t find a way that made sense or didn’t involve looping. That is, until we found `index_by`.

## Setup

Let’s assume we have an `Animal` class and an `Animals` list class:

{% highlight ruby %}
class Animal
  def initialize(id, name)
    @id = id
    @name = name
  end
  attr_reader :id, :name
end

class Animals
  def initialize(animals)
    @animals = animals
  end

  def animal_list
    animals.each_with_index.map { |animal, i| Animal.new(i, animal) }
  end
  attr_reader :animals
end
{% endhighlight %}

So we’d have a list of animals:

{% highlight plaintext %}
animal_array = ['King Mukla', 'Misha', 'Leokk', 'Huffer']
animals = Animals.new(animal_array).animal_list
=> [
 [0] #<Animal:0x007fde9678e098 @id=0, @name="King Mukla">,
 [1] #<Animal:0x007fde9678e070 @id=1, @name="Misha">,
 [2] #<Animal:0x007fde9678e048 @id=2, @name="Leokk">,
 [3] #<Animal:0x007fde9678e020 @id=3, @name="Huffer">
]
{% endhighlight %}

## Using Hash[a]
Our first solution took advantage of `Hash`:

{% highlight ruby %}
hash = animals.map { |animal| [animal.id, animal] }
Hash[hash]
{% endhighlight %}

This was our default approach in conversion in the past, but creating an array of arrays just to use this approach seemed lengthy and unnecessary.

## Using Enumerable .reduce()
Our second solution took advantage of the reduce method in the Enumerable Class:

{% highlight ruby %}
animals.reduce({}) { |hash, animal|
  hash[animal.id] = animal
  hash
}
{% endhighlight %}

This is slightly better, since we don’t have to map on the array only to convert; but needing to return the hash at the end isn’t ideal either. Both solutions so far would usually require a doc lookup for .reduce and Hash[a]. 

## Using Enumerable .index_by()

Our final solution, we used the Enumerable `index_by`:

{% highlight ruby %}
animals.index_by(&:id)
{% endhighlight %}

Note: `.index_by` accepts a block. So if you want to index by two fields such as first and last name, you could easily do this with the block:

{% highlight ruby %}
people.index_by { |person| "#{person.first_name} #{person.last_name}" }
{% endhighlight %}

All three solutions yield the same result; a hash with the keys being the ID of the object:

{% highlight plaintext %}
{
  0 => #<Animal:0x007fde935ce530 @id=0, @name="King Mukla">,
  1 => #<Animal:0x007fde935ce440 @id=1, @name="Misha">,
  2 => #<Animal:0x007fde935ce418 @id=2, @name="Leokk">,
  3 => #<Animal:0x007fde935ce3f0 @id=3, @name="Huffer">
}
{% endhighlight %}
