---
layout: post
title: "Autostarted Services"
date: 2012-07-26 22:11
comments: true
categories: [chef, debian, services]
---

It is quite common in Debian and Ubuntu that when installing a package
that provides a daemon, said daemon is started by the init script(s)
included in the package. This is a matter of
[Debian Policy](http://www.debian.org/doc/debian-policy/ch-opersys.html#s-sysvinit),
though I don't interpret that section to literally mean it is
required. However, it is
[common enough practice](http://wiki.debian.org/LSBInitScripts/) that [several people](https://www.varnish-cache.org/trac/ticket/1004)
have [asked](http://ask.debian.net/questions/how-to-prevent-daemons-from-starting-at-installation-time) (or [ranted](https://gist.github.com/1721727)) about [the topic](http://lists.opscode.com/sympa/arc/chef-dev/2012-02/msg00017.html).

The main issue of course is that the default configuration for the
software being installed may not be appropriate before starting up the
service and making it available on the network. Users of other Linux
distributions may be smugly smirking as their distribution doesn't
start the service on package installation.

This post isn't about that.

Instead, this post describes how this problem is resolved using
configuration management, specifically Chef. I'm also going to discuss
a couple nuances about service management, so watch carefully.

For the example service I'm going to use memcached, from the memcached
package. It is started on package installation as demonstrated:

```
vagrant@precise-housepub:~$ sudo apt-get install memcached
...
Setting up memcached (1.4.13-0ubuntu2) ...
Starting memcached: memcached.
vagrant@precise-housepub:~$ service memcached status
 * memcached is running
vagrant@precise-housepub:~$ ps awux | grep memcached
 memcache 15176  0.0  0.3 323212  1180 ?        Sl   04:32   0:00 /usr/bin/memcached -m 64 -p 11211 -u memcache -l 127.0.0.1
```

As we can see, the memcached service is started. Of course, it is
using the default configuration, which means that it has a very small
memory size, and listens on localhost. While the recipe would be very
simple:

```ruby
package "memcached"
```

This wouldn't be very useful for discussion, or practical use
purposes. For now, I'm going to post the entire recipe I'm going to
discuss, and then break it down.

```ruby
node.set['memcached']['memory_max'] = node['memory']['total'].to_i / 1024 * 0.65

package "memcached"

service "memcached" do
  supports :restart => true, :status => true
  action :enable
end

template "/etc/memcached.conf" do
  source "memcached.conf.erb"
  owner "root"
  group "root"
  mode 00644
  notifies :restart, "service[memcached]"
  variables(
    :memory_max => node['memcached']['memory_max']
    :ip_addr => node['ipaddress']
  )
end

service "memcached" do
  action :start
end
```

This recipe is fairly straightforward. First, it sets a node attribute
based on a calculation of the amount of memory installed in the
system. Then, it will install the memcached package. This of course
will start up the service with the unsuitable defaults already
discussed.

The first `service` resource occurance makes sure that the service is
enabled. This is the default behavior of the package manager, but this
also gives clear indication as to the intention of the recipe. Next,
the configuration file is managed. The exact content of the
`memcached.conf.erb` file isn't particularly important. Let us presume
that the variables passed in are what we care about - that we want to
use 65% of the system's total memory for memcached, and listen on the
default IP address. Maybe other tuning is happening, maybe not. Of
course, when the configuration is updated, we need to notify the
memcached service to restart.

Finally, the memcached service is started if it is not already
running. This occurs at the end to help remedy an issue where the
service might have been halted, or the configuration file was rendered
incorrectly (a typo?), so that we can correct such configuration
problems with Chef in a single subsequent run. This uses a feature of
Chef where resources can be declared multiple times with different
actions (or if desired, parameters).

The first time this recipe is run on a node, memcached will be
installed, started, configured, restarted. We're not aiming to prevent
the auto-start from occurring at all, but we do automate the
additional steps required for handling that easily.
