---
layout: post
title: "CRDT for real data in Riak and Ruby"
date: 2013-10-02 22:00
comments: true
categories: riak ruby crdt distributed systems
plugins: backtick_codeblock
---

## Intro

In the [previous post](http://batasrki.github.io/blog/2013/07/29/crdt-primer-in-riak-and-ruby/), I briefly introduced CRDTs in Ruby and have shown a basic usage case for one Conflict-Free Replicated Data Type. Tonight, we will explore one slightly more complicated use case that can be seen in real applications in the wild.

## Use case

For the purpose of this post, imagine we're running a Netflix clone. We have accounts and we have movies. For the purposes of recommendations, we want to store what movies each account holder has viewed. As is the case with Netflix, an account may be signed in and used across more than one device by holders of differing tastes. Since we don't really want to impose the restriction of just one account being signed in, we will record statistics from each session.

As you can see, this may lead to write conflicts, since any account holder may want to watch any movie at any time. For the sake of efficiency, we will only store an ID to each movie in the account's **movies watched** collection.

## The data type used

Out of the currently known CRDTs, the one that seems to fit this problem the best is the GSet, or Grow-only Set, data type. As its name suggests, this CRDT only ever grows in size, it does not shrink. Since recommendation engines thrive on tonnes of data, having an ever-increasing set of records is very beneficial.

The basic layout of a GSet is as follows:

``` json
{
  'type': 'g-set',
  'e': ['a', 'b', 'c']
}

```

The type information there is for the [meangirls library](https://github.com/aphyr/meangirls) to invoke the correct operational semantics. Other than that, it's just a simple list of data, always growing in size. Only, the **meangirls** library does not include a class for a GSet. It has a class for TwoPhaseSet, which is a combination of two GSets, but it doesn't actually implement a GSet by itself.

That's not a big deal, we will implement one ourselves.

## GSet implementation

Since it has fairly simple requirement, let's take a look at the GSet class. I'll explain parts of it after the code listing.

``` ruby gset.rb
class GSet < Meangirls::Set
  attr_accessor :a

  def initialize(hash=nil)
    if hash
      raise ArgumentError, "hash must contain a" unless hash['a']

      @a = Set.new hash['a']
    else
      # Empty set
      @a = Set.new
    end
  end

  def <<(e)
    @a << e
    self
  end
  alias add <<

  def ==(other)
    other.kind_of?(self.class) && a == other.a
  end

  def as_json
    {
      'type' => type,
      'a' => a.to_a 
    }
  end

  def clone
    c = super
    c.a = a.clone
    c
  end

  def merge(other)
    unless other.kind_of? self.class
      raise ArgumentError, "other must be a #{self.class}"
    end

    self.class.new('a' => (a | other.a))
  end

  def include?(e)
    @a.include? e
  end

  def to_set
    @a
  end

  def type
    'g-set'
  end
end
```

If you peek at **meangirls** implementation of a `TwoPhaseSet`, you will notice that we copied most of its code into here, save for the *remove set* logic. Let's go through the important parts.

``` ruby
def initialize(hash=nil)
  if hash
    raise ArgumentError, "hash must contain a" unless hash['a']

    @a = Set.new hash['a']
  else
    # Empty set
    @a = Set.new
  end
end
```

Our constructor may or may not take a hash. This is done, so that we can re-construct a `GSet` from JSON data, such as a Riak record.

``` ruby
def <<(e)
  @a << e
  self
end
alias add <<
```

We only ever add elements to our `GSet`. Additional guarantees, like that an element can only be an Integer, can be added here.

``` ruby
def merge(other)
  unless other.kind_of? self.class
    raise ArgumentError, "other must be a #{self.class}"
  end

  self.class.new('a' => (a | other.a))
end
```

When merging, we first must ensure that we're merging a `GSet` to a `GSet`. We would get into weird situations and breakage, otherwise. After that, a simple set union over our 2 sets of IDs will ensure we only ever have unique ID values here.

## Setup

With that done, let's add models for the `Account` records and `Movie` records. Persistence is done somewhere else. This time, we will store records through irb.

``` ruby account.rb
class Account
  attr_accessor :username, :email, :movies_watched

  def initialize(hash=nil)
    if hash
      @username = hash["username"]
      @email = hash["email"]
      @movies_watched = GSet.new(hash["movies_watched"])
    else
      @movies_watched = GSet.new
    end
  end

  def as_json
    {
      "username" => username,
      "email" => email,
      "movies_watched" => movies_watched.as_json
    }
  end

  def to_json
    as_json.to_json
  end
end
```

``` ruby movie.rb
class Movie
  attr_accessor :title

  def initialize(hash=nil)
    if hash
      @title = hash["title"]
    end
  end

  def as_json
    {
      "title" => title
    }
  end

  def to_json
    as_json.to_json
  end
end
```

Let's store a few arbitrary movies, keep their keys in memory and cause a write conflict. We will then see how we can resolve this conflict through the usage of our CRDT.

``` ruby irb
movie_bucket = client.bucket("movies")
 => #<Riak::Bucket {movies}>

movie_record_keys = []

m1 = Movie.new("title" => "Test Movie")
 => #<Movie:0x007ffa0be21348 @title="Test Movie">

m2 = Movie.new("title" => "Another test movie")
 => #<Movie:0x007ffa0be43ba0 @title="Another test movie">

r1 = movie_bucket.new
 => #<Riak::RObject {movies} [#<Riak::RContent [application/json]:nil>]>

r1.data = m1.as_json
 => {"title"=>"Test Movie"}

r1.store
 => #<Riak::RObject {movies,CAQ3ewkn0vVI5q8Kxt4LPcrFYk9} [#<Riak::RContent [application/json]:{"title"=>"Test Movie"}>]>

movie_record_keys << r1.key

r2 = movie_bucket.new
 => #<Riak::RObject {movies} [#<Riak::RContent [application/json]:nil>]>

r2.data = m2.as_json
 => {"title"=>"Another test movie"}

r2.store
 => #<Riak::RObject {movies,aU6uCCZJ10xGIPJonpSYftZ4H0w} [#<Riak::RContent [application/json]:{"title"=>"Another test movie"}>]>

movie_record_keys << r2.key
 => ["CAQ3ewkn0vVI5q8Kxt4LPcrFYk9", "aU6uCCZJ10xGIPJonpSYftZ4H0w"]
```

OK, now we have two movies, so let's simulate a simultaneous account access by 2 holders, each of which is watching a movie. As shown in the previous post, storing values at the same time will cause a write conflict, resulting in creation of siblings. As usual, the `accounts` bucket should have `allow_mult` set to **true**, so that Riak doesn't discard one of the writes.

Let's first create an account.

``` ruby irb
accounts_bucket = client.bucket("accounts")
 => #<Riak::Bucket {accounts}>

accounts_bucket.allow_mult = true
 => true

account = Account.new
 => #<Account:0x007ffa0ce89758 @movies_watched=#<GSet:0x007ffa0ce89730 @a=#<Set: {}>>>

account.username = "an-user"
 => "an-user"
account.email = "anuser@email.com"
 => "anuser@email.com"

record = accounts_bucket.new
 => #<Riak::RObject {accounts} [#<Riak::RContent [application/json]:nil>]>

record.data = account.as_json
 => {"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>[]}}

record.store
 => #<Riak::RObject {accounts,6z4xJjAFMJVHWn6Ni2MF5J3IBBq} [#<Riak::RContent [application/json]:{"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>[]}}>]>
```

OK, with the record stored, we retrieve it twice without updating. This ensures that the `vclock` value for the record is not updated, allowing us the opportunity to cause a write conflict.

``` ruby irb
one_access = accounts_bucket.get "6z4xJjAFMJVHWn6Ni2MF5J3IBBq"
 => #<Riak::RObject {accounts,6z4xJjAFMJVHWn6Ni2MF5J3IBBq} [#<Riak::RContent [application/json]:{"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>[]}}>]>

# same as above
second_access = accounts_bucket.get "6z4xJjAFMJVHWn6Ni2MF5J3IBBq"

first_instance = Account.new(one_access.data)
 => #<Account:0x007ffa0cf2b710 @movies_watched=#<GSet:0x007ffa0cf2b670 @a=#<Set: {}>>, @username="an-user", @email="anuser@email.com">

second_instance = Account.new(second_access.data)
```

Let's now add the ID of the first created movie to one instance of this `Account` record and the ID of the second movie to the other one.

``` ruby irb
first_instance.movies_watched.add(movie_record_keys.first)
 => #<GSet:0x007ffa0cf2b670 @a=#<Set: {"CAQ3ewkn0vVI5q8Kxt4LPcrFYk9"}>>

second_instance.movies_watched << movie_record_keys.last
 => #<GSet:0x007ffa0cf31f70 @a=#<Set: {"aU6uCCZJ10xGIPJonpSYftZ4H0w"}>>
```

Let's now try storing these objects back into Riak and seeing what happens.

``` ruby irb
one_access.data = first_instance.as_json
 => {"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>["CAQ3ewkn0vVI5q8Kxt4LPcrFYk9"]}}

one_access.store
 => #<Riak::RObject {accounts,6z4xJjAFMJVHWn6Ni2MF5J3IBBq} [#<Riak::RContent [application/json]:{"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>["CAQ3ewkn0vVI5q8Kxt4LPcrFYk9"]}}>]>

second_access.data = second_instance.as_json
 => {"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>["aU6uCCZJ10xGIPJonpSYftZ4H0w"]}}

second_access.store
 => #<Riak::RObject {accounts,6z4xJjAFMJVHWn6Ni2MF5J3IBBq} [#<Riak::RContent [application/json]:{"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>["aU6uCCZJ10xGIPJonpSYftZ4H0w"]}}>, #<Riak::RContent [application/json]:{"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>["CAQ3ewkn0vVI5q8Kxt4LPcrFYk9"]}}>]>
```

And, there we go. We have successfully (again) caused a conflict. Let's write some code to resolve it. The trick here to resolving conflicts like this is that you must not access the `data` nor the `raw_data` methods. These will throw an exception and mess up your day.

Let's instead, introduce a class method in the `Account` model that will handle conflict resolution for us.

``` ruby account.rb
class Account
  # existing methods
  #
  #
  #
  def self.from_persistence(obj)
    if obj.conflict?
      resolved = {}
      watched_lists = []
      obj.siblings.each do |sibling|
        watched_lists << sibling.data.delete("movies_watched")
        resolved.merge!(sibling.data)
      end

      first_one = GSet.new(watched_lists.shift)
      watched_lists.each {|list| first_one = first_one.merge(GSet.new(list))}

      resolved.merge!("movies_watched" => first_one.as_json)
      new(resolved)
    else
      new(obj.data)
    end
  end
end
```

In this method, we check first if the raw record coming from Riak has the `conflict?` flag set, which indicates that there's been a write conflict somewhere. If it is, we loop through all the `siblings` and pull out the **movies_watched** lists. The rest of this record should not be subject to conflict resolution, so we will accept whatever values there are for them.

We then initialize the first item in the list of `GSet`s and then merge the others in that list with the first one. We append this to the hash of resolved values and then instantiate our `Account` model with the new values.

The second part to this conflict resolution is storing the resolved object back into Riak. Again, as in the previous post, we do that by creating a new `Riak::RObject` instance, setting its key to the conflicted one and setting the `siblings` property of the conflicted instance to an array of `Riak::RObject`s of size 1. The only member is the new instance. We store that and we're done.

``` ruby irb
record = accounts_bucket.get "6z4xJjAFMJVHWn6Ni2MF5J3IBBq"
 => #<Riak::RObject {accounts,6z4xJjAFMJVHWn6Ni2MF5J3IBBq} [#<Riak::RContent [application/json]:{"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>["aU6uCCZJ10xGIPJonpSYftZ4H0w"]}}>, #<Riak::RContent [application/json]:{"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>["CAQ3ewkn0vVI5q8Kxt4LPcrFYk9"]}}>]>

a = Account.from_persistence(record)
 => #<Account:0x007ff318320c68 @username="an-user", @email="anuser@email.com", @movies_watched=#<GSet:0x007ff318320bc8 @a=#<Set: {"aU6uCCZJ10xGIPJonpSYftZ4H0w", "CAQ3ewkn0vVI5q8Kxt4LPcrFYk9"}>>>

resolved = accounts_bucket.new
 => #<Riak::RObject {accounts} [#<Riak::RContent [application/json]:nil>]>

resolved.data = a.as_json
 => {"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>["aU6uCCZJ10xGIPJonpSYftZ4H0w", "CAQ3ewkn0vVI5q8Kxt4LPcrFYk9"]}}

resolved.key = record.key
 => "6z4xJjAFMJVHWn6Ni2MF5J3IBBq"

record.siblings = [resolved]
 => [#<Riak::RObject {accounts,6z4xJjAFMJVHWn6Ni2MF5J3IBBq} [#<Riak::RContent [application/json]:{"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>["aU6uCCZJ10xGIPJonpSYftZ4H0w", "CAQ3ewkn0vVI5q8Kxt4LPcrFYk9"]}}>]>]

record.store
 => #<Riak::RObject {accounts,6z4xJjAFMJVHWn6Ni2MF5J3IBBq} [#<Riak::RContent [application/json]:{"username"=>"an-user", "email"=>"anuser@email.com", "movies_watched"=>{"type"=>"g-set", "a"=>["aU6uCCZJ10xGIPJonpSYftZ4H0w", "CAQ3ewkn0vVI5q8Kxt4LPcrFYk9"]}}>]>
```

## Conclusion

Hopefully, this has shed a bit more light on how CRDTs can be used in real applications. The big key here is that during data model design, a need for conflict resolution must be anticipated and then the data, or parts of it, must be structured as a CRDT. Once that is done, merging conflicted records is pretty easy and availability is preserved.

Until next time.