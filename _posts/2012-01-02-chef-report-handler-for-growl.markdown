---
layout: post
title: "Chef Report Handler for Growl"
date: 2012-01-02 20:37
comments: true
categories: [chef, plugins, osx]
---

A few weeks ago, I listened to the
[Changelog Podcast](http://thechangelog.com/post/11317828888/episode-0-6-8-growl-and-open-source-in-the-app-store-wit)
episode featuring Chris Forsythe, lead of the Growl project. I
actually don't^Wdidn't use Growl for a long time, because I really
disliked notifications of any kind, as they are distracting. However,
I do appreciate the project, and supporting them by purchasing Growl
on the App Store seemed totally reasonable.

Of course, buying the app means it was installed on the spot. I always
resisted it in the past because it was bundled with so many apps, but
I don't want unsolicited software to show up on my computer. Now, it
seemed a bit more natural, and I made a few configuration tweaks (I
should put those into Chef...). I'm actually happy to use it now,
especially since it has a nice network accessible API.

# Growlnotify

Shortly after I started using Growl on my work system, I looked into
ways I could get notifications fired off from the command-line after
long running processes finished. In particular, I wanted to get a
notification that `knife ec2 server create` was done. I found the
`growlnotify` program, which is available via
[homebrew](https://github.com/mxcl/homebrew). This is quite a nifty
tool, and I set it to use immediately, doing things like this:

```sh
knife ec2 server create \
  -f m1.small -I $lucid_small -x ubuntu \
  -r 'role[base],role[webserver]' && \
growlnotify -m "Finished launching 1 instance" || \
growlnotify -m "failed to launch instance"
```

I could kick off the server creation, switch focus to another
workspace, and then know via growl if the instance was created.

# Chef Handler

As many know, I
[manage my workstations with Chef](http://jtimberman.housepub.org/blog/2011/04/03/managing-my-workstations-with-chef/).
Chef has a pretty cool
[exception and report handler API](http://wiki.opscode.com/display/chef/Exception+and+Report+Handlers)
that has a lot of flexibility. I thought it would be fun to throw
together a simple report handler that would send a growl notification
after a Chef run. In this case, it will report the elapsed time of the
run if it was successful, or report an exception if it failed.

Using the handler is pretty straightforward. Install the
`chef-handler-growl` Gem, then configure chef-client (or solo).

```ruby
require "chef/handler/growl"
report_handlers << Chef::Handler::Growl.new
exception_handlers << Chef::Handler::Growl.new
```

Then run Chef, and see something like this:

![Chef Handler Growl](https://img.skitch.com/20120103-emxnsfht3557xjcp4rshxcufka.png)

The handler is available as a
[RubyGem](http://rubygems.org/gems/chef-handler-growl). You can also
view the [source](https://github.com/jtimberman/chef-handler-growl). I
created issues on the GitHub project for the two items on the roadmap, too.
