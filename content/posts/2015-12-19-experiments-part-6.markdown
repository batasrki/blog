---
layout: post
title: "Experiments part 6"
date: 2015-12-19 20:23:01 -0500
comments: true
categories: reactjs clojure core.async
---

## Previously in [Experiments, part 5](https://batasrki.github.io/blog/2015/12/19/experiments-part-5),...

I added a synchronous way of fetching a title from an HTML page, using Enlive.

## Let's async this thing

OK, I want to get the quick response back, but still keep this new functionality. This is where I introduce `core.async`.

``` clojure
;; in project.clj
[org.clojure/core.async "0.2.374"]

;; in handler.clj's require
(:require [clojure.core.async :as async :refer [>! < ! chan go]])
```

This library is based on concepts presented in a book called [Communicating Sequential Processes](http://www.usingcsp.com/cspbook.pdf). The core premise is that certain types of tasks can be handed off to a different process/thread, much like how web developers use background queues to do tasks like sending e-mail, etc. The beautiful thing about Clojure's implementation is that it's just a set of macros around core language features. That book is really worth reading. I am saying that more to myself than anyone else reading this.

Using this library, I am going to set up a pipeline that does the synchronous fetch-and-parse in an asynchronous way. For that, I need to set up 2 channels and a few functions that put values and take values from the channel and update things in the background.

``` clojure
(def request-chan (chan))
(def title-chan (chan))

(defn update-title [data record]
  (map #(if (= (:id record) (:id %))
          (assoc % :title (:title record))
          %) data))

(defn update-atom []
  (go
   (let [record (< ! title-chan)]
     (println "Got title" (:title record))
     (println "Updating the atom")
     (swap! links update-title record))))

(defn async-get-title []
  (go
   (let [record (< ! request-chan)]
     (println "Got the request, processing....")
     (>! title-chan (assoc record :title (get-title (:url record)))))))

(defn async-request-title [new-link]
  (go
   (println "Putting the request on channel")
   (>! request-chan new-link)))
```

Reading from the bottom up, the code does the following steps.

1. Put the newly created link record to the `request-chan` channel
2. Take the record off the `request-chan` channel, fetch the title from the HTML page, put the update onto `title-chan` channel
3. Take the updated record of `title-chan` channel, update the `links` atom.

Because the update is done in the background, I won't see the effect until server reload, but that's fine for now. I have plans for that (did someone say [Server-sent Events](https://en.wikipedia.org/wiki/Server-sent_events)). Programming can be so much fun, sometimes.

I want to explain the `update-title` function before moving on, though. Here's the source for it, again.

``` clojure
(defn update-title [data record]
  (map #(if (= (:id record) (:id %))
          (assoc % :title (:title record))
          %) data))

;; called with
;; (swap! links update-title record)
```

The function loops over the list of links, which are represented as hashmaps. If the `id` of the current item is equal to the one we want to update, we update the `title` value. If not, we keep just copy the link map into the new data set. I admit, this is a lot of rigmarole, but again, it's fun to try not use a database as a crutch. Once I do commit to having a database, this code will likely go away.

## Integrating the new way

Now that there's a way for me to update the atom, I need to integrate that into the flow of a web request.

``` clojure
(GET "/" []
  (async-get-title)
  (update-atom)
  (views/index))

(POST "/api/links" request
  (let [next-id (inc (apply max (map #(:id %) @links)))
        new-link (:body request)
        new-link-with-fields (assoc new-link :id next-id :title (:url new-link) :created_at "2015-12-15")]
    (async-request-title new-link-with-fields)
    (swap! links conj new-link-with-fields)
    (json/write-str @links)))
```

When I load the view, I call the 2 background functions, so that `go` blocks are set up. The `go` blocks are where putting and taking from channels are executed. These functions will wait for new input to come in, but since I am not interested in the return value, calling them returns immediately.

The code that creates a link is changed, as well. Firstly, the synchronous fetch is reverted back to what I had initially, where `title` is the URL. Then, just before adding the new link, a request is submitted for the title attribute of the URL. This kicks off the asynchronous pipeline.

## Incoming GIF!

This is how this looks now.

![core async magic](/images/core-async-magic.gif)

The speed of update is back, since we're just pre-pending a record to an in-memory data structure. After a reload (or 2), the title is changed, demonstrating the background work.

That's about it for now. Until the next post.
