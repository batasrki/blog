---
layout: post
title: "Riak Search as a means of querying"
date: 2012-04-24
comments: true
categories: riak ruby
---

### Introduction

Up tonight, I’ll explore enhancing the querying in my app with the use of Riak Search. Basho has recently added a list-keys warning to the Ruby driver and all of my current MapReduce code is triggering it. Riak Search should help there by enabling me to reduce the set of keys in the events bucket to the ones closer to what I’m looking. Since that won’t traverse the entire bucket, I’ll avoid triggering the warning.

### Background

To give you some background, buckets in Riak are just namespaces for keys and their values plus some metadata about each key/value pair. As such, writing a MapReduce query that selects only the bucket, and not some range of keys within that bucket, will cause Riak to traverse all of the keys on all of the nodes in the system. Needless to say that operation will get extremely slow as the number of keys grows. There is a bit more explanation on the recently added [wiki page](https://github.com/basho/riak-ruby-client/wiki/Key-Value-Operations) for the Riak Ruby client.

#### What not to do

As a quick example of what not to do for a production app, but something that you should play with in development, here’s a sample MapReduce query that tries to pull out events for a certain month:

{% codeblock  lang:ruby %}
# month is number from 1-12, validated elsewhere
# self.client is an instance of Riak::Client which hooks you up to the database
# parameter to add is the bucket name
def events_for(month)
  job = Riak::MapReduce.new(self.client).add("events")
  job.map("function(value, keydata, arg){ var item = Riak.mapValuesJson(value)[0]; var mo = item.event_date.split('-')[1]; if(month == #{month}){ return [data];}}", :keep => true)
  job.run
end
{% endcodeblock %}

The _Riak::MapReduce_ object is initialized with a client object that represents the database connection, whether it’s a cluster or a single node. Its _add_ method takes a few parameters, but initially all I comprehended it needed was a bucket name. The other two parameters concern themselves with keys and key filters, the latter being out of scope for this blog post.

The _map_ function takes a string of Javascript which is run on each node that has the data. This keeps the querying of the data very close to where data lives, making it very easy to not worry about node availability. There are built-in Javascript functions (and I’m using one there), as well as a set of [community-contributed functions](http://contrib.basho.com/) that one can use.

It’s in the Javascript function that I do the filtering by month. In another post, I’ll explore pushing functions down to each node as opposed to mixing a lot of Javascript into my Ruby code. While this looked like a straightforward way to introduce dynamic querying to a key/value store (and I felt smug about it), it has the wonderful property of traversing every single key looking for those namespaced to the events bucket and then shoving those one-by-one into my Javascript function. The function manages to find the documents whose date attribute has the month I want and returns a collection of those as items in a hash to Ruby. It’s, now, quite clearly the wrong way to go about querying in Riak. Let’s see if I can make it better.

### Riak Search

Out of the two enhancements to the querying capabilities added by [Basho](http://www.basho.com) to Riak, [Riak Search](http://docs.basho.com/riak/latest/tutorials/querying/Riak-Search/) is the older one. I am starting to appreciate having a full-text search engine tightly integrated into a database the way Riak, as well as PostgreSQL, has done. Having started my Rails career with MySQL-backed apps and futzing around with various search solutions, this is another thing I appreciate not having to worry about.

#### Enabling Riak Search

First thing I need to do is enable the full-text search on each node in the cluster. This is something that is easily automated, so it’s going on the growing list of things to automate. Right now, I’ll do it manually.

After getting the [cluster up and running](/blog/2012/04/22/riak-and-vagrant/), I’ll edit each node’s app.config to enable the searching. Changing the value of _enabled_ from false to true will get it going.

{% codeblock app.config lang:erlang %}
%% Riak Search Config
{riak_search, [
              %% To enable Search functionality set this 'true'.
              {enabled, true}
              ]},
{% endcodeblock %}

#### Indexing data

Both the wiki and Mathias Meyer’s great [Riak Handbook](http://riakhandbook.com) provide syntax for enabling the pre-commit hook for the search engine. The hook will index new data before it’s committed to the database.

Note I said _new_ data. As of right now, the pre-commit hook will not index existing records. I do feel that this is a bit of a weakness and something to be corrected, though. Anyway, cribbing from the wiki, the syntax to enable the pre-commit hook is:

{% codeblock %}
search-cmd install events
 :: Installing Riak Search <--> KV hook on bucket 'events'.
{% endcodeblock %}

With that done and with the database primed with a few records, let’s see if we can actually search the events.

#### Searching

OK, so, going back to the original MapReduce function, what I wanted were events whose date was in March. The search syntax for that is the following:

{% codeblock Event model in irb lang:ruby %}
Event.client.search("events", "event_date:*03*")

{"responseHeader"=>{"status"=>0, "QTime"=>16, "params"=>{"q"=>"event_date:*03*", "q.op"=>"or", "filter"=>"", "wt"=>"json"}}, "response"=>{"numFound"=>0, "start"=>0, "maxScore"=>"0.0", "docs"=>[]}}
{% endcodeblock %}

As I’m experimenting with search, I don’t have an elegant abstraction around the lower-level driver syntax just yet. That’s a TODO for another night. What I find strange is that I know there are events in March, but this search hasn’t found any. That’s either because the documents weren’t indexed or I did something wrong.

Let’s try with the year and month instead of just month. That’s closer to how it’ll be used anyway:

{% codeblock Event model in irb, part 2 lang:ruby %}
Event.client.search("events", "event_date:2012-03*")

{"responseHeader"=>{"status"=>0, "QTime"=>22, "params"=>{"q"=>"event_date:2012-03*", "q.op"=>"or", "filter"=>"", "wt"=>"json"}}, "response"=>{"numFound"=>25, "start"=>0, "maxScore"=>"0.00000e+0", "docs"=>[{"id"=>"1iCUQIvt4Zz7cccLFjDPAr4pLFf", "index"=>"events", "fields"=>{"category"=>"personal", "event_date"=>"2012-03-16T00:00:00+00:00", "location"=>"Vaughan"}, "props"=>{}}, {"id"=>"1uMMj5EBEJF26gGzM6ORvJIxBym", "index"=>"events", "fields"=>{"category"=>"personal", "event_date"=>"2012-03-31T00:00:00+00:00", "location"=>"Toronto"}, "props"=>{}}, {"id"=>"3eL8Zsq7uhXMcNVkiT2EqMOmjIx", "index"=>"events", "fields"=>{"category"=>"personal", "event_date"=>"2012-03-11T00:00:00+00:00", "location"=>"Maple"}, "props"=>{}}, {"id"=>"63uNQcQqunUIWuxFaUyFJmJvquf", "index"=>"events", "fields"=>{"category"=>"personal", "event_date"=>"2012-03-31T00:00:00+00:00", "location"=>"Vaughan"}, "props"=>{}}, {"id"=>"891nojSWgC86mc9a3sgBzogEhAB", "index"=>"events", "fields"=>{"category"=>"business", "event_date"=>"2012-03-09T00:00:00+00:00", "location"=>""}, "props"=>{}}, {"id"=>"8WFTYibfz46VNjzJeBYHpcvAKOU", "index"=>"events", "fields"=>{"category"=>"personal", "event_date"=>"2012-03-28T00:00:00+00:00", "location"=>"Vaughan"}, "props"=>{}}, {"id"=>"8XJZR8K6sXYylBz4Kq1xRwsiddd", "index"=>"events", "fields"=>{"category"=>"business", "event_date"=>"2012-03-13T00:00:00+00:00", "location"=>""}, "props"=>{}}, {"id"=>"8yfHBxxhSM12dM9u7RsCjInHehV", "index"=>"events", "fields"=>{"category"=>"personal", "event_date"=>"2012-03-16T00:00:00+00:00", "location"=>"Vaughan"}, "props"=>{}}, {"id"=>"9dAuFCdh32OKIoIMgGWtt1kE4gZ", "index"=>"events", "fields"=>{"category"=>"business", "event_date"=>"2012-03-23T00:00:00+00:00", "location"=>"Toronto"}, "props"=>{}}, {"id"=>"Alj2Iwqg0D3dCwAENMjDbeggM2t", "index"=>"events", "fields"=>{"category"=>"personal", "event_date"=>"2012-03-07T00:00:00+00:00", "location"=>"Woodbridge"}, "props"=>{}}]}}
{% endcodeblock %}

Ah, nice. So, interesting that having wildcards on both sides of the search term doesn’t work, yet adding a year and wildcard after does. It’s something to further explore later, but let’s analyze the results.

The Hash structure returned from the search engine has a few interesting fields, but what I’m interested is data retrieval. The _docs_ key points to an array of hashes, each hash being a document returned. I can easily parse that out, throw away fields I don’t care about and present the clean hash as attributes to my upstream object. Now that the low level is working nicely, that part will be easy.

It looks like I may be even able to drop MapReduce. Well, it did, until I came across these tidbits:

> Currently, the wildcard must come at the end of the term in both 
cases. (Basho [wiki.basho.com/…](http://wiki.basho.com/Riak-Search---Querying.html))

> It needs a minimum of three characters though,… (Mathias Meyer, [The Riak Handbook](http://riakhandbook.com))

Well, that explains why the first search failed. OK, so the search is not as full-featured yet as I’d like, but every piece of documentation I read suggests that it can be integrated with MapReduce. Surely with the MapReduce’s power, I’ll be able to get some nice querying going.

#### Integration with MapReduce

Since this is backing a web application, I need to make it easy for users to do adhoc searching of their events. As we’ve seen, search can get me partly there, so let’s integrate with the MapReduce object to add a bit more power. What I want is grab all personal events in March with no location. Although [Sean Cribbs](https://twitter.com/seancribbs) has started work on documentation for the Riak Ruby client, it’s still early enough for the MapReduce and Search documentation not to be there. As such, let’s refer to the [search spec](https://github.com/basho/riak-ruby-client/blob/master/spec/riak/search_spec.rb).

{% codeblock code I'm guessing will work lang:ruby %}
job = Riak::MapReduce.new(Event.client)
job.search("events", "event_date:2012-03*")
job.map("function(value, keydata, arg){ var data = Riak.mapValuesJson(value)[0]; if(data.location === \"\") {return [data];} else{ return [];} }", :keep => true)
job.run

#returns
[{"category"=>"business", "event_date"=>"2012-03-13T00:00:00+00:00", "location"=>""}, {"category"=>"business", "event_date"=>"2012-03-09T00:00:00+00:00", "location"=>""}]
{% endcodeblock %}

Nice, I guessed correctly. I wasn’t completely sure, but I got it on the first try, so that’s good. The best news is that the list-keys warning was not triggered. Exactly what I wanted!

### Caveats

The biggest caveat that I can immediately percieve is the inability of the search engine to index the existing set of documents. There are ways around that, of course. If the dataset is small enough, it can be reimported after the pre-commit hook was set up. That’s what I did here. There is also a way to index the data through the client. I may explore that later, but for now, you can refer to the spec file linked above.

Another, fairly obvious caveat is that inserting records with the search enabled is going to be slower and more taxing on each node than it would be without it. Mathias suggests to benchmark the before and after and I think that’s a worthwhile suggestion. For myself, I know that the app I’m writing is a lot more read-oriented than write-oriented, so I’m willing to take that hit. I’ll still benchmark it, because it’s good practice, but I need this type of querying power.

### Conclusion

Overall, I like Riak Search. It is not hard to pick up and it’s not hard to modify existing code to leverage it. I would like if indexing was a bit more comprehensive, but it’s not a showstopper for me.

As far as putting additional pressure on each node due to the overhead of indexing, it’s a manageable thing for me now. However, Basho has introduced secondary indexes to Riak lately. Since it seems that adding the secondary index to a document is an application-level concern rather than a database-level one, it seems as if it’s a lighter load on the nodes themselves, while introducing a bit of complexity to the application. I’ll explore that next.