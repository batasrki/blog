---
layout: post
title: "Experiments part 7"
date: 2015-12-21 21:52:24 -0500
comments: true
categories: reactjs clojure core.async
---

## Previously in [Experiments, part 6](https://batasrki.github.io/blog/2015/12/19/experiments-part-6),...

I used `core.async` to background a slow task, namely fetching the HTML of the saved URL and parsing out the `<title>` tag.

## Up next...

I need to do a bit of yak-shaving. As I alluded to before, I want to use [Server-sent Events](https://en.wikipedia.org/wiki/Server-sent_events), to send the above background update to the client on completion. To do that, I have to switch libraries. Up to now, I've used [Compojure](https://github.com/weavejester/compojure), but after reading up on SSE and Clojure, I have been convinced that I need to use either [Pedestal](https://github.com/pedestal/pedestal) or [yada](https://github.com/juxt/yada).

After doing a little bit of research, I feel that Pedestal will suit me better as it uses `core.async` to do all of its async things and hey, **I'm using that, too**! With that settled, I am now realizing that I need to port all of my currently written code to this new way of doing things. There doesn't seem to be a magical "take this Compojure app and make it a Pedestal app" command in Leiningen, so I am left with a few options. I think what I'll do is generate a Pedestal app, then copy over the generated bits into the current app and smush them together. I hope it works!

## Go, go, go, lein generator!

OK, in a directory above the current app directory, I need to run the Leiningen generator.

```
lein new pedestal-app prototype
```

This goes and does things and now I have a new directory. I now need to copy over bits from `project.clj`, the whole `server.clj` and `service.clj`, the `config`, `log`, and `target` directories. Then, in the new `service.clj` file, I need to copy over the code from `handler.clj`. All in all, this is how the project and service files look like.

``` clojure project.clj
(defproject blog-post-app "0.0.1"
  :description "ReactJS bookmarker backed by experimental Clojure stuff"
  :url "http://particletransporter.io"
  :min-lein-version "2.0.0"
  :dependencies [[org.clojure/clojure "1.7.0"]
                 [org.clojure/data.json "0.2.6"]
                 [org.clojure/core.async "0.2.374"]
                 ;;[compojure "1.4.0"]

                 [io.pedestal/pedestal.service "0.4.1"]
                 [io.pedestal/pedestal.jetty "0.4.1"]
                 [ch.qos.logback/logback-classic "1.1.3" :exclusions [org.slf4j/slf4j-api]]
                 [org.slf4j/jul-to-slf4j "1.7.12"]
                 [org.slf4j/jcl-over-slf4j "1.7.12"]
                 [org.slf4j/log4j-over-slf4j "1.7.12"]

                 [enlive "1.1.5"]
                 [hiccup "1.0.5"]]
  :resource-paths ["config" "resources"]
  :uberjar-name "blog-post-app.jar"
  :profiles
  {:dev {:dependencies [[io.pedestal/pedestal.service-tools "0.4.1"]
                        [cider/cider-nrepl "0.10.0"]]}
   :uberjar {:aot [blog-post-app.server]}}
  :main ^{:skip-aot true} blog-post-app.server)
```

``` clojure service.clj
(ns blog-post-app.service
  (:require [io.pedestal.http :as bootstrap]
            [io.pedestal.http.route :as route]
            [io.pedestal.http.body-params :as body-params]
            [io.pedestal.http.route.definition :refer [defroutes]]
            [io.pedestal.log :as log]
            [ring.util.response :as ring-resp]
            [clojure.data.json :as json]
            [clojure.core.async :as async :refer [>! <! go chan]]
            [net.cgrand.enlive-html :as html]
            [blog-post-app.views :as views]))

(def links (atom '({:id 1 :url "https://google.ca" :title "Google" :client "static" :created_at "2015-12-01"}
                   {:id 2 :url "https://twitter.com" :title "Twitter" :client "static" :created_at "2015-12-01"}
                   {:id 3 :url "https://github.com" :title "Github" :client "static" :created_at "2015-12-08"}
                   {:id 4 :url "https://www.shopify.ca" :title "Shopify" :client "static" :created_at "2015-12-08"}
                   {:id 5 :url "https://www.youtube.com" :title "YouTube" :client "static" :created_at "2015-12-08"})))

(defn fetch-url [url]
  (with-open [inputstream (-> (java.net.URL. url)
                              .openConnection
                              (doto (.setRequestProperty "User-Agent"
                                                         "ReadLaterCrawler/1.0 ..."))
                              .getContent)]
    (html/html-resource inputstream)))

(defn get-title [url]
  (first (map html/text (html/select (fetch-url url) [:title]))))

(def request-chan (chan))
(def title-chan (chan))

(defn update-title [data record]
  (map #(if (= (:id record) (:id %))
          (assoc % :title (:title record))
          %) data))
(defn update-atom []
  (go
   (let [record (<! title-chan)]
     (log/trace "Got title" (:title record))
     (log/trace "Updating the atom")
     (swap! links update-title record))))

(defn async-get-title []
  (go
   (let [record (<! request-chan)]
     (log/trace "Got the request, processing....")
     (>! title-chan (assoc record :title (get-title (:url record)))))))

(defn async-request-title [new-link]
  (go
   (log/trace "Putting the request on channel")
   (>! request-chan new-link)))

(defn home-page
  [request]
  (async-get-title)
  (update-atom)
  (ring-resp/response (views/index)))

(defn list-links
  [request]
  (ring-resp/response (json/write-str @links)))

(defn create-link
  [request]
  (let [next-id (inc (apply max (map #(:id %) @links)))
        new-link (:json-params request)
        new-link-with-fields (assoc new-link :id next-id :title (:url new-link) :created_at "2015-12-21")]
    (async-request-title new-link-with-fields)
    (swap! links conj new-link-with-fields)
    (ring-resp/response (json/write-str @links))))

(defroutes routes
  ;; Defines "/" and "/about" routes with their associated :get
  ;; handlers.
  ;; The interceptors defined after the verb map (e.g., {:get
  ;; home-page}
  ;; apply to / and its children (/about).
  [[["/" {:get home-page}
     ^:interceptors [(body-params/body-params) bootstrap/html-body]
     ["/about" {:get about-page}]
     ["/api/links" {:get list-links
                    :post create-link}
      ^:interceptors [bootstrap/json-body]]]]])
(def service {:env :prod
              ::bootstrap/routes routes
              ::bootstrap/resource "/public"
              ::bootstrap/type :jetty
              ::bootstrap/port 8080})
```

With that, I start the REPL and run the server from it.

``` bash
lein repl
```

``` clojure REPL
(server/start runnable-service)
```

## A proof in form of a GIF

![pedestal](/images/pedestal.gif)

That's it for now. The stage is set up for SSE, which will be the next thing I tackle. Until then.

P.S.

I deployed the app to Heroku, [here](https://cljrjs.herokuapp.com). The code can be seen [here](https://github.com/batasrki/blog-post-app).
