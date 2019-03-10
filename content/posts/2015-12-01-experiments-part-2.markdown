---
layout: post
title: "Experiments part 2"
date: 2015-12-01 20:25:32 -0500
comments: true
categories: clojure reactjs
---

## Previously in [Experiments](https://batasrki.github.io/blog/2015/11/27/experiments),...

I set up a Clojure backend and installed ReactJS front-end. I also created a basic skeleton, but that didn't actually show up, because JSX.

## In tonight's installment...

Firstly, I'm going to fix up the transpiling from JSX to JS, so that changes to the React front-end are picked up and shown. I'll add the expected route, but serve dummy data. Then, I'll get the front-end to consume that data.

## Onward, then...

OK, so, [ReactJS tutorial site](http://facebook.github.io/react/docs/tutorial.html) is a bit misleading. I am guessing they expect me to use straight-up HTML and add the `text/babel` script tag. However, I don't know how to do that through Hiccup. Moreover, that seems a bit weird, anyway. There's a way to do it without using the specialized tag, but it'll take a bit more effort.

Using the [Offline transform section](http://facebook.github.io/react/docs/getting-started.html#offline-transform) of the Getting Started page, I'm going to...ugh...install an NPM module _globally_ (seriously, tho?) and use it to transform my `core.js` JSX joint into a pure JavaScript file.

``` bash
$ npm install -g babel-cli babel-preset-react
$ cd resources/public/js
$ mkdir build
$ babel --presets react core.js --watch --out-dir build
```
Now that `babel` is transpiling my JSX into compatible JS, I need to adjust where I look for that JS file.

``` clojure
(include-js "/js/build/core.js")
```

Running `lein ring server` opens up a browser tab pointing to [http://localhost:3000](http://localhost:3000). As usual, I immediately open the Web Inspector tab and notice the following error: `ReferenceError: LinkList is not defined`. This is good news! It means that the JSX was successfully transpiled into JS and the browser has attempted to build a ReactJS component.

I'm going to fix the error quickly, by adding the missing component, which is just an empty table tag, for now.

``` javascript
var LinkList = React.createClass({
  render: function() {
    return (
      <table></table>
    );
  }
});
```

Cool. That brings the next error, `TypeError: this.state is null`. If you recall, that's from the following line:

``` javascript
<LinkList data = {this.state.data} />
```

It makes sense, I didn't initalise the state to anything, but I want an element of it, `data`. That's fine, I can set it to something empty. ReactJS has a handy function, called `getInitialState` that I can use to set...well, the initial state for each component.

``` javascript
// This is done on the outer component, LinkListContainer
getInitialState: function() {
  return { data: [] };
}
```

And with that, no more errors in Web Inspector console!

## An interlude

I want to stop here, for a second, and clarify something about ReactJS. ReactJS is a front-end technology. It only cares about consuming data. As such, the JavaScript snippets of code I am presenting can be used with _any_ web framework.

This is one of the things that I like about it and its competitors. However, unlike BackboneJS (though, I admit, it's been a long while since I've used it) and AngularJS, giving ReactJS data to consume is very easy. Insert your favourite "Facebook is a data consuming monster" joke here.

## Continuing....

Back in Clojure-land, I'm going to get the `api/links` route working. Initially, it'll just deliver a static list of links, each link being a hashmap. The nice thing about Clojure and its libraries is consistency of data structures. Querying a database returns a hashmap representation of a record. That representation is easily turned into a JSON response, which is itself a hashmap!

``` clojure
;; in require, need to add a JSON library
(:require [clojure.data.json :as json])

;; appending a route to defroutes
(GET "/api/links" []
  (let [links '({:url "https://google.ca" :title "Google" :client "static" :created_at "2015-12-01"}
                {:url "https://twitter.com :title "Twitter" :client "static" :created_at "2015-12-01"})]
    (json/write-str links)))
```

Giving it a quick run in the console, `curl -vi http://localhost:3000/api/links`, I see the desired results. With that done, I need to add a way to consume this data on the client and finish for tonight.

Oh, I forgot. I need to wrap the response as JSON, otherwise things go wonky.

``` clojure
(:require [ring.middleware.json :as wrapper])

;; then down at the bottom

(def app
  (wrap-defaults
    (wrapper/wrap-json-body app-routes {:keywords? true})
    site-defaults))
```
## Back to the client...

I specified the URL to the data as a property of the outermost container. I would like to query that URL now and populate `this.state.data` variable of the component with the data from the URL. For that, I'm going to use the `componentDidMount` function. Within, I am using the standard `$.ajax` call from jQuery and then binding the results to the `data` property of the `state` object.

``` javascript
componentDidMount: function() {
  $.ajax({
    url: this.props.url,
    dataType: "json",
    success: function(data) {
      this.setState({ data: data });
    }.bind(this),
    error: function(xhr, status, err) {
      console.error(this.props.url, status, err.toString());
    }.bind(this)
});
```

ReactJS frowns upon the usage of `=` as the assignment operator. Instead, I can use the `setState` function.

Reloading the page, I still see nothing, but that's fine, I haven't built a component to consume the data. In the Network tab of Web Inspector, though, I see the API call.

## Last thing in this installment...

So, the only thing left is render out a table of links. For that, I need to fill out the `LinkList` component and add a `Link` component. The former describes a list of links, while the latter is specific to one link record only. Old-timers reading this post will recognize as the master-detail layout.

``` javascript
var Link = React.createClass({
  render: function() {
    return (
      <tr>
        <td><a target="_blank" href={this.props.url}>{this.props.title}</a></td>
        <td>{this.props.client}</td>
        <td>{this.props.created_at}</td>
      </tr>
    );
  }
});

var LinkList = React.createClass({
  render: function() {
    var linkNodes = this.props.data.map(function (link) {
      return (<Link key={link.id} title={link.title} url={link.url} client={link.client} created_at={link.created_at} />);
    });
    return (
      <table className={"table table-striped table-hover"}>
        <thead>
          <tr>
            <th>URL</th>
            <th>From</th>
            <th>On</th>
          </tr>
        </thead>
        <tbody>
          {linkNodes}
        </tbody>
      </table>
    );
  }
});
```

I am fairly certain using that `map` function isn't the "ideal" way of building sub-components, but it works. Refreshing the tab, I see the 2 hard-coded links I added and, the UI looks like the existing UI.

That's it for now, then.

Good day/night/whatever.