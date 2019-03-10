---
layout: post
title: "Riak and Vagrant"
date: 2012-04-22
comments: true
categories: riak ruby vagrant
---

## tl; dr;

Creating a Riak cluster as a set of nodes through Vagrant and talking to the cluster through the official Ruby driver turned out to be harder than expected. Cryptic errors in Riak itself and lack of documentation for hooking up the driver to the cluster were two root causes. The final, working versions of the Vagrantfile and the Riak config files needed to get things working are below.

{% codeblock Vagrantfile lang:ruby %}
Vagrant::Config.run do |config|
  config.vm.define :node1 do |node|
    node.vm.box = "ubuntu-1110-server-amd64"
    node.vm.network :hostonly, "192.168.33.10"
    node.vm.forward_port 8098, 8091
    node.vm.forward_port 8087, 8081
  end

  config.vm.define :node2 do |node|
    node.vm.box = "ubuntu-1110-server-amd64"
    node.vm.network :hostonly, "192.168.33.11"
    node.vm.forward_port 8098, 8092
    node.vm.forward_port 8087, 8082
  end

  config.vm.define :node3 do |node|
    node.vm.box = "ubuntu-1110-server-amd64"
    node.vm.network :hostonly, "192.168.33.12"
    node.vm.forward_port 8098, 8093
    node.vm.forward_port 8087, 8083
  end
end
{% endcodeblock %}

{% codeblock vm.args lang:erlang %}
## Name of the riak node
## You need to change the values on both sides of the @ sign
-name node1@192.168.33.10
{% endcodeblock %}

### Introduction

A few nights ago, I embarked on getting a proper Riak cluster going on my big dev machine. Up to this point, I’ve been developing against a single Riak instance. Since Riak is meant to be used in a cluster and its behaviour can differ when multiple nodes are activated, I decided to upgrade to the cluster. I could have just used the devrel builds of Riak, by dowloading the latest version of Riak’s [source](http://s3.amazonaws.com/downloads.basho.com/riak/1.3/1.3.1/riak-1.3.1.tar.gz), issuing make all and make devrel commands and generally following [these steps](http://docs.basho.com/riak/latest/tutorials/fast-track/Building-a-Development-Environment/). However, since I do plan on eventually shipping my Riak-powered project, what I really wanted to do was build a true cluster of Ubuntu servers running the 64-bit [binary package](http://s3.amazonaws.com/downloads.basho.com/riak/1.3/1.3.1/debian/6/riak_1.3.1-1_amd64.deb) of Riak on each instance.

#### Existing information

After I’ve done too many of the steps below, I finally entered ‘riak vagrant’ as the search term in Google. I ended up on a [post](http://basho.com/creating-a-local-riak-cluster-with-vagrant-and-chef/) on the Basho blog that details how to use Vagrant and Chef to automatically provision nodes for the cluster. After much head-smacking, I realized that I actually preferred to step through the build manually, lest the automatic provisions obscure a part of the process. I do realize that others may want to expedite said process, which is why I’m linking to the post. Onwards then.

### Cluster build options

There are several ways available to me to have real instances running, including [EC2](http://aws.amazon.com/ec2) and [Rackspace](http://www.rackspace.com/cloud). I didn’t want, or really need to right now, to figure out how to set up instances I want on those two, much less trying to get Riak installed and running. There is a third option, and as you might have guessed from the title of this post, it involves [Vagrant](http://vagrantup.com/).

Vagrant is a wonderful, Virtualbox-powered, tool that lets one locally provision instances of operating systems and interact with them as if they’re real server boxes. After I downloaded and installed Vagrant, I went in search of a good-enough box that would serve as the basis of the nodes in my cluster. After a quick visit to [Vagrantbox.es](http://vagrantbox.es/), I downloaded the [Ubuntu 11.10 64-bit box](http://dl.dropbox.com/u/3886896/oneiric64.box).

The next steps added the box to my setup:

{% codeblock %}
vagrant box add (fully qualified box name)
vagrant init ubuntu-1110-server-amd64
{% endcodeblock %}

With the box added, I scoured Vagrant’s [excellent documentation](http://docs.vagrantup.com/v2/) for ways to get multiple instances of the same box up and running. I first tried making multiple subdirectories and either copying the Vagrantfile into each and making appropriate modifications or symlinking the original Vagrantfile to each subdirectory. While the first approach worked, it clearly wouldn’t scale as I added more nodes to the cluster. The latter approach, not surprisingly, failed spectacularly.

I, then, ran across the [Multi-VM Environments](http://docs.vagrantup.com/v2/multi-machine/index.html) page in the documentation. This is actually what I wanted and I quickly removed the subdirectories and began building my cluster in the parent directory.

### Setting up Riak on the nodes

Now that I have 3 nodes up and running, it’s time to get Riak installed and running. After SSH-ing into each instance in turn, I download the above binary distribution using curl:

{% codeblock %}
# same for node2 and node3
vagrant ssh node1

curl -O http://downloads.basho.com.s3-website-us-east-1.amazonaws.com/riak/1.1/1.1.2/riak_1.1.2-1_amd64.deb

{% endcodeblock %}

This will download the 1.1.2 version of Riak, which is currently the latest stable version. Change the url as needed. The install originally fails, due to libssl being the version 1.0 and Riak wanting the 0.9.8. A sudo apt-get install command later, the install succeeds and we’re up and running. The nice thing about the binary install is that it will set up Riak to start on boot, which means I don’t have to muck around with that.

### Setting up the Riak cluster

Having Riak installed on all the VMs, it’s time to make them into a cluster. The documentation on how to connect nodes to the cluster is sparse and in multiple places. Currently, the best place to look for Riak documentation is in the [Riak handbook](http://riakhandbook.com/). It’s a self-published e-book, but it does aggregate a lot of information about Riak in one place and it’s much easier to search than Basho’s wiki and the Riak mailing list, at least right now.

So, as per the handbook, all one needs to do is change the 127.0.0.1 to the current machine’s IP address in the _vm.args_ file.

{% codeblock vm.args lang:erlang %}
## From
-name riak@127.0.0.1

## To
-name riak@192.168.33.11
{% endcodeblock %}

Restarting the node fails with an error, though.

{% codeblock %}
riak restart
# Node 'node1@192.168.33.11' not responding to pings.
{% endcodeblock %}

After referring to the Vagrant/Chef example above, I notice that ports in _app.config_ are bound to 0.0.0.0, i.e. all interfaces. Changing that lets me finally use the following command to add the current node to a cluster:

{% codeblock riak admin %}
riak-admin join riak@192.168.33.10
{% endcodeblock %}

After I set up each node with its own static IP, I changed the _vm.args_ file as above and executed the command. The _riak-admin status_ command should show nodes connected like this:

{% codeblock %}
riak-admin status | grep connected
# connected_nodes: [192.168.33.10]
{% endcodeblock %}

Everything seems connected and working, so we should be good.

### Talking to Riak nodes

Now that each node has Riak installed and running, I want to make sure that I can talk to each node from the host machine. The Vagrant documentation specifies that I can make a few types of networks for my nodes:

1. _hostonly_, meaning no external access, all traffic is between the nodes and the host machine
2. _bridged_, meaning each node shows up as a physical interface, presumably letting external traffic access it

I chose the _hostonly_ option for a few reasons, chief being that this is an experiment. Also, I think it’s beneficial from a system design perspective to not expose my database nodes to the vagaries of the internet. I could be wrong, but I’ve heard something to that effect.

Anyhow, I also would like to query these instances for data in Riak, so I have to forward traffic from my host machine to each node’s Riak port which, by default, is 8098. Actually, from what I’ve gathered from reading through the Chef blog post linked above, I have to set up two forwards. Port 8098 is the HTTP traffic port to Riak. It also exposes a protocol buffers port, it being 8087. In Vagrant, it’s done like so:

{% codeblock Vagrantfile lang:ruby %}
node.vm.forward_port 8098, 8091
node.vm.forward_port 8087, 8081
{% endcodeblock %}

This needs to be done for each node in the cluster where the second set of numbers is the port numbers on the host machine. I found that syntax slightly unintuitive and subject to multiple documentation lookups, but I guess I’ll get used to it eventually.

A quick check through curl reveals that each Riak instance responds to the basic curl command.

{% codeblock curl %}
curl -i http://localhost:8091

HTTP/1.1 200 OK
Vary: Accept
Server: MochiWeb/1.1 WebMachine/1.9.0 (participate in the frantic)
Link: </buckets>; rel="riak_kv_wm_buckets",</riak>; rel="riak_kv_wm_buckets",</buckets>; rel="riak_kv_wm_index",</buckets>; rel="riak_kv_wm_keylist",</buckets>; rel="riak_kv_wm_link_walker",</riak>; rel="riak_kv_wm_link_walker",</mapred>; rel="riak_kv_wm_mapred",</buckets>; rel="riak_kv_wm_object",</riak>; rel="riak_kv_wm_object",</ping>; rel="riak_kv_wm_ping",</buckets>; rel="riak_kv_wm_props",</stats>; rel="riak_kv_wm_stats"
Date: Mon, 23 Apr 2012 01:27:57 GMT
Content-Type: text/html
Content-Length: 616

<html><body><ul><li><a href="/buckets">riak_kv_wm_buckets</a></li><li><a href="/riak">riak_kv_wm_buckets</a></li><li><a href="/buckets">riak_kv_wm_index</a></li><li><a href="/buckets">riak_kv_wm_keylist</a></li><li><a href="/buckets">riak_kv_wm_link_walker</a></li><li><a href="/riak">riak_kv_wm_link_walker</a></li><li><a href="/mapred">riak_kv_wm_mapred</a></li><li><a href="/buckets">riak_kv_wm_object</a></li><li><a href="/riak">riak_kv_wm_object</a></li><li><a href="/ping">riak_kv_wm_ping</a></li><li><a href="/buckets">riak_kv_wm_props</a></li><li><a href="/stats">riak_kv_wm_stats</a></li></ul></body></html>%
{% endcodeblock %}

It’s now time to hook up the Riak Ruby client. It’s at this point that all documentation runs out and I end up reading the specs, source code and such. I lie, there is documentation on how to add nodes to the client’s initialization:

{% codeblock riak-ruby-client lang:ruby %}
# Automatically balance between multiple nodes
client = Riak::Client.new(:nodes => [
  {:host => '10.0.0.1'},
  {:host => '10.0.0.2', :pb_port => 1234},
  {:host => '10.0.0.3', :http_port => 5678}
])
{% endcodeblock %}

The trouble here is that I went through the pain of setting up a YAML file with the database configuration and I needed to port the above to YAML syntax. Reading through the YAML documentation, I find out the syntax needed for representing the array in YAML:

{% codeblock riak.yml %}
development: &base
  http_backend: :Excon
  nodes:
    -
      host: 'localhost'
      http_port: 8091
    -
      host: 'localhost'
      http_port: 8092
    -
      host: 'localhost'
      http_port: 8093
{% endcodeblock %}

I had to modify my homegrown code that turns string keys in hash into symbols, since the Riak client expects symbols as key types and not strings. This has caused me problems a few times, but I don’t know if it should be patched to accept either. All this being done, I expected to have full access to all nodes, reading from them and writing to them.

### Um, no

Instead, the error I got through the client and through curl is _{insufficient\_vnodes, 0, expected, 3}_. I had set up read/write values for Riak in order to control how data is replicated during writes, as well as how it’s read from the cluster. 

This terse message hints at the issue at hand. Even though I had set up the cluster as it is detailed in all of the sources I found, the nodes did not distribute the ring of hashed keys that point to data between themselves.

The way to see what’s really going on in the cluster is to make sure that the _connected\_nodes_ information in the status correlates with the _ring\_members_ information a few rows lower. In my case, as you can see below, the two rows did not reflect the same information. The nodes seem to see and connect to each other, but they have not shared the ring of data between them and are not in a cluster.

{% codeblock %}
connected_nodes : [riak@192.168.33.11, riak@192.168.33.12]
---snip 10 rows
ring_members : ['riak@127.0.0.1']
{% endcodeblock %}

Another tipoff is that _ring\_num\_partitions_ number of the node you’re querying is the size of the ring you’ve set up, 64 by default. When the node is in a cluster, it has a fraction of that number, that is _size of the ring \/ number of nodes_ in cluster.

Changing the information in _vm.args_ seemed to help.

{% codeblock vm.args lang:erlang %}
% Doesn't work
-name riak@192.168.33.10

% Works
-name node1@192.168.33.10
{% endcodeblock %}

Restarting the nodes and querying each node’s _ring\_status_ and _member\_status_ shows me conflicting information. The members are picked up as expected, but the ring status still seems to refer to _riak@127.0.0.1_, which it shouldn’t. Searching through the _app.config_ file, I find out where the ring information for each node is stored on the filesystem. In my case, it’s at _/var/lib/riak/ring_. Stopping the node, clearing out the files in this directory and starting the node finally, mercifully, shows me the correct information on all nodes.

### Finally, it works

Double checking whether I can query a node through curl and through Ruby confirms for me that the system now works. As I expected it, a 404 is returned for a non-existent key, a 204 responses for a successful POST and DELETE. I can now continue writing code against this system in the same way as I had done against a single node.

### Conclusion

[Mark Phillips](https://twitter.com/pharkmillups), Basho’s Director of Community, recently asked on the Riak mailing list how adoption of Riak can be improved. After going through the above, I have to say that documentation should be priority #1. I do realize that Riak is still in fast-development mode. Features are flying in every day and the core developers are continously improving stability and user-friendliness.

Having said that, the [wiki on Basho’s website](http://docs.basho.com) either doesn’t have all of the necessary information or it doesn’t have it organized well. It’s nearly impossible to search it intuitively. The Riak Handbook I linked to above is a better source, but it’s not free and the examples in it are written in JavaScript. I have nothing against JavaScript, but porting the examples is yet another thing I need to do to get things working in Ruby. In any case, the book cannot cover every error, such as the error I ran into here. This is information that needs to be a part of the living documentation provided by the database manufacturer themselves.

Furthermore, the documentation for the Riak’s Ruby driver is sorely lacking. I refer to the test suite for almost every single thing I need to do with the driver, because the README does not provide examples of how to do things. This is something I will definitively help out with, since Github provides a wiki that I can contribute to. I encourage everyone else using this driver to do the same.

I really like Riak as a database and these are all growing pains one needs to experience, I guess. I just expected it not to hurt as much as it did, :)