---
layout: post
title: "Implementing a queue using an array"
date: 2016-07-05 21:48:39 -0400
comments: true
categories: algorithms
---

Up tonight, I'm going to write a queue implementation using a vector in Clojure and a fixed-size array in Go. The reason for the latter is that the [Data Structures](https://www.coursera.org/learn/data-structures/) course on Coursera shows an interesting way of making a queue work when the backing data structure is of fixed size.

Essentially, what I will do is implement a circular buffer. As items are popped off the front of the queue to be worked on, I'll have two pointers that will wrap around to the beginning of the array as needed.

Before I get ahead of myself, though, I'm going to do the easy thing first and write a simple implementation in Clojure.

## Queue in Clojure with a vector

As I said in the [original post](http://batasrki.github.io/blog/2016/06/23/stacks-and-queues/), Clojure has a `list` data structure that is optimized for pushing items to the front of it and a `vector` data structure whose optimization is for pushing to the back of it. The latter one sounds perfect for a queue implementation.

As per usual, I'm going to store the state in an `atom`.

``` clojure
(def my-queue (atom []))

(defn enqueue [item queue]
  (swap! queue conj item))

(defn dequeue [queue]
  (let [item (first @queue)]
    (reset! queue (rest @queue))
    item))

(defn empty*? [queue]
  (= [] @queue))
```

Here's the usage of it.

``` clojure
user> (use 'tester.array-queue :reload)
nil
user> my-queue
#atom[[] 0x1b9ca292]
user> (enqueue 2 my-queue)
[2]
user> (enqueue 21 my-queue)
[2 21]
user> (enqueue 1 my-queue)
[2 21 1]
user> (empty*? my-queue)
false
user> (dequeue my-queue)
2
user> (dequeue my-queue)
21
user> (dequeue my-queue)
1
user> (empty*? my-queue)
true
```

The implementation is straightforward, eased by the tools built into Clojure. `conj` pushes to the back of a `vector`. Using an `atom` and its interface (`swap!` and `reset!`) allows me to easily pop the item off the queue and return it. Making new things in Clojure using the existing things is as simple and enjoyable as advertised.

## Queue in Go using a fixed-size array

Now, I'm going to up the challenge a bit. Go is one of few languages intended to replace C. Well, at least, that's how I look at it. It doesn't have the niceties of other languages similar in age. There are no generics, for example, so there is no generalized data structure interface like there is in Clojure.

There *is*, however, a nice(-ish) implementation of an array. I'm going to try using that to create a queue.

### enqueue first

Go seems to me to be a verbose language, not due to being overly ceremonious like Java, but because it seems as if it's a DIY language. It gives you some basic stuff, but the rest is up to you. Some like that. I don't know if I do.

Since I have nearly 0 experience in it, my implementation is likely to be circumlocutory (LOL, thesaurus). I'm going to split it into two sections: `enqueue` and `dequeue`. These are implemented as methods on a `struct`, so that I can encapsulate the computation of the `readIdx` and `writeIdx`. It's kind of OOP in its approach.

``` go
type ArrayQueue struct {
  readIdx  int
  writeIdx int
  buffer   [4]int
}

func (aq *ArrayQueue) enqueue(item int) error {
  newWriteIdx := -1
  aq.buffer[aq.writeIdx] = item

  if aq.writeIdx == len(aq.buffer)-1 {
    newWriteIdx = 0
  } else {
    newWriteIdx = aq.writeIdx + 1
  }

  aq.writeIdx = newWriteIdx

  if newWriteIdx == aq.readIdx {
    return errors.New("Queue is full")
  }

  return nil
}

```

The interesting part here is keeping track of the `writeIdx` value. Since I'm using a fixed size array, I need to detect when I've moved the `writeIdx` past the end of the array and reset it to the head. I also need to detect when I've ran out of space in the array, so that I don't overwrite items in the queue. Here's how that would look.

``` go
func main() {
  queueables := [5]int{21, 11, 1, 3, 40}
  err := interface{}(nil)

  queue := ArrayQueue{readIdx: 0, writeIdx: 0}

  for i := 0; i < len(queueables); i++ {
    err = queue.enqueue(queueables[i])

    if err != nil {
      break
    }
  }
  fmt.Println(queue.readIdx, queue.writeIdx, queue.buffer, err)
}
```

The `queueables` slice is there just so that I can easily add items to the queue. One o the idioms in Go is to return an error type from a call. I am using that to print out the error when I've run out of space in my queue.

The `fmt.Println` outputs things nicely, `0 3 [21 11 1 3] Queue is full`. Number 40 is not added to the queue, since there is no space left.

### on to dequeue

The above test will only ever enqueue and it'll run out of space quickly. For a queue to be useful, it needs to have items removed, as well. Removing items from a queue needs to update the `readIdx` pointer value. Having this value be equal to `writeIdx` would mean that the queue is empty.

``` go
func (aq *ArrayQueue) dequeue() int {
  item := aq.buffer[aq.readIdx]

  if aq.readIdx == len(aq.buffer)-1 {
    aq.readIdx = 0
  } else {
    aq.readIdx = aq.readIdx + 1
  }
  return item
}
```

The item is fetched using the existing `readIdx` value, then calculations are made to ensure that the `readIdx` also wraps around the end of the array (slice, whatever). This makes implementing `peek()` and `empty()` easy.

``` go
func (aq *ArrayQueue) peek() int {
  if aq.empty() {
    return -1
  }

  return aq.buffer[aq.readIdx]
}

func (aq *ArrayQueue) empty() bool {
  return aq.readIdx == aq.writeIdx
}

```

The full test output then looks like this:

```
$ go build stack_array.go && ./stack_array

# initial queuing
0 0 [21 11 1 3] Queue is full

# first dequeue
21

# writeIdx wrapped around so 40 is now in the first array slot
# readIdx points to next item, 11
[40 11 1 3]

# dequeue
11

# dequeue
1

# dequeue
3

# readIdx wraps around and dequeues last item
40

# peek sees nothing, since queue is empty
-1 true
```

## Challenges and conclusion

It's fairly simple and easy to implement queues and stacks using dynamically-sized backends. Well, at least, it's simple and easy to do the basics. The backend scales up and down as needed and the programmer is left with focusing on the behaviour of each data structure. The tradeoff is that queues and stacks that use dynamically-sized backends (like a linked list) are essentially unbounded. In the worst case scenario, this could cause OoM (Out of Memory) errors on the machine running the implementation.

Writing a basic bounded queue implementation using a circular array has been more challenging. It made me appreciate not just the complexities around keeping track of read and write indexes, but also API design. As the client programmer, I wouldn't want to keep track of that. All I want is to push items onto a queue, remove them from it and check its state. This has been an illuminating exercise.

Onto trees!
