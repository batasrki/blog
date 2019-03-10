---
layout: post
title: "Experiments part 4"
date: 2015-12-15 22:02:56 -0500
comments: true
categories: reactjs clojure
---

## Previously in [Experiments, part 3](https://batasrki.github.io/blog/2015/12/08/experiments-part-3),...

I set up a card-based UI and added a few more records to my test data set.

## In tonight's installment...

It's still (mostly) about the UI. I'm going to create a form for adding new links. I will be appending the links to the test data set and push dealing with the DB until later.

## Onward, then...

Up to now, this has been a read-only app. It's time to add the creation part. I'm going to not worry about persistently storing yet. Instead, I'll add the newly created link to the `atom` I created in the last post.

Firstly, I'm going to add a form component.

``` javascript
var LinkForm = React.createClass({
  render: function() {
    return (
      <div className={"col-lg-6", "well"}>
        <form className="form-horizontal" onSubmit={this.handleSubmit}>
          <fieldset>
            <legend>Save a link</legend>
            <div className="form-group">
              <label className={"col-lg-2 control-label"}>URL</label>
              <div className="col-lg-10">
                <input name="url" placeholder="URL" type="url" className="form-control" ref="url" />
              </div>
            </div>
            <div className="form-group">
              <label className={"col-lg-2 control-label"}>From</label>
              <div className="col-lg-10">
                <input name="client" placeholder="From" type="text" className="form-control" ref="client" />
              </div>
            </div>
            <div className="form-group">
              <div className={"col-lg-10 col-lg-offset-2"}>
                <button className={"btn btn-primary"} type="submit">Save</button>
              </div>
            </div>
          </fieldset>
        </form>
      </div>
    );
  }
});
```

For now, I'll stick it under the list component, but later I may break it into a modal.

``` javascript
render: function() {
  return (
    <section className="linkListContainer">
      <LinkList data = {this.state.data} />
      <LinkForm onLinkSubmit={this.handleLinkSubmit} />
    </section>
  );
}
```

So far, so straightforward. I've added a form, laid out in typical Bootstrap style, then added it as a component to the parent component.

### Submits

A sharp-eyed reader will notice two seemingly missing pieces, `onLinkSubmit={this.handleLinkSubmit}` in the declaration of the form component and `onSubmit={this.handleSubmit}` in the action of the `<form>` element.

These two functions are the drivers of the form's behaviour. I'm going to show and explain the latter one first.

``` javascript
var LinkForm = React.createClass({
  handleSubmit: function(e) {
    e.preventDefault();
    var url = this.refs.url.value.trim();
    var client = this.refs.client.value.trim();

    if (!url || !client) {
      return;
    }

    this.props.onLinkSubmit({url: url, client: client});
    this.refs.url.value = '';
    this.refs.client.value = '';
    return;
  },
  // render function below
```

All the form's submit function does is parse out the values from the two input fields, does a quick presence validation, ***passes on the values to the parent component***, and clears the fields.

I've highlighted the cool and important part. The function itself doesn't actually do any XHR calls. Its responsibility is to get the values and pass them up the hierachy. As far as I can tell, this helps with centralizing state changes. The parent component will deal with the actual data changes and it will then tell every descendant to re-render themselves.

Here's the function that does the actual server call.

``` javascript
// componentDidMount above
handleLinkSubmit: function(link) {
  $.ajax({
    url: this.props.url,
    dataType: "json",
    contentType: "application/json",
    type: "POST",
    data: JSON.stringify(link),
    success: function(data) {
      this.setState({data: data});
    }.bind(this),
    error: function(xhr, status, err) {
      console.error(this.props.url, status, err.toString());
    }.bind(this)
  });
},
// render below
```

It's pretty straightforward. The object named `link` is passed from the form component's `handleSubmit` function, which is then stringified and sent to the server. On a successful response, the `LinkListContainer` component receives a full set of data, which is what the internal state is set to. This will in turn cause child components to re-render themselves.

### Fake it until you...need it

I'm now going to implement the `POST` route on the server-side, where I'll attach the new link to the existing data set and send that down.

``` clojure
;; data set refresher
(def links
  (atom '({:id 1 :url "https://google.ca" :title "Google" :client "static" :created_at "2015-12-01"}
          {:id 2 :url "https://twitter.com" :title "Twitter" :client "static" :created_at "2015-12-01"}
          {:id 3 :url "https://github.com" :title "Github" :client "static" :created_at "2015-12-08"}
          {:id 4 :url "https://www.shopify.ca" :title "Shopify" :client "static" :created_at "2015-12-08"}
          {:id 5 :url "https://www.youtube.com" :title "YouTube" :client "static" :created_at "2015-12-08"})
  )
)
;; in app-routes
(POST "/api/links" request
  (let [next-id (inc (apply max (map #(:id %) @links)))
        new-link (:body request)
        new-link-with-fields (assoc new-link :id next-id :title (:url new-link) :created_at "2015-12-15")]
    (swap! links conj new-link-with-fields)
    (json/write-str @links)))
```

I love how this works. I really, really do. I'm going to explain each line starting from `let`.

``` clojure
next-id (inc (apply max (map $(:id %) @links)))
```

ReactJS would like a key to hang its rendering on. I am using the `ID` of each record. This line implements a simplistic ID auto-increment. It lifts the ID values out of each record in the `links` atom. It then applies `max` to the new list to find the highest value. Finally, it increments that value by one.

``` clojure
new-link (:body request)
```

Maps are the predominant data structure in Clojure web development. At least, they are from what I've seen in my limited exposure. Here, I take the body of the request which represents the JSON data I passed in from the client.

``` clojure
new-link-with-fields (assoc new-link :id next-id :title (:url new-link) :created_at "2015-12-15")
```

I need to add a few more things to the new record, namely `title`, `id`, and `created_at`. The last one will be hard-coded, the title can be equal to the URL and ID will be the `next-id` I computed above.

Also, isn't it neat how I can create a new value within a `let` and then use it to create another one within the same `let`?!? I think it is.

``` clojure
(swap! links conj new-link-with-fields)
```

Once I have a new, valid record, I need to add it to my dataset. Since I'm using the `atom` construct, I need to use `swap!` to update its value. Atoms are neat, too. So many neat things.

``` clojure
(json/write-str @links)
```

Finally, with all of the transformation done, I create a JSON response that is sent back to the client. When I run the whole app, it works exactly as if there was a database backing it. Woot.
