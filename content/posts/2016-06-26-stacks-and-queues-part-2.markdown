---
layout: post
title: "Stacks and queues part 2"
date: 2016-06-26 21:31:26 -0400
comments: true
categories: algorithms
---

Following on from the last post, tonight I am going to try and implement the stack using a linked list instead of an array.

## Short background

As explained in the course, a linked list might be preferred over an array, because in languages like C, an array has to be of fixed size. Adding a one more element to a full stack will cause an overflow error at best and overwrite random memory addresses at worst. Each element of the linked list is dynamically allocated on the heap, which means that the stack's size is unbounded. The tradeoff of using a linked list is increased memory size. The memory structure of each stack item needs to hold a pointer to the next item in the list, as well as the stored value.

## Ruby implementation first

Ruby is a language I have far more experience in, so I'm going to start there. I'm going to reuse the `Stack` interface from the last post.

``` ruby
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

I'm going to supplant the `store` parameter with a linked list implementation. The linked list needs to have two things, a place to store the value and a pointer to the next item in the list.

``` ruby
class LinkedListStackContainer
  def initialize
    @head = nil
  end

  def write(item)
    node = Node.new(item, head)
    @head = node
  end

  def read(remove: false)
    if remove
      first_node = @head
      @head = @head.next
      first_node.value
    else
      @head.value
    end
  end

  def empty?
    head.nil?
  end

  attr_accessor :head
end

class Node
  attr_accessor :value

  def initialize(item, next_node)
    @value = item
    @next_node = next_node
  end

  def next
    next_node
  end

  private
  attr_accessor :next_node
end
```

The implementation of a linked list in an object-oriented language is straightforward. You need a `Node` that holds a `value` and a pointer to the next `Node`. A stack on top of a linked list consists of the following steps:

- Add a new `Node`
- Set its `next_node` pointer to the current value of `head`
- Set `head` to point to the new `Node`

With these steps completed, `top` and `pop` mean reading the value of `head` and setting `head` to the next node in the list, respectively. An empty check is just checking that `head` points to nothing.

Here's how that looks like:

``` ruby
2.3.0 :738 > store = LinkedListStackContainer.new
 => #<LinkedListStackContainer:0x007f9067453838 @head=nil>
2.3.0 :739 > Stack.push(1, store)
 => #<Node:0x007f906744b9d0 @value=1, @next_node=nil>
2.3.0 :740 > Stack.push(2, store)
 => #<Node:0x007f9067443b40 @value=2, @next_node=#<Node:0x007f906744b9d0 @value=1, @next_node=nil>>
2.3.0 :741 > Stack.top(store)
 => 2
2.3.0 :743 > Stack.empty?(store)
 => false
2.3.0 :744 > Stack.pop(store)
 => 2
2.3.0 :745 > Stack.pop(store)
 => 1
2.3.0 :746 > Stack.empty?(store)
 => true
```

## Clojure implementation

I suspect that writing a linked list in Clojure will be just as awkward as trying to hide the array as the implementation detail in Ruby. I am going to try anyway.

Clojure has a few macros that allow programmers to write something like OO code. I'm talking about [defprotocol](https://clojuredocs.org/clojure.core/defprotocol) and [defrecord](https://clojuredocs.org/clojure.core/defrecord). I'm going to try using those for my experiment.

Firstly, I'll define a record to hold my data. The data held is the same, a `value` and a `next-node` pointer.

``` clojure
(defrecord StackNode [val next-node])
```

Then, I'll define functions that manipulate the stack. I need a helper function to resolve the top of the stack, as well as the expected ones.

``` clojure
(defn head* [stack]
  (cond
    (nil? (:val stack)) nil
    :else stack))

(defn push* [item stack]
  (cond
    (nil? @stack) (reset! stack (StackNode. item nil))
    :else (reset! stack (StackNode. item (head* @stack)))))

(defn top* [stack]
  (:val @stack))

(defn pop* [stack]
  (let [val (:val @stack)]
    (reset! stack (:next-node @stack))
    val))

(defn empty*? [stack]
  (nil? (top* stack)))

```

I'm storing the current state in an `atom`, like before, which is there are `reset!` calls in the `push*` and `pop*` functions. The logic for `push*` is fairly simple. If the `head*` returns `nil`, it means we don't have anything on the stack, so create a new instance of the defined record. If there is something there, create a new instance of the defined record and set its `next-node` value to the old instance.

`pop*` does nearly the opposite. It saves the value of the current instance, then resets the atom such that it points to the next instance, effectively discarding the current one.

`empty*?` and `top*` should be self-explanatory. I wanted to use `defprotocol` to define the `read`, `write`, `empty?` methods as in the Ruby version, but I still don't fully understand how that works. Here is my desired implementation, so maybe someone can point me in the right direction.

``` clojure
(defprotocol MyStack
  (read* [stack & args])
  (write* [item stack])
  (empty? [stack]))

(extend-type clojure.lang.Atom
  MyStack
  (read* [stack & {:keys [remove] :or {remove false}}]
    (if remove
      (pop* stack)
      (top* stack)))
  (write* [item stack]
    (push* item stack))
  (empty? [stack]
    (empty*? stack)))

(read* my-stack)
(read* my-stack {:remove true})
```

That's it for tonight. Next up, I'll try on the queues for a size.