---
layout: post
title: "Last Check-in Time for Nodes"
date: 2013-02-16 20:14
comments: true
categories: [chef, plugins]
---

This one liner uses the knife exec sub-command to iterate over all the
node objects on the Chef Server, and print out their `ohai_time`
attribute in a human readable format.

```
knife exec -E 'nodes.all {|n| puts "#{n.name} #{Time.at(n[:ohai_time])}"}'
```

Let's break this up a little.

```
knife exec -E
```

The exec plugin for knife executes a script or the given string of
Ruby code in the same context as `chef-shell` (or `shef` in Chef 10
and earlier) if you start it up in it's "main" context. Since it is
knife, it will also use your `.chef/knife.rb` settings, so it knows
about your user, key and Chef Server.

```
nodes.all
```

The `chef-shell` main context has helper methods to access the
corresponding endpoints in the Chef Server API. Clearly we're working
with "nodes" here, and the `#all` method returns all the node objects
from the Chef Server. This differs from search in that there's a
commit delay between the time when data is saved to the server, and
the data is indexed by Solr. This is usually a few seconds, but
depending on various factors like the hardware you're using, how many
nodes are converging, etc, it can take longer.

Anyway, we can pass a block to nodes.all and do something with each
node object. The example above is a oneliner, so let's make it more
readable.

```ruby
nodes.all do |n|
  puts "#{n.name} #{Time.at(n[:ohai_time])}"
end
```

We're simply going to use `n` as the iterator for each node object,
and we'll print a string about the node. The `#{}`'s in the string to
print with puts is Ruby string interpolation. That is, everything
inside the braces is a Ruby expression. First, the `Chef::Node` object
has a method, `#name`, that returns the node's name. This is usually
the FQDN, but depending on your configuration (`node_name` in
`/etc/chef/client.rb` or using the `-N` option for `chef-client`), it
could be something else. Then, we're going to use the node's
`ohai_time` attribute. Every time Chef runs and it gathers data about
the node with Ohai, it generates the `ohai_time` attribute, which is
the Unix epoch of the timestamp when Ohai ran. When Chef saves the
node data at the end of the run, we know approximately the last time
the node ran Chef. In this particular string, we're converting the
Unix epoch, like `1358962351.444405` to a human readable timestamp
like `2013-01-23 10:32:31 -0700`.

Of course, you can get similar data from the Chef Server by using
`knife status`:

```
knife status
```

The `ohai_time` attribute will be displayed as a relative time, e.g.,
"585 hours ago." It will include some more data about the nodes like IP's. This
uses Chef's search feature, so you can also pass in a query:

```
knife status "role:webserver"
```

The `knife exec` example is simple, but you can get a lot more data
about the nodes than what `knife status` reports.

In either case, `ohai_time` isn't 100% accurate, since it is generated
at the beginning of the run, and depending on what you're doing with
Chef on your systems, it can take a long time before the node data is
saved. However, it's close enough for many use cases.

If more detailed or completely accurate information about the Chef run
is required for your purposes, you should use a
[report handler](http://docs.opscode.com/chef/essentials_handlers.html),
which does have more data about the run available, including whether
the run was successful or not.
