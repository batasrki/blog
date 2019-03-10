---
layout: post
title: "No Rails in my specs"
date: 2011-12-07
comments: true
categories: fast rails rspec tests
---

#### tl; dr

Removing Rails from the runtime of an RSpec suite will result in massive performance improvements while reducing the number of steps needed to run them. The trade-offs are:

A different way of writing specs, especially controller ones
There is overhead due to having to handle requires on your own
These trade-offs may not be worth the performance gain. Skim the code blocks if the post is too long.

Since my models are __not__ subclasses of ActiveRecord::Base, I am unaware of what additional work is involved. Hat tip to [Tommy Morgan](https://twitter.com/duwanis) for bringing up that concern.

#### Introduction

So, last night, I tweeted my amazement at a massive performance improvement while running specs for a Rails app I’m building. A few people have asked if I could write down my thoughts, so here they are.

First, though, a little bit of background. This after-hours app of mine is a bank account analysis web application. Basically, I wanted to be able to see at a glance what my wife and I are spending money on, how much of it and to eventually come up with a prediction model based on past financial history. Also, I’m treating this application as a non-trivial testbed of technologies and techniques. For example, the data store is [Riak](http://basho.com/riak/) and I am splitting the application up into services. Currently, I have two parts:
1. A [Sinatra-powered](http://sinatrarb.com) API service for querying the data and importing new source data
2. A thin [Rails](http://rubyonrails.org) client whose sole job is presenting the query results from the API service to the end user

#### Initial assessment

So, the Rails client is running on Ruby 1.9.3 and Rails 3.1.3, while [Typhoeus 0.3.3](https://github.com/dbalatero/typhoeus) is my HTTP library of choice. For testing, I’m using RSpec 2.7.0. It’s important that I note the versions here, because things may change for the better. I used test-first methodology to drive out the initial iteration of the client, based on a few requirements I jotted down on a piece of paper. I deliberately skipped creating ActiveRecord (or any other ORM) models and I went with straight-up Ruby objects. After all, the data store for the client is the API service, so I didn’t see the need to duplicate the data store part here.

While the model part is not the usual Rails fare, the controller and view parts are pretty standard with some modern techniques, like using presenters/decorators instead of helpers, mixed in. I wrote the controller and model specs in the usual way, mocking out the API response in the controller and letting the model specs hit the API service like they would in the production mode.

When I ran the suite of 30 specs, I was flabbergasted to see the suite take 9 seconds to run. The specs ran fairly fast and most of the time seemed to be spent loading up the environment. This run time surprised me, because the entire Ruby universe on Twitter kept saying how much better 1.9.3 was at loading up the Rails environment than 1.9.2. I can only imagine how long this would’ve taken to run using 1.9.2. To me, 9 seconds for each spec suite run is unacceptable. To hell with premature optimization, this was an issue right now and it’d be better if I solved it while I only have 30 specs.

#### Spork

As usual, the first tool I reach for when trying to improve the run time of a spec suite is Spork. It’s a well-known and trusted tool among Rails people and I’ve used it before. A quick gem install later, followed by the bootstrap and I’m off and running.

Originally, my spec_helper.rb file looked like this:

{% codeblock lang:ruby %}
require 'rubygems'
require 'spork'

Spork.prefork do
end

Spork.each_run do
  # This code will be run each time you run your specs.
end
ENV["RAILS_ENV"] ||= 'test'
require File.expand_path("../../config/environment", __FILE__)
require 'rspec/rails'
require 'rspec/autorun'

# Requires supporting ruby files with custom matchers and macros, etc,
# in spec/support/ and its subdirectories.
Dir[Rails.root.join("spec/support/**/*.rb")].each {|f| require f}

RSpec.configure do |config|
  config.mock_with :rspec
  config.infer_base_class_for_anonymous_controllers = false
end
{% endcodeblock %}

I moved everything into the __Spork.prefork__ block and left the __Spork.each\_run__ block empty. This worked immediately dropping my run time from 9 seconds (well, after the initial 8 second load time) to 0.8 seconds. I cannot emphasize how big a win Spork is early in the life of a project. I mean, 2 minutes of work has shaved off a tonne of time. However, this wasn’t without its issues.

The big issue I ran into almost right away was that the changes to my model classes weren’t being applied between runs. This is a rhythm-killer for me. I had to stop and reload spork every time I changed a model class, paying the 8 second load penalty. Controller specs weren’t affected, so I suspect that this had something to do with the fact that my model classes weren’t subclassing ActiveRecord::Base. A bit of googling later, my spec_helper file became this:

{% codeblock lang:ruby %}
require 'rubygems'
require 'spork'

Spork.prefork do
  # Loading more in this block will cause your tests to run faster. However, 
  # if you change any configuration or code from libraries loaded here, you'll
  # need to restart spork for it take effect.
  ENV["RAILS_ENV"] ||= 'test'
  require File.expand_path("../../config/environment", __FILE__)
  require 'rspec/rails'
  require 'rspec/autorun'
  Dir[Rails.root.join("spec/support/**/*.rb")].each {|f| require f}
  RSpec.configure do |config|
    config.mock_with :rspec
    # If true, the base class of anonymous controllers will be inferred
    # automatically. This will be the default behavior in future versions of
    # rspec-rails.
    config.infer_base_class_for_anonymous_controllers = false
  end
end

Spork.each_run do
  Dir["#{Rails.root}/app/controllers//*.rb"].each do |controller|
    load controller
  end
  Dir["#{Rails.root}/app/models//*.rb"].each do |model|
    load model
  end
  Dir["#{Rails.root}/app/models/api/*.rb"].each do |model|
    load model
  end
  Dir["#{Rails.root}/app/presenters//*.rb"].each do |presenter|
    load presenter
  end
end
{% endcodeblock %}

I was annoyed that I had to do that, but it worked as advertised so I moved on.

#### To the extreme

The immediate annoyance I encountered with this setup is the new process I had to adopt. In order to write new specs, I had to do the following:
1. Start up the API service, complete with the database start-up
2. Start up spork, wait 8 seconds
3. Run the existing specs
4. Write a new one and cycle

Now, I realize that there is a gem out there, [guard](https://github.com/guard/guard), which will automate steps 2 and 3 for me. However, this is a thin Rails client with minimally few moving parts and I didn’t feel like repeating the incantations needed to set up guard properly. Moreover, step 1 is still something I have to do manually, so the win with guard isn’t as big.

Furthermore, forgetting to start guard or spork will cause me to get frustrated, while I wait the ~9 seconds it will take to run the suite plus the usual 8 second load time when I do start spork up. I just want to run __rspec spec/__, for crying out loud.

I had heard of Corey Haines’ presentation on [fast rails tests](http://confreaks.net/videos/641-gogaruco2011-fast-rails-tests) where he essentially removes Rails from the spec suite runtime. There were also a few blog posts around the same issue. I figured I had nothing to lose trying out this approach, since my Rails client isn’t a “real”™ Rails application.

So, I removed:

{% codeblock lang:ruby%}
require 'spec_helper'
{% endcodeblock %}

from every spec file I had and set about fixing all of the errors that popped up. What I came up with within an hour is this:

{% codeblock lang:ruby %}
APP_ROOT = File.expand_path(File.join(File.dirname(__FILE__), '..'))
$: << File.join(APP_ROOT, "app", "controllers") << File.join(APP_ROOT, "app", "models") << File.join(APP_ROOT, "app", "presenters")

module ActionController
  class Base
    def self.protect_from_forgery; end
    def params; {}; end
  end
end

require 'active_support/core_ext/string/inflections'
require 'date'
require 'vcr_setup'
require 'application_controller'
require 'transactions_controller'
require 'categories_controller'
require 'api/transaction'
require 'transaction_collection'
require 'transaction'
require 'category_formatter'

def assigns(name)
  controller.instance_variable_get "@#{name}"
end

RSpec.configure do |config|
  config.mock_with :rspec
end
{% endcodeblock %}

As you’ll note, the spec_helper file is pretty different when compared to a regular one, no surprise there. I used the setup from this blog post, [Running Rails Rspec Tests - Without Rails](http://www.adomokos.com/2011/04/running-rails-rspec-tests-without-rails.html) and an inspiration from this [gist](https://gist.github.com/1294164) with a few modifications.

As the blog post explains in detail, requiring Rails is the thing that kills the startup time of the spec suite. I don’t think it’s a big revelation, but it’s one worth repeating. If you want really fast Rails tests, __remove RAILS!__

The above code monkey-patches __ActionController::Base__ and provides a dummy implementation of a few methods to ensure that ApplicationController and its subclasses don’t blow up. After adding the paths to various parts of the app to the load path (that’s the __$:__ thingie), requiring the various classes and defining a helper method, we’re pretty much done.

There are a few other things I needed to explicitly require after ripping out Rails, namely the __date__ library from Ruby’s stdlib and the __inflector__ from ActiveSupport that is used in the presenter spec.

The controller specs look like this now:

{% codeblock lang:ruby %}
require 'spec_helper'

describe TransactionsController do

  describe "GET 'index'" do
    let(:controller) { TransactionsController.new}
    before do
      builder = TransactionCollection.new
      parsed_json = [{"key12" =>{:category => "insurance", :amount => 332}}]
      API::Transaction.stub!(:transactions_for).with("insurance").and_return(parsed_json)
    end

    it "has a category assigned" do
      controller.class.send(:define_method, :params) do
        {:category_id => "insurance"}
      end
      controller.index
      assigns("category").should_not be_nil
    end

    it "returns a set of transactions" do
      controller.class.send(:define_method, :params) do
        {:category_id => "insurance"}
      end
      controller.index
      transactions = assigns("transactions")
      transactions.first.key.should eql "key12"
    end

    it "returns an empty array when there are no transactions for the category" do
      controller.class.send(:define_method, :params) do
        {:category_id => "testme"}
      end
      API::Transaction.stub!(:transactions_for).with("testme").and_return([])
      controller.index
      assigns("transactions").should eql []
    end
  end
end
{% endcodeblock %}

Disregarding the duplication setup that I left in there, the invocation of the __index__ action in the __TransactionsController__ is really not that much different from how it’s normally done. The duplicated part sets up the params hash with the expected key/value pair.

The model specs have not changed while I was ripping stuff apart, which made me really happy.

#### VCR

The eagle-eyed among you, might have noticed this line in the spec_helper:

{% codeblock lang:ruby %}
require 'vcr_setup'
{% endcodeblock %}

After I got the suite running, I noticed that what was previously 0.8 seconds with spork was now consistently around 2 seconds. The only cause for this slow down were the model specs that hit the API service. Looking through [Avdi Grimm's](https://twitter.com/avdi) blog posts and various podcast appearances, I came upon the [VCR](https://github.com/myronmarston/vcr) gem.

What this gem does is records the HTTP requests your specs make in the course of a run, stores the results of said call into a YAML file to which you can give a name, cuts off HTTP access to your specs and returns the contents of each YAML file made as if the spec still made that HTTP request. I think showing a bit of code may explain this better than words:

{% codeblock lang:ruby %}
it "should return a set of categories" do
  VCR.use_cassette("built-in categories") do
    API::Transaction.categories.should_not be_empty
  end
end
{% endcodeblock %}

As you can see, I’ve named this cassette “built-in categories” which will cause VCR to store a YAML file named _built\_in\_categories.yml_ under the __spec/vcr\_cassettes__ directory. On the first run after install, VCR will execute the HTTP request inside the _use\_cassette_ block and save the result, replaying it with every subsequent run. There are options available to let you control if and when these cassettes should expire, as well as many more. The documentation is [here](https://relishapp.com/myronmarston/vcr).

This library is an absolute gem (pardon the pun). It works as advertised and extremely smoothly. Even things that weren’t well-documented, but seemed like they should work, did. For example, if you’re executing an HTTP request in a before block for multiple spec runs, the same syntax applies and works. I am thoroughly impressed with this gem, as is Avdi.

#### The long-awaited payoff

So, why go through all of this? If you’re asking yourself that, don’t worry, I’ve asked myself the same. For me, there are a few big wins.

Firstly, the speed. Repeated full suite runs clock in at 0.5 seconds (I said 0.7 on Twitter, but I was wrong). That’s right, it takes me __half a second__ to run all the specs, the controller ones, the model ones and the presenter ones. When I compare this to the initial run of 9 seconds in total, I grin.

Secondly, spork is now out of the picture. For me, this is a huge relief on mental load. The process that I outlined above is reduced to steps 3 and 4. I can type in __rspec spec/__, note any failures I left for myself from the night before, make the specs pass, refactor and move. The rhythm is back and it’s the rhythm that makes me productive.

This is does not imply that spork sucks. Installing spork, modifying the spec_helper, firing spork up and testing is a very legitimate way of improving spec suite run time. I’ve outlined why I don’t like this process, but I’m not ever going to argue against it if it works for you.

I wanted to prove to myself that it’s possible to have Rails specs without Rails. Having an unusual use case helped push me down this path. It is possible and, as of right now, it isn’t that much work. __I do not know if this approach scales, though__. I need to emphasize that point. I may come back a few months down the road saying that this is too much work. I don’t know.

What I do know is that it’s been a worthwhile experiment and I hope that documenting it in this way will help someone else, as well. Good luck.

Props and fist bumps go to [Tommy Morgan](https://twitter.com/duwanis) for proof-reading, [Tony Collen](https://twitter.com/tcollen) for the gist and the blog post link and [Myron Marston](https://twitter.com/myronmarston) for the VCR gem.