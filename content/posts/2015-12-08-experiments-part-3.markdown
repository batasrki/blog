---
layout: post
title: "Experiments part 3"
date: 2015-12-08 00:42:50 -0500
comments: true
categories: reactjs clojure
---

## Previously in [Experiments, part 2](https://batasrki.github.io/blog/2015/12/01/experiments-part-2),...

I set up dummy data on the backend, then created the initial set of components on the front end that consume that data.

## In tonight's installment...

It's all about the UI. I'm going to add a few more links to the dummy set, then set up my card-based UI.

## Onward, then...

Firstly, I'm going to add 3 more links to my dataset so that I can play with the layout.

``` clojure
(def links
  (atom '({:id 1 :url "https://google.ca" :title "Google" :client "static" :created_at "2015-12-01"}
          {:id 2 :url "https://twitter.com" :title "Twitter" :client "static" :created_at "2015-12-01"}
          {:id 3 :url "https://github.com" :title "Github" :client "static" :created_at "2015-12-08"}
          {:id 4 :url "https://www.shopify.ca" :title "Shopify" :client "static" :created_at "2015-12-08"}
          {:id 5 :url "https://www.youtube.com" :title "YouTube" :client "static" :created_at "2015-12-08"})
  )
)

(defroutes app-routes
  (GET "/" []
       (views/index))
  (GET "/api/links" []
       (json/write-str @links))
  (route/not-found "Not Found"))
```

I'm extracting the static set into an [atom](http://clojure.org/atoms), and then de-referencing it to grab its value inside the JSON render call. I can easily add more to this set if I need to play with bigger sets while I lay things out.

Now, I'm ready to switch the table UI to a card one.

## Card UI

To do that, I'm going to need to add a component. I'd like to have a group of 4 cards per row, so I need partition my set into groups of 4 and I need a `LinkGroup` component to manage each partition.

``` javascript
var LinkGroup = React.createClass({
  render: function() {
    var linkNodes = this.props.data.map(function (link) {
      return (<Link key={link.id} title={link.title} url={link.url} client={link.client} created_at={link.created_at} />);
    });

    return (
      <section className={"card-columns"}>
        {linkNodes}
      </section>
    );
  }
});

var LinkList = React.createClass({
  render: function() {
    var grouped = this.partition(this.props.data, 4);

    var linkGroups = grouped.map(function (link_group) {
      return(<LinkGroup data={link_group} />);
    });

    return (
      <section>
        {linkGroups}
      </section>
    );
  },
  partition: function(list, in_groups_of) {
    var container = [];
    var sublist = [];
    var initial = 0;
    var iteration_count = Math.ceil(list.length / in_groups_of);

    for(var k = 1; k <= iteration_count; k++) {
      for(var i = initial; i < (in_groups_of * k); i++) {
            if(list[i] !== undefined) {
            sublist.push(list[i]);
            }
      }
      container.push(sublist);
      sublist = [];
      initial = in_groups_of * k;
    }
    return container;
  }
});
```

Let me explain the changes. The `LinkList` component is now rendering a set of `LinkGroup` components. Each `LinkGroup` component is rendering its own subset of `Link` components. I chose, for now, to do the partitioning of the data in Javascript. It was a fun diversion, though I may decide to push that onto the server at a later point.

I've also added a class for card columns that will be leveraged below. Now, I need to modify my `Link` component to suit the new layout.

``` javascript
var Link = React.createClass({
  render: function() {
    return (
      <div className={"card card-block"}>
        <h4 className="card-title">
          <a target="_blank" href={this.props.url}>{this.props.title}</a>
        </h4>
        <p className="card-text">
          Created by {this.props.client} at {this.props.created_at}
        </p>
      </div>
    );
  }
});
```

The changes include ripping out `<tr>` and `<td>` elements and replacing them with `<div>`s and headers and paragraphs. I like this change, because I plan on adding another aspect of each link, namely a bit of text parsed out from it. I feel this layout suits that plan better.

Refreshing the page, I see the new layout and with that, I'm done. Next up is database work.

Until then.