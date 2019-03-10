---
layout: post
title: "Passing lambdas as parameters"
date: 2015-09-15 23:54:19 -0400
comments: true
categories: ruby lambda
plugins: backtick_codeblock
---

## Background

At [my current job](https://shopify.com), I have been working on merchants stores' integration with third parties. As a part of this task, I found that I needed to ensure that our app batches API calls to the third party. After doing a quick measurement, I found that batching API calls lowered the processing time by about 2x-5x when compared to doing the same calls in sequential order. However, whereas sequential processing had a clear approach of dealing with successful and failed calls, the batch call had none of that.

## Lambdas

The answer was suggested by my team lead and clarified by a co-worker. I then remembered I knew this about Ruby and I've been using a similar approach extensively when writing Clojure programs.

Much like JS, Ruby allows us to pass a `lambda` or a `Proc` into a method as a parameter. We can then call this stored function with certain parameters. In the case above, what I wanted was to partition the responses to batched API calls into successful and failed ones. I could then invoke 2 lambdas, one for each set of responses and do the appropriate thing.

## A trivial example

I don't want to share the code specific to my problem, so here's a trivialized version of the same approach:

``` ruby
for_evens = lambda { |evens| p "All even numbers in the set are: #{evens}" }

for_odds = lambda { |odds| p "All odd numbers in the set are: #{odds}" }

the_set = (1..10).to_a

def partition_set_and_invoke_lambdas(the_set, for_evens, for_odds)
  evens, odds = the_set.partition {|item| item % 2 == 0}

  for_evens.call(evens)
  for_odds.call(odds)
end

partition_set_and_invoke_lambdas(the_set, for_odds, for_evens)

"All odd numbers in the set are: [2, 4, 6, 8, 10]"
"All even numbers in the set are: [1, 3, 5, 7, 9]"
 => "All even numbers in the set are: [1, 3, 5, 7, 9]"
```

Cool, eh?