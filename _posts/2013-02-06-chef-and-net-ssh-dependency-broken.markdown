---
layout: post
title: "Chef and Net::SSH Dependency Broken"
date: 2013-02-06 10:34
comments: true
categories: [chef, ruby]
---

**2nd UPDATE**
[CHEF-3835](http://tickets.opscode.com/browse/CHEF-3835) was opened by
a member of the community; Chef versions 11.2.0 and 10.20.0 have been
released by Opscode to resolve the issue.

**UPDATE** Opscode is working on getting a new release of the Chef gem
  with updated version constraints.


# What Happened?

Earlier today (February 6, 2013), a new version of the various net-ssh
RubyGems were published. This includes:

* net-ssh 2.6.4
* net-ssh-multi 1.1.1
* net-ssh-gateway 1.1.1

Chef's dependencies have a pessimistic version constraint (`~>`) on
net-ssh 2.2.2.

# What's the Problem?

So what is the problem?

It appears to lie with net-ssh-gateway. The version of net-ssh-gateway
went from 1.1.0 (released in April 2011), to 1.1.1. It depends on
net-ssh. In net-ssh-gateway 1.1.0, the net-ssh version constraint was
`>= 1.99.1`, which is fine with Chef's constraint against `~> 2.2.2`.
However, in net-ssh-gateway 1.1.1, the net-ssh version constraint was
changed to `>= 2.6.4`, which is obviously a conflict with Chef's
constraint.

# What's the Solution?

So, how can we fix it?

One solution is to use the Opscode Omnibus Package for Chef. This
isn't a solution for everyone, of course, but it does include and
contain all the dependencies. This also doesn't help if one wishes to
install another gem that depends on Chef under the "Omnibus" Ruby
environment along with Chef, because the conflict will be found. For
example, to use the minitest-chef-handler gem for running
minitest-chef tests.

``` vagrant@ubuntu-12-04:~$ /opt/chef/embedded/bin/gem install
minitest-chef-handler ERROR: While executing gem ...
(Gem::DependencyError) Unable to resolve dependencies: net-ssh-gateway
requires net-ssh (>= 2.6.4) ```

Another solution is to relax / modify the constraint in Chef. This may
be okay, but as of *right now* we don't know if this will affect
anything in the way that Chef uses net-ssh. We have tickets related to
net-ssh version constraints in Chef:

* http://tickets.opscode.com/browse/CHEF-2977
* http://tickets.opscode.com/browse/CHEF-3156
