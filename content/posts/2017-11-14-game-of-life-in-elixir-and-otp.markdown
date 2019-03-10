---
layout: post
title: "Game of Life in Elixir and OTP"
date: 2017-11-14T23:51:27-05:00
categories: concurrency elixir otp
---

I'm back! During my time away from the blog, I've been working hard and (half-)taking various Coursera and EdX courses. The [latest one](https://courses.edx.org/courses/course-v1:KTHx+ID2203.1x+3T_2017/course/) is the inspiration for this blog post.

The course wants me to use Scala and Kompics (course facilitators' distributed framework thing). I really don't have an interest in Scala, and I'm auditing the course so I can't submit work anyway. I thought it'd be interesting to use Elixir and OTP to solve the problems.

## Game of Life

I have never had a satisfactory solution to this programming challenge, nor did I have a motivation to come up with one. Writing about it, though, may provide that motivation.

The rules are simple. Each cell in a grid has up to 8 neighbours. Each cell's state is determined using the following:

1. Any live cell with fewer than two live neighbours dies, as if caused by underpopulation.
2. Any live cell with two or three live neighbours lives on to the next generation.
3. Any live cell with more than three live neighbours dies, as if by overpopulation.
4. Any dead cell with exactly three live neighbours becomes a live cell, as if by reproduction.

The idea is to seed the entire grid (heretofore known as _world_) with an initial state, set up a transition cycle (say every 10 seconds) and let the world evolve from this initial state. Each subsequent generation is a pure function of the previous generation's state through the application of rules.

![Functional Superman](/images/sounds_like_a_job_for_a_functional_language.jpg)

This sounds like an appropriate use case for a functional language like Elixir.

## The plan

Firstly, I will implement a naive version of this program. Cell state will be managed manually and generation of new state of the world will be triggered manually.

Then, I'll explore OTP and how it can help with parts of the implementation. For example, I've heard of `gen_fsm`, the state machine OTP behaviour. It sounds like it could help with state transitions. Furthermore, I will want to explore how to parallelize the generation of the new state. Each cell acts on the information available to it at the time of transition _and only on that information_. That sounds to me like an [embarrassingly parallel](https://en.wikipedia.org/wiki/Embarrassingly_parallel) problem.

## Naive implementation

Let's start from the bottom, with a module that defines a single cell.

```
defmodule GameOfLife.Cell do
  alias __MODULE__

  @states [:alive, :dead]
  defstruct [:state]

  def new() do
    %Cell{state: Enum.random(@states)}
  end

  def toggle(%Cell{state: :alive}) do
    %Cell{state: :dead}
  end

  def toggle(%Cell{state: :dead}) do
    %Cell{state: :alive}
  end

end
```

A Cell can have 2 states, `alive` and `dead`. When a new Cell is generated, its state is randomly picked from one of the two defined. This allows the progam to have a randomized initial state of the world.

As is Erlang and Elixir custom, there's a single method provided, with an overloaded definition. Which definition runs depends on the pattern matching on the Cell's state when it's passed in. The `toggle` method then returns a new Cell with its state set.

This allows me to set up a board. The board is just a two-dimensional array of `Cell`s. 

```
defmodule GameOfLife.World do
  alias __MODULE__

  defstruct [:board]

  def new() do
    # generate a 10x10 matrix of initial state cells
    %World{board: Enum.map(Range.new(0,10), fn(x) -> Enum.map(Range.new(0,10), fn(x) -> GameOfLife.Cell.new end) end)}
  end
end

```

Since the `Cell` initializer picks a state at random, the easy thing to do is to loop over the range and add a cell to the board.

I'll also add a display module so that I can see the progression of the board state over time.

```
defmodule GameOfLife.Display do
  alias GameOfLife.{Cell, World}

  def print(%Cell{} = cell) do
    case cell.state do
      :alive ->
        "X"
      :dead ->
        " "
    end
  end

  def print(%World{} = world) do
    IO.puts print_board(world.board)
  end

  defp print_row(row) do
    Enum.join(
      Enum.map(row, fn(cell) -> print(cell) end),
      ""
    )
  end

  defp print_board(board) do
    Enum.join(
      Enum.map(board, fn(row) -> print_row(row) end),
      "\n"
    )
  end
end

```

For each row and each cell in the row, the code inspects the cell's state and emits either an "X" or " " (single whitespace). It then joins each cell in a row to the rest of it and joins rows with a new line, visually creating a board.

Here's how it looks like,

```
iex(34)> Display.print(World.new)
    X XXXX
X     XX  X
XXX  X X
XX   X  X
 X XXXX XXX
XXX X  X  X
    X X  X
  XX XX X X
XXXX X    X
 XXX X XXX
   XX    X
:ok
```

Cool! Running this code more than once gives me different initial board configurations. Now that the basics are set up, the next post will cover handling the logic to determine the state of the cell based on its neighbours.

Toodles.
