---
layout: post
title: "Deletion on server-side"
date: 2016-04-10 21:23:15 -0400
comments: true
categories: clojure reactjs
---

Previously, I wrote about how to delete a link from the list of links client-side in the new ReactJS version of my bookmarking app. Tonight, I will finish off that feature by enabling the deletion on the server.

## First, a confession

It took me nearly a month to write the follow-up post for a few reasons, but the chief problem was this piece of code:

``` clojure
(defn destroy-link [request]
   (let [id (get-in request [:path-params :id])]
    (reset! links (remove #(= id (:id %)) @links))
    ;; one more line below
)
```

This looks straightforward, and it should be. Get an ID from the path parameters, match it to the ID key of each map in the list of maps, and remove the map whose value under the ID key matches the passed in parameter. However, this piece of code doesn't work. Worse yet, it's not reproducible in the REPL, if you try to substitute the parameter with its value.

## Dumb mistakes

The reason the above code doesn't work is that the path parameter is a string, not an integer. `(= "1" 1)` returns false and I don't know how I feel about that. It is a dumb assumption on my part that either Clojure or Pedestal will do conversion for me or maybe it's my lack of knowledge about the framework. In either case, the fix is simple.

``` clojure
(defn destroy-link [request]
  (let [id (get-in request [:path-params :id])]
    (reset! links (remove #(= (Integer/parseInt id) (:id %)) @links))
    (ring-resp/response (json/write-str @links))))
```

Casting the string to an integer makes the `remove` function work correctly, which then allows me to set a new state for the `links` atom.

## Moving on

Now that the first item on the previous post's TODO is done, I can move on with my life.

## TODO

1. Find a cleaner way to propagate the event to the root component
2. Edit the title field of each link