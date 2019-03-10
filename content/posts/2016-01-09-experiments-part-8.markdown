---
layout: post
title: "Experiments part 8"
date: 2016-01-09 23:20:30 -0500
comments: true
categories: reactjs clojure core.async sse
---

## Previously in [Experiments, part 7](https://batasrki.github.io/blog/2015/12/21/experiments-part-7),...

I switched from [Compojure](https://github.com/weavejester/compojure) to [Pedestal](https://github.com/pedestal/pedestal) in order to take advantage of its support for [Server-sent Events](https://en.wikipedia.org/wiki/Server-sent_events). I want to use that to send background updates to the client on completion of a task.

## This took a while
I expected this feature to be done very quickly. Up to now, every library I had to integrate into my app did so nearly seamlessly. It was a matter of reading the appropriate section of the library's documentation, doing a few REPL experiments, integrating the code and doing manual testing.

SSE integration did not go like that. Not at all. It's mostly my fault, because I did not fully understand how the whole feature works. However, I do have to also blame Pedestal's SSE guides. Firstly, the guides section does not align with the examples section. Secondly, their example of an SSE usage is the epitome of useless blog post code. I don't know if I have the permission to copy the code here, so I'll [link to it](https://github.com/pedestal/pedestal/blob/master/guides/documentation/service-sse.md).

The problem with this example is that this isn't how I expect *anyone* to use the SSE support. This code generates 20 events on the server and sends them one by one down to the client, one second apart. It then closes the channel. It's concise, but it doesn't help **at all**. Some of the questions I had when I looked at this code were:

1. Uh, can I have the implementation in a function in another namespace?
2. Can I use data from a `core.async` channel?
3. If so, how?

My first crack at this nearly worked. By nearly, I mean, I got the first event as expected, which was great! Submitting the second link gave me no event back in the browser. It took me a better part of 2 weeks to figure out just what the hell is the problem.

## In the beginning...
I want to try and re-construct this painful process, because I truly believe that others will get snagged on this rough edge in Pedestal. Maybe, just maybe, providing a failure case and a solution in this ~~widely~~ blog series will help someone somewhere.

Adapting the code from the guide I linked above, I end up with something roughly like this:

``` clojure
;; Add another channel to the set there already.
;; This channel will receive the link with the title applied
;; It will then forward that onto the SSE channel completing the pipeline
(def update-chan (chan))

;; Put the record on the channel after processing it
(defn update-atom []
  (go
   (let [record (<! title-chan)]
     (log/trace "Got title" (:title record))
     (log/trace "Updating the atom")
     (swap! links update-title record)
     (>! update-chan record))))

;; SSE functions as I understand them being needed

;; This one does the actual work
(defn stream-link-updates [channel]
  (go
   (let [record (<! update-chan)]
     (>! channel {:name "link-update" :data (json/write-str record)}))))

;; This one is called by the route setup below and is the inital point into the whole thing
(defn stream-ready [channel ctx]
  (stream-link-updates channel))

;; This is added to `defroutes` to set up a SSE streaming
["/stream" {:get [::stream (sse/start-event-stream stream-ready)]}]
```

This is the server-side implementation *as I understand it* and how I would like it to fit into my app. Since `core.async` allows me to set up a pipeline, I would like the SSE event to be at the end of that pipeline. I've used `go` blocks for all pipeline parts, so it makes sense to me to do so here, too.

I need to name the event, so that I can attach a JS listener to it and convert the record into JSON in order to send it to the client.

``` javascript
var evtSource = new EventSource("/stream");
evtSource.onmessage = function(e) {
    console.log("in onmessage");
    console.log(e);
}

evtSource.onerror = function(e) {
    console.log("in onerror");
    console.log(e);
}

evtSource.addEventListener("link-update", function(e) {
    console.log("in event listener");
    console.log(e);
    console.log(JSON.parse(e.data))
});

```

I'm keeping the JS side extremely simple for now. All I want to see is an event in the console log every time I submit a link.

With all that set up, I fire up the REPL and start the app server.

```
lein repl
```

``` clojure
blog-post-app.server=>(run-dev)
```

I visit `http://localhost:8080`, post a test link and I get a result! Whoop, there it is. I post another, nothing.

![what the hell](/images/wth.png)

What is going on? How can I one work, but not another? I reload the browser and in the console, I see the expected event. This tells me that the record was put to `update-chan`, but it never got picked up on the other side and sent to the browser. In the REPL log output, I notice this exception:

``` clojure
ERROR i.p.http.impl.servlet-interceptor - {:line 105, :msg "An error occured when async writing to the client", :throwable #error {
 :cause "Broken pipe"
 :via
 [{:type org.eclipse.jetty.io.EofException
   :message nil
   :at [org.eclipse.jetty.io.ChannelEndPoint flush "ChannelEndPoint.java" 200]}
  {:type java.io.IOException
   :message "Broken pipe"
   :at [sun.nio.ch.FileDispatcherImpl write0 "FileDispatcherImpl.java" -2]}]
```

Broken pipe means that a connection got shut down uncleanly on one side of it. That does not match my understanding of things. Pedestal is supposed to be keeping that connection open for me. It says in that guide that it sets up a heartbeat to ensure that. Why does it shut down uncleanly then?

## The quest...
I then went off to find the reason why this happened and how I can fix it. I first opened an [issue](https://github.com/pedestal/pedestal/issues/384) in Pedestal's Github repo, even though I didn't ever believe that it was a bug. I guess this is the accepted norm in the Ruby community and I got used to it. One of the maintainers was polite, helped me out a bit with the code and pointed me to the mailing list. I would like to note, though, that the first thing he says I should be aware of isn't supported by the guides. In the guide, the `stream-ready-fn` clearly sets up a function that sends events. I have not seen *any* counter-examples, but maybe my searching skills aren't up to par.

The crucial thing in that explanation is that I need a `go-loop` function rather than a `go` one. OK, that's easy to fix.

``` clojure
(defn stream-link-updates [channel]
  (go-loop []
   (let [record (<! update-chan)]
     (>! channel {:name "link-update" :data (json/write-str record)}))))

```

I add `go-loop` to the list of functions referred to from `core.async`, reload the server, run the same test process, and I get the same result. This time, though, I get the broken pipe error almost immediately after the successful SSE message.

I post to the [mailing list next](https://groups.google.com/forum/#!topic/pedestal-users/J8oj33cg_e0), as suggested by the maintainer. I've yet to receive a response there. I then turn to the [Clojurians Slack channel](https://clojurians.slack.com). People here are awesome and I am grateful for their help and understanding.

Jonah Benton (I can't find his Twitter nor Github profiles) helped me out tremendously. He provided me with a real clue as to what was going on.

## The solution...
This goes back to me not fully understanding nor reading through Pedestal documentation. Jonah said off-handedly that Pedestal's interceptors always take a parameter when they're being called. SSE was just another interceptor and I should have my functions return either the event channel or context all the way back to the original `stream-ready` function. This is exactly what I ended up doing.

``` clojure
;; Loop, but always return the event channel that was passed in
(defn stream-link-updates [channel]
  (loop []
    (let [record (<!! update-chan)]
      (>!! channel {:name "link-update" :data (json/write-str record)})
      (recur)))
  channel)
```

Actually, I ended up doing a hybrid of two advices. I return the channel as advised by Jonah, but I also make sure that I set up an infinite loop, because I don't know when the next update will come. As I expected initially, this makes the whole thing work.

![yay](/images/yay.png)

## Reflection...
It took me quite a few nights of frustration to get to this point. I should have read through Pedestal's docs. In my defence, though the word "interceptor" is mentioned, there were no links to the Interceptors documentation. Was I expected to read *all* the documenation provided? I guess so.

The mailing list seems dead, to be honest. I don't know how many views there are on my post right now, but 0 responses in 3 weeks does not look well for that particular medium. If you are interested in Clojure, though, the Clojurians Slack channel is the place to be.

I am happy that I got the implementation working as I originally envisioned it. I still need to handle errors and different content types (like, what happens if I give it a PDF link?), but next, I will get ReactJS to...well, react to the incoming SSE message. I hope that this will less fun than what I had on the server.

Until the next blog post.
