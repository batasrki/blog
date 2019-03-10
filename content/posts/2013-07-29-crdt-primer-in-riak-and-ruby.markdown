---
layout: post
title: "CRDT primer in Riak and Ruby"
date: 2013-07-29 23:31
comments: true
categories: riak ruby crdt distributed systems
plugins: backtick_codeblock
---
## Introduction

In this post, I shall attempt how to use CRDTs in a Ruby class, backed by the Riak database. CRDT stands for [**Commutative Replicated Data Type**](http://highscalability.com/blog/2010/12/23/paper-crdts-consistency-without-concurrency-control.html). There is no Wikipedia entry for this, yet, so I'm linking to a blog post which is linking to a paper.

## Background

CRDTs solve a particular problem well. In a distributed database, like Riak, it is quite possible for a value under a key to receive multiple writes from different clients. Now, by default, Riak will discard everything but the latest write. However, it is possible to instruct Riak to keep all conflicting writes, so that we may resolve the conflicts at the application level. Resolving these conflicts can be really hard and this is where CRDTs step in.

It needs to be said that, as with almost everything in software development, whether or not one needs to use these data types depends on one's application and data usage needs. It's quite possible that keeping the last write is all that a developer will ever want.

While the concept itself may be fairly easy to grasp, the implementation of the concept has been a struggle for me. I hope to get more of an understanding, while explaining this topic.

## Let's get started

There are a few Ruby gems implementing CRDTs in existence. The most known one is [meangirls](https://github.com/aphyr/meangirls/tree/master/lib) by distributed systems extraordinaire, [Kyle Kingsbury](https://github.com/aphyr). We will be using this one, as it seems the most complete.

> Caveat: the *meangirls* library isn't gemified, so we will have to include its source. Also, it only takes care of creating the proper JSON representation of various CRDTs it supports. We have to take care of storing the representation, as well as how things are added or removed from a set.

Firstly, let's create a directory for the test code and pull down the source code for *meangirls*.

```
$ mkdir crdt-test
$ cd crdt-test
$ mkdir vendor
$ git clone git@github.com:aphyr/meangirls.git vendor/
$ touch Gemfile tester.rb
```

Next, let's add the Ruby Riak client gem and a JSON gem.

``` ruby Gemfile
source 'https://rubygems.org'

gem 'yajl-ruby'
gem 'riak-client'
```
Let's install those and we will be off and running:

```
$ bundle install
```
OK, with all that set up, let's start playing around with the *meangirls* library. In this post, I will use the two-phase-set CRDT, but I will add the how-tos for other supported types in later blog posts.

The two-phase-set CRDT is interesting, because it allows us to add and remove things from a set while never actually deleting data. It also prevents re-insertion of deleted elements, which can come in handy.

Let's first set up a playground.

``` ruby tester.rb
require_relative 'vendor/lib/meangirls.rb'
require 'yajl'
require 'riak'

client = Riak::Client.new(:nodes => [
                          {:host => "33.33.33.11", :http_port => 8111},
                          {:host => "33.33.33.12", :http_port => 8112},
                          {:host => "33.33.33.13", :http_port => 8113}
                          ])
bucket = client.bucket 'meangirls'
bucket.allow_mult = true
p bucket.props
```
We load in the library and the two supporting gems. We then hook up to a Riak cluster and create a bucket for our data.

Let's open up irb and write some code.

``` ruby irb
require './tester.rb'
twop_set = Meangirls::TwoPhaseSet.new

twop_set.add :alpha
 => #<Meangirls::TwoPhaseSet:0x007fd7549680b0 @a=#<Set: {:alpha}>, @r=#<Set: {}>>
twop_set.add :beta
 => #<Meangirls::TwoPhaseSet:0x007fd7549680b0 @a=#<Set: {:alpha, :beta}>, @r=#<Set: {}>>
twop_set.to_json
 => "{\"type\":\"2p-set\",\"a\":[\"alpha\",\"beta\"],\"r\":[]}"
 twop_set.delete :alpha
  => :alpha
twop_set.to_json
 => "{\"type\":\"2p-set\",\"a\":[\"alpha\",\"beta\"],\"r\":[\"alpha\"]}"
 twop_set.add :alpha
 Meangirls::ReinsertNotAllowed: Meangirls::ReinsertNotAllowed
  from /Users/spejic/code/crdt-post/vendor/lib/meangirls/two_phase_set.rb:25:in '<<'
  from (irb):15
  from /Users/spejic/.rvm/rubies/ruby-2.0.0-p195/bin/irb:16:in '<main>'
```
As we can see, we can add elements to the set, remove elements from the set (by adding them to the remove set), get a JSON representation of the merge between the *add* and *remove* sets and we are prevented from re-adding a deleted element to the whole thing. The last part makes sense, because we didn't actually delete the element, it's still there in the *add* set. How would we add the same element again if we never removed it?

Before explaining why we're using this CRDT, let's first come up with a way to store the data.

Let's create an empty two phase set and store that in Riak.

``` ruby irb
twop_set = Meangirls::TwoPhaseSet.new
data_object = bucket.new
data_object.raw_data = twop_set.to_json
data_object.store
=> #<Riak::RObject {meangirls,7ELYcWLmeoaiphhgCjyemyz54JB} [#<Riak::RContent [application/json]:{"type"=>"2p-set", "a"=>[], "r"=>[]}>]>
```
OK, the set is stored. One of the usual patterns used while developing Riak-backed applications is to store the keys in something like Redis or memcache. This allows the developers to access the data quickly, since key-based access is the fastest type of data access in Riak.

However, this also opens up the possibility that two distinct sets of data will be written to the same key. This is exactly what a CRDT aims to prevent.

Let's simulate that possibility and see how *meangirls* handles it.

``` ruby irb
fetch_object = bucket.get '7ELYcWLmeoaiphhgCjyemyz54JB'

# fetch another object
another_fetch_object = bucket.get '7ELYcWLmeoaiphhgCjyemyz54JB'

# make sets
set_from_first = Meangirls::TwoPhaseSet.new(fetch_object.data)
set_from_second = Meangirls::TwoPhaseSet.new(another_fetch_object.data)

# add data to each set
set_from_first.add(:alpha)
set_from_second.add(:bravo)

# serialize each set back
fetch_object.raw_data = set_from_first.to_json
fetch_object.store

another_fetch_object.raw_data = set_from_second.to_json
another_fetch_object.store

# fetch the conflicts and merge them
fetch_object = bucket.get '7ELYcWLmeoaiphhgCjyemyz54JB'
fetch_object.conflict?
=> true

conflicted_parts = fetch_object.siblings.map do |sibling|
  Meangirls::TwoPhaseSet.new(sibling.data)
end
=> [#<Meangirls::TwoPhaseSet:0x0000000309d898 
            @a=#<Set: {"beta"}>, @r=#<Set: {}>>,
    #<Meangirls::TwoPhaseSet:0x0000000309d578 @a=#<Set: {"alpha"}>, @r=#<Set: {}>>]

resolved = conflicted_parts.first.merge(conflicted_parts.last)
=> #<Meangirls::TwoPhaseSet:0x00000003501c90
    @a=#<Set: {"beta", "alpha"}>, @r=#<Set: {}>>

# write the merged object back
resolved_data_object = Riak::RObject.new(bucket, fetch_object.key)
resolved_data_object.raw_data = resolved.to_json
resolved_data_object.content_type = "application/json"

fetch_object.siblings = [resolved_data_object]
fetch_object.store
```

We did a few things here, so let's break it down. Firstly, fetching the same object out of Riak twice will ensure that we have to references to the same **vector clock** value. [Vector clocks](http://docs.basho.com/riak/latest/theory/concepts/Vector-Clocks/) are how a distributed system like Riak keeps track of object updates in the system.

Having the same vector clock on 2 in-memory instances of our Riak object means that were both of them to write updates to that object, we would have a conflict. The default way Riak handles conflicts is by using the *last write wins* strategy, which is determined by the timestamp of the write coming into the system. All previous writes are discarded in that case, potentially losing information.

However, since we set our bucket to allow multiple values for an object, Riak will instead keep all conflicts and present us with them.

This is exactly the scenario we follow. We add an item to each set and write both sets back using our 2 in-memory references. Riak sees the conflict, but keeps both writes.

Upon the next read, we are presented with those conflicts. The Ruby client for Riak does a nice thing here and keeps things coherent by encapsulating those conflicts into the *Riak::RObject* instance.

Accessing any field on this conflicted instance before resolving the conflicts will result in an exception, forcing us to resolve the conflicts before proceeding.

``` ruby conflicted
fetch_object = bucket.get '7ELYcWLmeoaiphhgCjyemyz54JB'
fetch_object.conflict?
=> true
fetch_object.data
#=> Riak::Conflict: The object is in conflict (has siblings) and cannot be treated singly or saved: #<Riak::RObject {meangirls,7ELYcWLmeoaiphhgCjyemyz54JB} [#<Riak::RContent [application/json]:{"type"=>"2p-set", "a"=>["beta"], "r"=>[]}>, #<Riak::RContent [application/json]:{"type"=>"2p-set", "a"=>["alpha"], "r"=>[]}>]>
  from /home/srdjan/.rvm/gems/ruby-2.0.0-p195@crdt-post/gems/riak-client-1.2.0/lib/riak/robject.rb:169:in `content'
  from (irb):91
  from /home/srdjan/.rvm/rubies/ruby-2.0.0-p195/bin/irb:13:in `<main>'
```

So, we have conflicts and need to resolve them. The conflicted objects are presented as a collection of *siblings*. We loop through the collection and create *TwoPhaseSet* objects out of each one and add them to a collection of their own. We only have 2 conflicts here, so I cheated a bit on how they merge together, but I hope that the intent is clear.

``` ruby resolve conflicts
conflicted_parts = fetch_object.siblings.map do |sibling|
  Meangirls::TwoPhaseSet.new(sibling.data)
end
=> [#<Meangirls::TwoPhaseSet:0x0000000309d898 
            @a=#<Set: {"beta"}>, @r=#<Set: {}>>,
    #<Meangirls::TwoPhaseSet:0x0000000309d578 @a=#<Set: {"alpha"}>, @r=#<Set: {}>>]

resolved = conflicted_parts.first.merge(conflicted_parts.last)
=> #<Meangirls::TwoPhaseSet:0x00000003501c90
    @a=#<Set: {"beta", "alpha"}>, @r=#<Set: {}>>

```

The last part of the conflict resolution is tricky and under-documented. I only have two tweets from [Sean Cribbs](https://twitter.com/seancribbs) to go on.

> @batasrki @davidannweiser No, just return the same object, with siblings resolved. Best not to immediately write back though

and

>  @batasrki RObject.siblings = [ resolved_content_object ]

I, honestly, could not find any other information on this part, so what I'm presenting is probably not the best way. It does work, however.

``` ruby create a new object
# write the merged object back
resolved_data_object = Riak::RObject.new(bucket, fetch_object.key)
resolved_data_object.raw_data = resolved.to_json
resolved_data_object.content_type = "application/json"

fetch_object.siblings = [resolved_data_object]
fetch_object.store
```

What we're doing here is creating a new *Riak::RObject* instance, but setting its key to be the same as the conflicted object. This ensures that we update the data in Riak rather than create a new copy of it.

We then set that object's *raw_data* property to the resolved set we created earlier and its *content type* to JSON. We don't store this object, instead we overwrite the conflicted object's siblings collection with a list of size 1 containing the new object.

We then store the conflicted object back into Riak. If we pull this data from Riak subsequently, we will see the updated, resolved set store and no conflicts.

### Conclusion

So, there we have it. This is a first in a few posts on how to use CRDTs to resolve conflicts in an eventually consistent system. This is a powerful technique (pattern?), especially when using a distributed database like Riak.

In the next post, I will attempt to use the 2-phase set to resolve conflicts in richer JSON documents like user records.

In posts after that one, I will explore the other provided data types in the **Meangirls** library.

Until next time, please watch Kyle Kingsbury's excellent Ricon East video on what happens in distributed databases when CRDTs aren't used, [Call me maybe](http://www.youtube.com/watch?v=mxdpqr-loyA).