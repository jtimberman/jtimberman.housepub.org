---
layout: post
title: "Ruby In Ubuntu 11.10"
date: 2011-11-24 00:16
comments: true
categories: [ruby, debian]
---

I was playing around with Ubuntu 11.10 the other day, to explore some of the
changes that have happened to Ruby lately, and thought I'd share my findings.

First off, there are still Ruby 1.8(.7p352) packages. This is the
default you get with the `ruby` package. However, the `ruby1.9.1` has
Ruby 1.9.2p290 as the default version. This isn't the absolute latest
released (officially, 1.9.3 is the latest stable release), but it's a
widespread release that a lot of people are using in production.

The main package for getting the Ruby executables is `ruby1.9.1`. It
is now a "full" installation and includes:

* /usr/bin/gem1.9.1
* /usr/bin/rdoc1.9.1
* /usr/bin/rake1.9.1
* /usr/bin/erb1.9.1
* /usr/bin/ri1.9.1
* /usr/bin/ruby1.9.1
* /usr/bin/irb1.9.1
* /usr/bin/testrb1.9.1

Previous versions had these as separate packages, and you had to
remember to install everything to have a full Ruby installation
available.

In the package name and the above binaries, `1.9.1` indicates the Ruby
library compatibility, `1.9.2` is compatible with `1.9.1`.

The Debian alternatives system is used to set the default ruby, gem
and irb binaries in the $PATH as links to the 1.9.1 versions, e.g.:

```
% ls -l /usr/bin/ruby /etc/alternatives/ruby
lrwxrwxrwx 1 root root 18 2011-11-24 00:02 /etc/alternatives/ruby -> /usr/bin/ruby1.9.1
lrwxrwxrwx 1 root root 22 2011-11-24 00:02 /usr/bin/ruby -> /etc/alternatives/ruby
```

Users can also update RubyGems via the `gem update --system` command,
but a shell environment variable must be exported first.

``` bash
export REALLY_GEM_UPDATE_SYSTEM=yes
sudo -E gem update --system
```

The `-E` is significant so the variable is available via
`sudo`. Without this variable set, this command will warn noisily
that you should know what you're doing, and what this is up
to. Personally, I'm fine with RubyGems 1.3.7, as that is the version
that the Chef `gem_package` provider uses for the API. With the
default RubyGems, you also get binaries in `/usr/local/bin`, rather than
the awkward `/var/lib/ruby` location in earlier versions.

These changes are a result of work that has been going in
[Debian Wheezy by the Ruby packaging team](http://wiki.debian.org/Teams/Ruby/Packaging),
kicked off by
[Lucas Nussbaum](http://lists.debian.org/debian-devel/2011/03/msg00210.html).

That said, let's put this to use so we can have Chef running under the
Ruby 1.9.2 available via APT.

```
% sudo apt-get install ruby1.9.1 ruby1.9.1-dev build-essential
Reading package lists... Done
Building dependency tree
Reading state information... Done
build-essential is already the newest version.
Suggested packages:
  ruby1.9.1-examples ri1.9.1 graphviz
The following NEW packages will be installed:
  libruby1.9.1 ruby1.9.1 ruby1.9.1-dev
0 upgraded, 3 newly installed, 0 to remove and 1 not upgraded.
Need to get 0 B/5,027 kB of archives.
After this operation, 19.5 MB of additional disk space will be used.
Selecting previously deselected package libruby1.9.1.
(Reading database ... 141637 files and directories currently installed.)
Unpacking libruby1.9.1 (from .../libruby1.9.1_1.9.2.290-2_amd64.deb) ...
Selecting previously deselected package ruby1.9.1.
Unpacking ruby1.9.1 (from .../ruby1.9.1_1.9.2.290-2_amd64.deb) ...
Selecting previously deselected package ruby1.9.1-dev.
Unpacking ruby1.9.1-dev (from .../ruby1.9.1-dev_1.9.2.290-2_amd64.deb) ...
Processing triggers for man-db ...
Setting up libruby1.9.1 (1.9.2.290-2) ...
Setting up ruby1.9.1 (1.9.2.290-2) ...
update-alternatives: using /usr/bin/gem1.9.1 to provide /usr/bin/gem (gem) in auto mode.
update-alternatives: using /usr/bin/ruby1.9.1 to provide /usr/bin/ruby (ruby) in auto mode.
Setting up ruby1.9.1-dev (1.9.2.290-2) ...
Processing triggers for libc-bin ...
ldconfig deferred processing now taking place
```

One of Chef's dependency gems is `json`, which has native C
extensions, so the development headers for Ruby, and build tools are
required (though already installed on my system).

```
% ruby --version
ruby 1.9.2p290 (2011-07-09 revision 32553) [x86_64-linux]
% gem --version
1.3.7
```

Now, I can install Chef as a RubyGem.

```
% sudo gem install chef
Building native extensions.  This could take a while...
[Version 0.7.8] test suite cleanup (eliminated some race conditions related to queue.message_count)
Building native extensions.  This could take a while...
Successfully installed mixlib-config-1.1.2
Successfully installed mixlib-cli-1.2.2
Successfully installed mixlib-log-1.3.0
Successfully installed mixlib-authentication-1.1.4
Successfully installed yajl-ruby-1.1.0
Successfully installed systemu-2.2.0
Successfully installed ohai-0.6.10
Successfully installed mime-types-1.17.2
Successfully installed rest-client-1.6.7
Successfully installed bunny-0.7.8
Successfully installed json-1.5.2
Successfully installed polyglot-0.3.3
Successfully installed treetop-1.4.10
Successfully installed net-ssh-2.1.4
Successfully installed net-ssh-gateway-1.1.0
Successfully installed net-ssh-multi-1.1
Successfully installed erubis-2.7.0
Successfully installed moneta-0.6.0
Successfully installed highline-1.6.8
Successfully installed uuidtools-2.1.2
Successfully installed chef-0.10.4
21 gems installed
```

And running Chef just works:

```
% sudo chef-client
[Thu, 24 Nov 2011 14:02:38 -0700] INFO: *** Chef 0.10.4 ***
...
[Thu, 24 Nov 2011 14:02:50 -0700] INFO: Chef Run complete in 9.862972181 seconds
```

The results of the node object on the Chef Server:

```
% knife node show virt1test -a languages.ruby
languages.ruby:
  bin_dir:        /usr/bin
  gem_bin:        /usr/bin/gem1.9.1
  gems_dir:       /var/lib/gems/1.9.1
  host:           x86_64-pc-linux-gnu
  host_cpu:       x86_64
  host_os:        linux-gnu
  host_vendor:    pc
  platform:       x86_64-linux
  release_date:   2011-07-09
  ruby_bin:       /usr/bin/ruby1.9.1
  target:         x86_64-pc-linux-gnu
  target_cpu:     x86_64
  target_os:      linux
  target_vendor:  pc
  version:        1.9.2
```

Overall, I'm happy with the changes that the Debian Ruby Packaging
Team has made, and I'm glad to see these come to Ubuntu in
11.10+. It's not the absolute latest version available, but this is
good forward progress for binary Linux distributions. I also like that
the alternatives system is used, so users could choose to install and
use other Ruby interpreters. Hopefully these changes quell some of the
arguments about Debian and Ubuntu and their handling of Ruby.
