---
layout: post
title: "Experiments part 5"
date: 2015-12-19 18:34:43 -0500
comments: true
categories: reactjs clojure core.async
---

## Previously in [Experiments, part 4](https://batasrki.github.io/blog/2015/12/15/experiments-part-4),...

I added a form for creating new links. Instead of introducing a database, the links are being added to an `atom`.

## In tonight's installment...

I am going to start working the trickier parts of bookmarking links, retrieving a title of a link and possibly the body. At some point, I'll have to commit to a database, but I can push that off for at least one more post. I would like to have a background queue-like functionality without introducing an actual background queue, and I think [core.async](https://clojure.github.io/core.async/) can help me do that

## Onward, then...

What I would like to do first is fetch the title of the posted URL. This will enable me to have a nice description of the link I saved. I need a library to which I can give a URL and have it fetch the HTML of the page. It would be nice if it could also parse that HTML. The most popular library in Clojure that does this is called [Enlive](https://github.com/cgrand/enlive). 

As usual, I'm going to add the library to the `project.clj` file and require it in the `handler.clj`.

``` clojure
;; in project.clj, in the :dependencies vector
[enlive "1.1.5"]

;; in handler.clj
(:require [net.cgrand.enlive-html :as html])
```

With that done, I need to add a few methods to fetch and parse HTML. 

``` clojure
(defn fetch-url [url]
  (with-open [inputstream (-> (java.net.URL. url)
                              .openConnection
                              (doto (.setRequestProperty "User-Agent"
                                                         "ReadLaterCrawler/1.0 ..."))
                              .getContent)]
    (html/html-resource inputstream)))
```

This is the first function where Clojure's interop with Java starts to show. Using the Java's URL library to get open the connection, the code threads that connection through a few things ending with the `getContent` function. Enlive then turns that resource into a map that I can iterate over. A quick REPL test shows me how this looks.

![enlive repl](/images/enlive-repl.png)

OK, I have a map representing the HTML, I need to get at the title.

``` clojure
(defn get-title [url]
  (first (map html/text (html/select (fetch-url url) [:title]))))
```

Enlive lets me select the element based on a vector I pass in. Since I all I want is a title, it's a simple vector being passed in. `html/select` takes the map of HTML and the selector vector and gives me back a map representing the selected thing.

![enlive repl 2](/images/enlive-repl-2.png)

I then extract the content and make it the return value.

![enlive repl 3](/images/enlive-repl-3.png)

### Integration

Now that I have that working, I am going to do a simple integration into the current flow. POSTing the new link will block until the title is extracted, which is what `core.async` will solve.

``` clojure
;; in POST
(POST "/api/links" request
  (let [next-id (inc (apply max (map #(:id %) @links)))
        new-link (:body request)
        new-link-with-fields (assoc new-link :id next-id :title (get-title (:url new-link)) :created_at "2015-12-15")]
    (swap! links conj new-link-with-fields)
    (json/write-str @links)))
```

The change I made is in `assoc`, where instead of `:title (:url new-link)`, I now have `:title (get-title (:url new-link))`. This should block while getting the title.

![sync save](/images/sync-save.gif)

The time it takes for the new link to show up varies based on the response time of the website, so while it can be quick some of the time, other times it might take a while.

That's it for now. In the next post, I am going to make that fetching asynchronous, which may take a bit of rejiggering of things.

Until then.
