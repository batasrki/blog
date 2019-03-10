---
layout: post
title: "Stacks and queues, part 1"
date: 2016-06-23 20:11:26 -0400
comments: true
categories: algorithms
---

I'm auditing the [Data Structures](https://www.coursera.org/learn/data-structures/) course on Coursera. Auditing means I'm taking it for free and not paying them like they want me to. This also means I can't submit solutions to quizzes. Instead, I will try to write things up here as I learn them.

First up, I'd like to attempt to implement a stack in Clojure and Ruby using an array as the base data structure.

## Stack using an list in Clojure and an array in Ruby

A stack is known as a `Last In, First Out` data structure. The simplest visualization of a stack are dinner plates...well, stacked on top one another. You can't take and use the bottom plate without removing all the plates on top of it. It's a simple data structure, really. The API for a stack is typically the following functions:

* push (add an item to the front)
* pop (remove and return the top item)
* top (return the top item without removing it)
* empty? (is the stack empty?)

### Clojure implementation

My appreciation of Clojure's coolness is well-documented on this blog. This appreciation continues here. There are a few sequence-like data structures in Clojure, such as a `list`, `vector`, and `map`. For a while, I was confused as to the need for both a `list` and a `vector`. Tonight, though, the difference is clear and useful. A `list` is optimized for pushing to the front of it. A `vector` is optimized for pushing to the rear of it. This makes writing code for this blog post that much easier.

I'll use the `list` as the basis for my stack.

``` clojure
(def my-stack (atom '()))

(defn push* [item stack]
  (swap! stack conj item))

(defn top* [stack]
  (first @stack))

(defn pop* [stack]
  (let [item (top* stack)]
    (reset! stack (rest @stack))
    item))

(defn empty*? [stack]
  (= 0 (count @stack)))
```

I'm sticking the list in an atom in order to keep state around. I'm also adding `*` to the function names, so as not to clobber the built-in Clojure functions.

`push*`, `top*` and `empty*?` are straightforward to implement. Pushing onto the stack involves using `conj` on the `list` data structure which will add things to the front of the list. Returning the top element without removing it is also simple, as is comparing the number of items in the list to 0, which is our empty check.

`pop*` is a little trickier, as it's a two step process. I need to remove the item from the stack, as well as return it. I can actually reuse `top*` for the first step and then reset my atom to the rest of the list as a means of removing that first one.

Here is how the API works:

``` clojure
user> (use 'array-stack)
nil
user> @my-stack
'()
user> (push* 1 my-stack)
(1)
user> (push* 2 my-stack)
(2 1)
user> (top* my-stack)
2
user> (pop* my-stack)
2
user> @my-stack
(1)
user> (empty*? my-stack)
false
user> (pop* my-stack)
1
user> (empty*? my-stack)
true
```

Nice! As I keep saying, well-designed languages make things so easy to use. I wonder if I'll be able to do this as easily in Ruby.

### Ruby implementation

Since everything in Ruby is an object, that's what I'll start with. Now, Ruby only has a `vector` implementation, which might writing a stack implementation a bit more challenging. Luckily, the one thing I don't need to worry about is the size of the stack.

``` ruby
class MyStack
  attr_accessor :store

  def initialize
    @store = []
  end

  def push(item)
    store.insert(0, item)
  end

  def top
    store.first
  end

  def pop
    store.shift
  end

  def empty?
    store.count == 0
  end
end
```

Huh. It's actually pretty simple, as well. Well, after I dug into [Ruby's Array docs](http://docs.ruby-lang.org/en/2.0.0/Array.html), it got simple. The `insert` method is pretty nice, though I suspect it is not nice at all from a performance point of view.

Here's the API as implemented in Ruby:

``` ruby
2.3.0 :065 > stack = MyStack.new
 => #<MyStack:0x007fd2fd0e87b0 @store=[]>
2.3.0 :066 > stack.push 1
 => [1]
2.3.0 :067 > stack.push 2
 => [2, 1]
2.3.0 :068 > stack.push 3
 => [3, 2, 1]
2.3.0 :069 > stack.top
 => 3
2.3.0 :070 > stack.store
 => [3, 2, 1]
2.3.0 :071 > stack.pop
 => 3
2.3.0 :072 > stack.store
 => [2, 1]
2.3.0 :073 > stack.empty?
 => false
2.3.0 :074 > stack.pop
 => 2
2.3.0 :075 > stack.pop
 => 1
2.3.0 :076 > stack.empty?
 => true
```

## Reflect and conclude

I think this is a good stopping point. Soon _(I was going to say tomorrow night, but let's face it, these posts aren't that regular)_, I will implement a queue using the `list`/`vector` data structures. I suspect that it will be equally as easy.

The differences between language philosophies are laid bare, I hope.

In Clojure, not only did I separate the functions that operate on the data structure from itself, I also managed to reuse a function I just created in order to implement another. To me, this is one of the core principles of functional programming and of Clojure. Each function is a self-contained unit of work that operates on the parameters passed into it. I could have just assumed that the `atom` holding my list is just available, but I am already in the habit of not making that assumption.

On the other hand, my habit in Ruby is to create a class that contains the data that I need to perform computations on and the functions that do those computations. This is the most-straightforward way to do things in Ruby. It falls squarely on the language's golden path. I could, with greater effort, to do what I did in Clojure. I could have made two classes, one as the container of the underlying data and another that holds the functions to operate on it.

``` ruby
class StackContainer
  def initialize
    @store = []
  end

  def write(item)
    store.insert(0, item)
    store
  end

  def read(remove: false)
    if remove
      store.shift
    else
      store.first
    end
  end

  def empty?
    store.count == 0
  end

  private
  attr_accessor :store
end

class Stack
  def self.push(item, store)
    store.write(item)
  end

  def self.top(store)
    store.read
  end

  def self.pop(store)
    store.read(remove: true)
  end

  def self.empty?(store)
    store.empty?
  end
end
```

Usage:

``` ruby
2.3.0 :271 > store=StackContainer.new
 => #<StackContainer:0x007f9068550c08 @store=[]>
2.3.0 :272 > Stack.push(1, store)
 => [1]
2.3.0 :273 > Stack.push(10, store)
 => [10, 1]
2.3.0 :274 > Stack.empty?(store)
 => false
2.3.0 :275 > Stack.top(store)
 => 10
2.3.0 :276 > Stack.pop(store)
 => 10
2.3.0 :277 > Stack.pop(store)
 => 1
2.3.0 :278 > Stack.empty?(store)
 => true
2.3.0 :279 >
```

~~I am likely over-complicating things, but that is because this style of writing code in Ruby is totally unfamiliar to me. Also, the underlying implementation is leaking through. I probably need to abstract further, but I honestly don't know how. On further thought, the initial design looks better to me, as I only expose a very limited set of functions and I can change the underlying data structure and the implementations of the API functions. However, it's also possible that a user of my stack API will realize that it's really an array and start to depend on that.~~

EDIT: With help from my peeps at (Practicing Developer's Slack channel)[https://practicingdeveloper.com/], I actually came up with a nicer implementation that doesn't necessarily leak.

I would love to hear people's thoughts on this. Has anyone had to implement their own stack data structure? What did you use as a basis? How did it work out for you?
