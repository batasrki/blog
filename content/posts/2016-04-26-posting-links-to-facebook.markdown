---
layout: post
title: "Posting links to Facebook"
date: 2016-04-26 21:20:03 -0400
comments: true
categories: facebook clojure reactjs
---

Tonight, I'll be doing something slightly different and actually, possibly relevant to my day job. I'm going to take the links I have saved and post them on my Facebook account. Using Clojure and ReactJS! It may be an interesting exploration into interfacing with Facebook or it may not. I have no clue and isn't that lack of knowledge what makes the life worth living?

## Let's go

I'm guessing I can post straight from Javascript, but I'd like to track how links I share will be received. I can have a `core.async` process somewhere poll the Facebook API and get data. I'll have to associate post IDs with the links I share.

I've created a Facebook App from my account. The next thing to do is copy the given JS into a file somewhere. I'm choosing to create a new file `js/fb-integration.js` and require it in the `views.clj` file at the end of the `body` declaration.

``` clojure
(include-js "/js/fb-integration.js")
```

After loading the app in the browser and checking there are no errors, I'm ready to do the next step. I want to log in and get permissions to post on a Facebook Page. I don't need to do this unless there's a desire to post a link to Facebook. So, I'm going to add a button into the `Link` component much like the delete button already added. The markup for the `Link` component looks like this now.

``` javascript
<div className={"card card-block"}>
  <h4 className="card-title">
    <a target="_blank" href={this.props.url}>{this.props.title}</a>
    <form onSubmit={this.handleDelete}>
      <input type="submit" value="X"/>
    </form>
    <!-- THE NEW THING! -->
    <form onSubmit={this.handleFBShare}>
      <input type="submit" value="Share!" />
    </form>
    <!-- END -->
  </h4>
  <p className="card-text">
    Created by {this.props.client} at {this.props.created_at}
  </p>
</div>
```

I now need to figure out how to invoke the log in modal and have it ask for permissions. Additionally, I need to find out what permissions I need. The documentation seems to suggest that I need the `manage_pages` and `publish_pages` permissions, so I'll add that to the scope of permissions requested on log in. A few experiments in the browser yields results I want to see.

``` javascript
FB.login(function(resp) { console.log(resp) },
         {scope: ['manage_pages', 'publish_pages']});

//returns
{authResponse: Object, status: "connected"}

FB.api('/me/accounts', function(resp) { console.log(resp) });

//returns
{data: Array[2], paging: Object}
```

The `FB.api` call returns a list of pages with their IDs and access tokens that I can use to publish links to. I'm going to send that info to the Clojure backend and see what happens.

## Initial front-end implementation

Since this is an experiment, I'm not concerned with the code quality. I just want it to work. Propagating the event from the `Link` component to the `LinkListContainer` component works much the same way as it did in the link deletion. I'm going to chain callbacks, which is widely accepted as "the wrong thing to do", so don't do this in a real application. The event handler will log in, ask for all the pages using the `/me/accounts` endpoint, grab the JS object representing the page I'd like to post to, and then send that page's access token and ID to the server. The server will use that information to actually make the post.

``` javascript
// handlers above
handleFBShare: function(data) {
  var parent = this;
  FB.login(function(response) {
    if(response.status === "connected") {
      FB.api("/me/accounts", function(response) {
        var link_experiments_page = response.data.filter(function(page) {
          if(page.name === "Link Experiments") {
            return page;
          }
        })[0];
        $.ajax({
          url: parent.props.url + "/" + data.id + "/share",
          dataType: "json",
          contentType: "application/json",
          type: "POST",
          data: JSON.stringify({access_token: link_experiments_page.access_token, page_id: link_experiments_page.id}),
          success: function(data) {
            console.log(data);
          }.bind(parent),
          error: function(xhr, status, err) {
            console.error(parent.props.url +"/" + data.id + "/share", status, err.toString());
          }.bind(parent)
        });
      });
    }
  });
}
// render below
```

Clicking on the `Share` button illicits a 404, since the server has no endpoint set up. That's up next.

## Initial server-side implementation

This should be as easy as adding a route and a function that responds to the request.

``` clojure
(defn post-link-to-facebook
  [request]
  (println request)
  (let [id (Integer/parseInt (get-in request [:path-params :id]))]
    (ring-resp/response (json/write-str @links))))
```

All I want to do is inspect the request, but as I expected, this is simple. I really like well-designed languages and libraries. The data I sent from the client shows up in the `json-params` map, so I can extract the access token and the page ID as easily as the link ID.

I'm going to create a function that will take in the above data and make a post. I need to pull in the great `http-kit` library, as well, so that I can make HTTP requests in a simple manner. Firstly, I'm going to fill out the rest of the handler function.

``` clojure

(defn post-link-to-facebook
  [request]
  (let [link-id (Integer/parseInt (get-in request [:path-params :id]))
        link (first (filter #(= (:id %)) @links))
        page-id (get-in request [:json-params :page_id])
        access-token (get-in request [:json-params :access_token])]
    (create-a-post page-id access-token link)
    (ring-resp/response (json/write-str @links))))
```

With that done, I'm ready to actually make a post. Firstly, I'm adding `http-kit` to the list of required libraries.

``` clojure
(:require [org.httpkit.client :as http]
          ;; all the other things)
```

Now, I'm going to write a quick-and-dirty function to do the posting. I need to hit the `graph.facebook.com` domain with the ID of the page and its access token. The `message` field is the actual thing shown. I'll keep this field really simple right now.

``` clojure
(defn create-a-post [fb-page-id fb-page-token link]
  (let [post-url (str "https://graph.facebook.com/" fb-page-id "/feed?message=Look%20at%20this%20link%20" (:url link) "&access_token=" fb-page-token)
        {:keys [status headers body error] :as resp} @(http/post post-url)]
    (if error
      (println "Failed, exception: " error)
      (println "Success, status: " status))
  ))
```

If I now reload the app and click `Share`, I get a post on the desired Facebook Page!

![facebook post](/images/facebook-post.png)

That is great. As much as I dislike Facebook's privacy nonsense, I have to admit that it is really easy to do things such as this. Of course, I also always enjoy Clojure's ability to make these things so, so simple, as well.

OK, so I have a few cleanup items left, but I'm happy that it only took a couple of hours to do this feature from scratch. The time taken includes researching everything from how to actually connect to Facebook using its JS SDK, finding the relevant Graph API docs, integrating that into the existing ReactJS app and writing the backend functionality.


TODO:

1. Clean up that ugly URL creation
2. Figure out how to post sane content
3. Send a response to the UI so that a shared link doesn't get shared again
