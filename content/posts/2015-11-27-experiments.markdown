---
layout: post
title: "Experiments, part 1"
date: 2015-11-27 19:50:25 -0500
comments: true
categories: reactjs clojure
---

## A different tack

I've been very lucky to have had steady employment using Ruby and Rails. At times, it felt like cheating. How can I have so much fun and get paid well for it?

Continuing my search for that fun high, I started looking at other technologies. I've been an admirer of Clojure for a while. Its approach to composing programs seems to suit how I think software development should be. In the same vein, ReactJS's approach to building UIs using components fits how I think UIs should be built. As such, I'm embarking on a rewrite of my bookmarking site, [Particle Transporter](http://particletransporter.io), using Clojure and ReactJS.

## Goals

The big goals for this rewrite are:

1. Have a nicer UI for the listing of links than what's on the current site
2. Implement a filter (filter by tags to start with)
3. Fetch the title of a given link in the background using core.async
4. Fetch the first paragraph (if available) of a given link in the background using core.async
5. Have an offline mode, to ease development from multiple machines

Having spent a few months at Shopify, I really started liking a UI concept called [Cards](http://semantic-ui.com/views/card.html). It seems to nicely encapsulate a piece of data. This makes it an easy choice for ReactJS, since we can take each saved link and all its data and metadata and wrap it into a card. However, I'm getting ahead of myself. Firstly, I'm going to set up the environment in which I can create this app.

##Setup

Since this will be a Clojure app, I'm going to use [Compojure](https://github.com/weavejester/compojure) as the base. Leiningen is a must, obviously, and it provides a template that generates a Compojure-flavoured skeleton.

``` bash
$ lein new compojure read-later
$ cd read-later && ls
```

Cool, the filesystem structure is there. I am going to give it a quick whirl, running `lein ring server` on the command line within the directory. A few seconds later, I see "Hello World" in a browser tab.

Now that I have a running server, I'm going to add a view that sets up the skeleton of the html page. To do that, I'll install [Hiccup](https://github.com/weavejester/hiccup). It allows me to write views in Clojure, which will then render HTML. Stopping and starting the server using `lein` will install the newly specified dependency automatically.

OK, next, I need to add the base view and add ReactJS files.

``` clojure
(ns blog-post-app.views
  (:require [hiccup.page :refer [html5 include-js include-css]]))

(defn base-page [title & body]
  (html5
   [:head
    (include-css "https://cdn.rawgit.com/twbs/bootstrap/v4-dev/dist/css/bootstrap.css")
    [:title title]]
   [:body
    [:div.container-fluid
     [:h1
      [:a {:class :brand :href "/"} "Particle Transporter Reborn"]]
     [:div {:id "content"}]]
    (include-js "https://cdnjs.cloudflare.com/ajax/libs/react/0.14.3/react.js")
    (include-js "https://cdnjs.cloudflare.com/ajax/libs/react/0.14.3/react-dom.js")
    (include-js "https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.23/browser.min.js")
    (include-js "https://ajax.googleapis.com/ajax/libs/jquery/2.1.4/jquery.min.js")
    (include-js "/js/core.js")]))

(defn index []
  (base-page
      "Saved Links - fPT"))
```

I am going to reference the views file in the default `handler.clj` file and adjust the home route to render the HTML page.

``` clojure
(ns blog-post-app.handler
  (:require [compojure.core :refer :all]
            [compojure.route :as route]
            [blog-post-app.views :as views]
            [ring.middleware.defaults :refer [wrap-defaults site-defaults]]))

(defroutes app-routes
  (GET "/" []
       (views/index))
  (route/not-found "Not Found"))

(def app
  (wrap-defaults app-routes site-defaults))
```

Refreshing the page, I see the header link and an empty page. Checking the console, I see no errors, so all the JS files have been required successfully. Onto the next step, creating a ReactJS view.

## ReactJS view

The JavaScript file is straightforward to begin with.

``` javascript
var LinkListContainer = React.createClass({
  render: function() {
    return (
      <section className="linkListContainer">
        <LinkList data = {this.state.data} />
      </section>
    );
  }
});

ReactDOM.render(<LinkListContainer url="/api/links" />, document.getElementById("content"));
```
I specified the top-level component and the child component that will do most of the work. The top-level component is given a URL from which data will be fetched and an HTML element that it should render into.

That's it for part 1. Next, I will specify the data source and render out the links.