---
title: "Periodic Tasks in Elixir, Conclusion"
date: 2019-03-17T21:46:23-04:00
draft: true
omit_header_text: true
featured_image: 'images/periodic_task.png'
---

Following on from the [last post](https://batasrki.github.io/2019-03-12-periodic-tasks-in-elixir), I needed a way to kick off the task of periodically checking for new entries in Redis and adding them to Postgres.

## Scheduling tasks
There are quite a few libraries to do this. There are also [`Task`s](https://hexdocs.pm/elixir/Task.html) and [`Agent`s](https://hexdocs.pm/elixir/Agent.html). None of these really seem to suit my problem definition.

I don't need state other than the initial interval and the Redis key, so there's no need for an `Agent`. A `Task` seems to be a way to run one-off things, which is sort of what I want. However, I need this to be a recurring task. I would also prefer not to have to add a dependency just to accomplish this simple thing.

### GenServer
In Elixir's arsenal, there is a third thing that could work, the venerable `GenServer`. I can add the `GenServer` to my application supervision tree, kick it off with initial values and just let it run in the background.

Firstly, in my `application.ex` file, I'll add the spec for the new process.

```elixir
# application.ex
# other things are here also
worker(Links.PeriodicImporter, [[interval: 1_440_000, key: "redis:key"]])
```

This will start the process, pass in a value of 1,440,000 ms which is equal to 24 hours, the interval at which I want to check Redis for new content. It also passes in the Redis key, so this `GenServer` can be configurable.

```elixir
defmodule Links.PeriodicImporter do
  use GenServer

  def start_link(opts) do
    {:ok, pid} = GenServer.start_link(__MODULE__, opts)
    :timer.apply_interval(opts[:interval], __MODULE__, :perform, [pid])
    {:ok, pid}
  end

  def init(opts) do
    {:ok, opts}
  end

  def perform(pid) do
    GenServer.cast(pid, :perform)
  end

  def handle_cast(:perform, opts) do
    last_added_at_record =
      hd(Links.Repo.by_last_added_at(%{sort_direction: :desc, per_page: 1, page: 1}))

    Links.PostgresImporter.import(opts[:key], last_added_at_record.added_at)
    {:noreply, :ok, opts}
  end
end
```

The implementation is a bit wordy, but it's mostly boilerplate `GenServer` code. The interesting bit is calling `:timer.apply_interval` to essentially sleep the process before invoking the `perform` function.

The `handle_cast` contains all the business logic. I want the last record in the table and I want its `added_at` timestamp value. I can then pass that into the `import` function and let it do its work.

It's a pretty simple setup and it solves the problem.
