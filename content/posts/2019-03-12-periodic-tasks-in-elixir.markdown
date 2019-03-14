---
title: "Periodic Tasks in Elixir"
date: 2019-03-13T20:07:38-04:00
omit_header_text: true
featured_image: 'images/periodic_task.png'
---

Tonight, I'll attempt to set up a Task that runs on some schedule and have it populate the Postgres database.

## Quick background
I built myself a [link-saving site](https://s2dd.ca) in Ruby, using Sinatra and fully backed by Redis. The site runs fine and I have a tonne of content saved. Amusingly for me, this content mirrors my career progression from the time I built the site until now. In spirit of learning Elixir, I'm rebuilding the site with it and backed by Postgres. This will allow me to more easily build some needed features going forward.

## Periodic task
However, tonight all I need is a way to move the saved links from Redis to Postgres. This way, I can have my regular site up and running and being used, while I build out the set of features I want before I deem the site operational.

The easy way to do this is, of course, just to copy out the data manually and use the `iex` REPL to load it into Postgres. But, this is my project to learn things and that means I get to over-engineer solutions! So, instead, I am going to dig into the support for running a one-off task and figuring out how to schedule its run.

### Redis extraction
Firstly, I need to write a function to query Redis for a list of records. I store the links in a sorted set, keyed by the Unix timestamp of the insertion time. So, to pull only the latest links, I need to pass in a Unix timestamp of the last exported link and map over the results to get a list of `Map`s.

``` elixir
def list(from_timestamp) do
  Process.whereis(:registered_redis_connection)
  |> Exredis.Api.zrangebyscore("key", from_timestamp, "+inf")
  |> Enum.map(fn(item) -> Poison.Parser.parse!(item) end)
end
```

Next, I'm going to build a module that utilizes this code. The module needs to read the latest record from the existing table, convert the timestamp to a Unix format, invoke the above code, and then create new records in Postgres.

``` elixir
defmodule Links.PostgresImporter do
  def convert_timestamp(nil) do
    "-inf"
  end

  def convert_timestamp(timestamp) do
    DateTime.to_unix(timestamp)
  end

  def fetch_redis_records(key, from_timestamp) do
    Links.RedisRepo.list_recent(key, convert_timestamp(from_timestamp))
  end
end

```

The `convert_timestamp` functions probably should be private, but I'm keeping them public for now to keep them testable. Adding to this code is a way to create Postgres records and it will all be tied together by a function that will stitch these functions together with `|>`.

```elixir
def persist_records(redis_records) do
  Links.Repo.batch_save!(redis_records)
end
```

In the DB layer, I'm using the [Moebius library](https://hexdocs.pm/moebius/). It provides a nice bulk insert API that makes this operation transactionally safe as well as a one-liner. Now, it's time to stitch it all together.

```elixir
def import(key, from_timestamp) do
  fetch_redis_records(key, from_timestamp)
  |> persist_records()
end
```

Nice and neat way of importing Redis data into Postgres. With that sorted, onto the next task!
