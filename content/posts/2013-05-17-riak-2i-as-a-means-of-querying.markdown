---
layout: post
title: "Riak 2i as a means of querying"
date: 2012-05-23
comments: true
categories: riak ruby
---

## Introduction

Up tonight, I will explore how different Riak’s [secondary indexes](http://wiki.basho.com/Secondary-Indexes.html) feature is from Riak Search in the context of a query. I will be using the same structure as in my post on Riak Search and Map/Reduce.
Riak’s secondary indexes have a different setup than Riak Search. Whereas search had to be enabled on each node in turn, secondary indexes (or 2i from now on) are stored as metadata on each key/value pair. There are a few advantages to this approach and, at this present moment, a few disadvantages.

## Easier integration

The major disadvantage of Riak Search, in my mind, is that you have to know that you will need it before you start putting data into Riak. There is no way, right now, to index existing data in order to take advantage of the wonderful capabilities Riak Search offers. At this point in time, the only two ways to add the index is to blow away your dataset, enable search and restore from backup or stream keys and their values through the system after turning Riak Search on. I, admittedly, don’t fully understand the latter option, so when I decided to use Riak Search in my app, I did the former. It may not be possible for everyone to do so.
2i, on the other hand, is stored per key. This means that when you’re ready to add 2i to your data, it’s as simple as reading in a key, adding the index through the application layer and re-saving the data, illustrated below.

{% codeblock add index - event.rb %}
class Event
  def add_indexes(key)
    event = Event.find(key)
    event.indexes['month_year_bin'] << "#{event.month}-#{event.year}"
    event.indexes['year_month_int'] << "#{event.year}-#{event.month}".to_i
  end
end
{% endcodeblock %}

I’ve added two indexes that look the same for illustration purposes, as well as, because I’m unsure exactly how I will use them yet. Besides, deleting an unwanted index is as easy as deleting a key from a hash.

{% codeblock delete index - event.rb %}
event = Event.find("somekey")
event.indexes.delete("month_year_bin")
event.save
{% endcodeblock %}

It can be done while the system is up and running and serving users. This is a major boost in convenience.

## Querying

There are a several ways I can select a collection of keys from 2i.

### equality
The simplest is the equality query.

{% codeblock 2i m/r integration - event.rb %}
job = Riak::MapReduce.new(Event.client)
job.index("events", "year_month_int", "#{event.year}#{event.month}".to_i)
job.map("function(value, keydata, arg){ var data = Riak.mapValuesJson(value)[0]; if(data.location === \"\") {return [data];} else{ return [];} }", :keep => true)
job.run

#returns
[{"category"=>"business", "event_date"=>"2011-08-13T00:00:00+00:00", "location"=>""}, {"category"=>"business", "event_date"=>"2011-08-09T00:00:00+00:00", "location"=>""}]

{% endcodeblock %}

All right. I like that. I think I’ll keep the int index and drop the bin one. It’s easier to construct and I think it’s more efficient.

### range

Even better, the int index type enables me to do a range query easily. For example, let’s add another index that stores the day, along with month and year of an event and then select all of the events in the month range.

{% codeblock 2i m/r integration v2 - event.rb %}
event.indexes["year_month_day_int"] << "#{event.year}#{event.month}#{event.day}".to_i

#and then
job = Riak::MapReduce.new(Event.client)
job.index("events", "year_month_day_int", 201181..2011831)
job.map("function(value, keydata, arg){ var data = Riak.mapValuesJson(value)[0]; if(data.location === \"\") {return [data];} else{ return [];} }", :keep => true)
job.run

#returns
[{"category"=>"business", "event_date"=>"2011-08-13T00:00:00+00:00", "location"=>""}, {"category"=>"business", "event_date"=>"2011-08-09T00:00:00+00:00", "location"=>""}]

{% endcodeblock %}

That’s very nice. This enables me to offer to my users a weekly view of their data, as well as a monthly one. I don’t think I can manage that with Riak Search.
There is also another type of index, the multi-valued index, [whose usage is detailed](https://github.com/basho/riak-ruby-client/wiki/Secondary-Indexes) on the GitHub wiki. As usual, the results returned are piped back into my domain object which pulls out keys and values and creates out of them something I can use in the rest of the system.

## Disadvantages

The main disadvantage, for me, is the inability to combine multiple secondary indexes to pull out data. Ideally, I would love to only query by index without having to have a portion of my code in Javascript or Erlang to do the additional filtering. Also, as of right now, there is no consistent ordering of the keys selected through the index. That will be [corrected later](https://gist.github.com/2721295), but it is something that I need to be careful of right now.

## Conclusion

I think I’ll be able to live with the disadvantages as they are. I am not a real power user as far as I’ve seen and the current, minimal abilities of both Riak Search and Riak 2i suit me well.
Well, I’m off to build the front-end for the new goodies I discovered writing this post. Until the next time.