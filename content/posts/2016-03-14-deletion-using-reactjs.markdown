---
layout: post
title: "Deletion using ReactJS"
date: 2016-03-14 00:01:34 -0400
comments: true
categories: clojure reactjs
---

Tonight, I will start on enabling deletion of links in my ReactJS single-page app. It should be straightforward, but with new tech, you just never know.

## Client-side first

On the client-side, I need add a small form that will submit the `DELETE` request to the server using the given ID. The most logical place for the markup is on the component, but I suspect that I'll need to pass the event up to the root component in order to keep the one-way data flow and easy re-rendering.

### Markup

Firstly, I am going to amend the `Link` component to add the delete button.


``` html
  <div className={"card card-block"}>
    <h4 className="card-title">
      <a target="_blank" href={this.props.url}>{this.props.title}</a>
      <form onSubmit={this.handleDelete}>
        <input type="submit" value="X"/>
      </form>
    </h4>
    <p className="card-text">
      Created by {this.props.client} at {this.props.created_at}
    </p>
  </div>
```

Following the pattern laid out for submitting new links, I added the simplest of forms to the title part of the link markup. The `onSubmit` handler is a local function.

The function, much like for link creation will just pass its specific local state up to the root component that will take care of the actual request to the backend.

``` javascript
handleDelete: function(e) {
  e.preventDefault();
  var id = this.props.id;
  this.props.onLinkDelete({id: id});    
}
```

### Event propagation...the ugly way

I am certain I'm doing things the wrong way, because in order for me to have the root component do the actual communication and state rendering, I have to pass the callback function *all the way* from root component down to the lowest level.

``` javascript
var LinkGroup = React.createClass({
// some code
  return (<Link onLinkDelete={callback} key={link.id}
                id={link.id} title={link.title}
                url={link.url} client={link.client}
                created_at={link.created_at} />);
// more code
});

var LinkList = React.createClass({
// some code
  return(<LinkGroup data={link_group}
                    onLinkDelete={callback} />);
// more code
});

var LinkListContainer = React.createClass({
// some code
  <LinkList data = {this.state.data}
            onLinkDelete={this.handleLinkDelete} />
// more code
```

This works, but I am sure it is **not** the way. I'll have to research more.

## The actual event handler

Now that I'm passing the handler into the correct spot and returning correct data, I'm going to write the simplest handler that can possibly work.

``` javascript
handleLinkDelete: function(data) {
  $.ajax({
    url: this.props.url + "/" + data.id,
    dataType: "json",
    contentType: "application/json",
    type: "DELETE",
    success: function(data) {
      this.setState({ data: data });
    }.bind(this),
    error: function(xhr, status, err) {
      console.error(this.props.url + "/" + data.id, status, err.toString());
    }.bind(this)
  });
}
```

This is just the simple, standard jQuery-based AJAX call. The server will return the updated atom that the ReactJS will use to re-render the UI.

That's it for tonight. Tomorrow, I'm going to hook up the backend.

## TODO

1. Hook up the backend
2. Find a cleaner way to propagate the event to the root component