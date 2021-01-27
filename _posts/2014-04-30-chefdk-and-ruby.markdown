---
layout: post
title: "ChefDK and Ruby"
date: 2014-04-30 22:32:18 -0600
comments: true
categories: [chef, ruby]
---

Recently, Chef [released ChefDK](http://www.getchef.com/blog/2014/04/15/chef-development-kit/), the "Chef Development Kit." This is a self-contained package of everything required to run Chef, work with Chef cookbooks, and includes the best of breed community tools, test frameworks, and other utility programs that are commonly used when working with Chef in infrastructure as code. [ChefDK version 0.1.0 was released last week](http://www.getchef.com/blog/2014/04/29/chefdk-0-1-0-released/). A new feature mentioned in the [README.md](https://github.com/opscode/chef-dk/blob/master/README.md#using-chefdk-as-your-primary-development-environment) is very important, in my opinion.

**Using ChefDK as your primary development environment**

What does that mean?

It means that if the only reason you have Ruby installed on your local system is to do Chef development or otherwise work with Chef, *you no longer have to maintain a separate Ruby installation*. That means you won't need any of these:

* [rbenv](https://github.com/sstephenson/rbenv)
* [rvm](http://rvm.io)
* [chruby](https://github.com/postmodern/chruby) (*note)
* "system ruby" (e.g., OS X's included /usr/bin/ruby, or the `ruby` package from your Linux distro)
* [poise ruby](https://github.com/poise/poise-ruby))

(*note: You can optionally use chruby with ChefDK if it's part of your workflow and you have other Rubies installed.)

Do not misunderstand me: These are all extremely good solutions for getting and using Ruby on your system. They definitely have their place if you do other Ruby development, such as [web applications](http://www.sinatrarb.com/). This is especially true if you have to work with multiple versions of Ruby. However, if you're like me and mainly use Ruby for Chef, then ChefDK has you covered.

In this post, I will describe how I have set up my system with ChefDK, and use its embedded Ruby by default.

## Getting Started

Download ChefDK from the [downloads page](http://www.getchef.com/downloads/chef-dk/). At the time of this blog post, the available builds are limited to OS X and Linux (Debian/Ubuntu or RHEL), but Chef is [working on Windows packages](http://www.getchef.com/blog/2014/05/06/chefdk-for-windows-progress-update/).

For example, here's what I did on my Ubuntu 14.04 system:

```
wget https://opscode-omnibus-packages.s3.amazonaws.com/ubuntu/13.10/x86_64/chefdk_0.1.0-1_amd64.deb
sudo dpkg -i /tmp/chefdk_0.1.0-1_amd64.deb
```

OS X users will be happy to know that the download is a .DMG, which includes a standard OS X .pkg (complete with developer signing). Simply install it like many other products on OS X.

For either Linux or OS X, in [omnibus](https://github.com/opscode/omnibus-ruby/) fashion, the post-installation creates several symbolic links in `/usr/bin` for tools that are included in ChefDK:

```
% ls -l /usr/bin | grep chefdk
lrwxrwxrwx 1 root root 21 Apr 30 22:13 berks -> /opt/chefdk/bin/berks
lrwxrwxrwx 1 root root 20 Apr 30 22:13 chef -> /opt/chefdk/bin/chef
lrwxrwxrwx 1 root root 26 Apr 30 22:13 chef-apply -> /opt/chefdk/bin/chef-apply
lrwxrwxrwx 1 root root 27 Apr 30 22:13 chef-client -> /opt/chefdk/bin/chef-client
lrwxrwxrwx 1 root root 26 Apr 30 22:13 chef-shell -> /opt/chefdk/bin/chef-shell
lrwxrwxrwx 1 root root 25 Apr 30 22:13 chef-solo -> /opt/chefdk/bin/chef-solo
lrwxrwxrwx 1 root root 25 Apr 30 22:13 chef-zero -> /opt/chefdk/bin/chef-zero
lrwxrwxrwx 1 root root 23 Apr 30 22:13 fauxhai -> /opt/chefdk/bin/fauxhai
lrwxrwxrwx 1 root root 26 Apr 30 22:13 foodcritic -> /opt/chefdk/bin/foodcritic
lrwxrwxrwx 1 root root 23 Apr 30 22:13 kitchen -> /opt/chefdk/bin/kitchen
lrwxrwxrwx 1 root root 21 Apr 30 22:13 knife -> /opt/chefdk/bin/knife
lrwxrwxrwx 1 root root 20 Apr 30 22:13 ohai -> /opt/chefdk/bin/ohai
lrwxrwxrwx 1 root root 23 Apr 30 22:13 rubocop -> /opt/chefdk/bin/rubocop
lrwxrwxrwx 1 root root 20 Apr 30 22:13 shef -> /opt/chefdk/bin/shef
lrwxrwxrwx 1 root root 22 Apr 30 22:13 strain -> /opt/chefdk/bin/strain
lrwxrwxrwx 1 root root 24 Apr 30 22:13 strainer -> /opt/chefdk/bin/strainer
```

These should cover the 80% use case of ChefDK: using the various Chef and Chef Community tools so users can follow their favorite workflow, without shaving the yak of managing a Ruby environment.

But, as I noted, and the thesis of this post, is that one could use this Ruby environment included in ChefDK as their own! So where is that?

## ChefDK's Ruby

Tucked away in every "omnibus" package is a directory of "embedded" software - the things that were required to meet the end goal. In the case of Chef or ChefDK, this is Ruby, openssl, zlib, libpng, and so on. This is a fully contained directory tree, complete with lib, share, and yes indeed, bin.

```
% ls /opt/chefdk/embedded/bin
(there's a bunch of commands here, trust me)
```

Of particular note are `/opt/chefdk/embedded/bin/ruby` and `/opt/chefdk/embedded/bin/gem`.

To use ChefDK's Ruby as default, simply edit the `$PATH`.

```
export PATH="/opt/chefdk/embedded/bin:${HOME}/.chefdk/gem/ruby/2.1.0/bin:$PATH"
```

Add that, or its equivalent, to a login shell profile/dotrc file, and rejoice. Here's what I have now:

```
$ which ruby
/opt/chefdk/embedded/bin/ruby
$ which gem
/opt/chefdk/embedded/bin/gem
$ ruby --version
ruby 2.1.1p76 (2014-02-24 revision 45161) [x86_64-linux]
$ gem --version
2.2.1
$ gem env
RubyGems Environment:
  - RUBYGEMS VERSION: 2.2.1
  - RUBY VERSION: 2.1.1 (2014-02-24 patchlevel 76) [x86_64-linux]
  - INSTALLATION DIRECTORY: /opt/chefdk/embedded/lib/ruby/gems/2.1.0
  - RUBY EXECUTABLE: /opt/chefdk/embedded/bin/ruby
  - EXECUTABLE DIRECTORY: /opt/chefdk/embedded/bin
  - SPEC CACHE DIRECTORY: /home/ubuntu/.gem/specs
  - RUBYGEMS PLATFORMS:
    - ruby
    - x86_64-linux
  - GEM PATHS:
     - /opt/chefdk/embedded/lib/ruby/gems/2.1.0
     - /home/ubuntu/.chefdk/gem/ruby/2.1.0
  - GEM CONFIGURATION:
     - :update_sources => true
     - :verbose => true
     - :backtrace => false
     - :bulk_threshold => 1000
     - "install" => "--user"
     - "update" => "--user"
  - REMOTE SOURCES:
     - https://rubygems.org/
  - SHELL PATH:
     - /opt/chefdk/embedded/bin
     - /home/ubuntu/.chefdk/gem/ruby/2.1.0/bin
     - /usr/local/sbin
     - /usr/local/bin
     - /usr/sbin
     - /usr/bin
     - /sbin
     - /bin
     - /usr/games
     - /usr/local/games
```

Note that this is the current stable release of Ruby, version 2.1.1 patchlevel 76, and the {almost} latest version of RubyGems, version 2.2.1. Also note the Gem paths - the first is the embedded gems path, which is where gems installed by `root` with the `chef gem` command will go. The other is in my home directory - ChefDK is set up so that gems can be installed as a non-root user within the `~/.chefdk/gems` directory.

## Installing Gems

Let's see this in action. Install a gem using the `gem` command.

```
$ gem install knife-solve
Fetching: knife-solve-1.0.1.gem (100%)
Successfully installed knife-solve-1.0.1
Parsing documentation for knife-solve-1.0.1
Installing ri documentation for knife-solve-1.0.1
Done installing documentation for knife-solve after 0 seconds
1 gem installed
```

And as I said, this will be installed in the home directory:

```
$ gem content knife-solve
/home/ubuntu/.chefdk/gem/ruby/2.1.0/gems/knife-solve-1.0.1/LICENSE
/home/ubuntu/.chefdk/gem/ruby/2.1.0/gems/knife-solve-1.0.1/README.md
/home/ubuntu/.chefdk/gem/ruby/2.1.0/gems/knife-solve-1.0.1/Rakefile
/home/ubuntu/.chefdk/gem/ruby/2.1.0/gems/knife-solve-1.0.1/lib/chef/knife/solve.rb
/home/ubuntu/.chefdk/gem/ruby/2.1.0/gems/knife-solve-1.0.1/lib/knife-solve.rb
/home/ubuntu/.chefdk/gem/ruby/2.1.0/gems/knife-solve-1.0.1/lib/knife-solve/version.rb
```

## Using Bundler

ChefDK also includes [bundler](http://bundler.io/). As a "non-Chef, Ruby use case", I installed octopress for this blog.

```
% bundle install --path vendor --binstubs
Fetching gem metadata from https://rubygems.org/.......
Fetching additional metadata from https://rubygems.org/..
Installing rake (0.9.6)
Installing RedCloth (4.2.9)
Installing chunky_png (1.2.9)
Installing fast-stemmer (1.0.2)
Installing classifier (1.3.3)
Installing fssm (0.2.10)
Installing sass (3.2.12)
Installing compass (0.12.2)
Installing directory_watcher (1.4.1)
Installing haml (3.1.8)
Installing kramdown (0.14.2)
Installing liquid (2.3.0)
Installing maruku (0.7.0)
Installing posix-spawn (0.3.6)
Installing yajl-ruby (1.1.0)
Installing pygments.rb (0.3.7)
Installing jekyll (0.12.1)
Installing rack (1.5.2)
Installing rack-protection (1.5.0)
Installing rb-fsevent (0.9.3)
Installing rdiscount (2.0.7.3)
Installing rubypants (0.2.0)
Installing sass-globbing (1.0.0)
Installing tilt (1.4.1)
Installing sinatra (1.4.3)
Installing stringex (1.4.0)
Using bundler (1.5.2)
Updating files in vendor/cache
  * classifier-1.3.3.gem
  * fssm-0.2.10.gem
  * sass-3.2.12.gem
  * compass-0.12.2.gem
  * directory_watcher-1.4.1.gem
  * haml-3.1.8.gem
  * kramdown-0.14.2.gem
  * liquid-2.3.0.gem
  * maruku-0.7.0.gem
  * posix-spawn-0.3.6.gem
  * yajl-ruby-1.1.0.gem
  * pygments.rb-0.3.7.gem
  * jekyll-0.12.1.gem
  * rack-1.5.2.gem
  * rack-protection-1.5.0.gem
  * rb-fsevent-0.9.3.gem
  * rdiscount-2.0.7.3.gem
  * rubypants-0.2.0.gem
  * sass-globbing-1.0.0.gem
  * tilt-1.4.1.gem
  * sinatra-1.4.3.gem
  * stringex-1.4.0.gem
Your bundle is complete!
It was installed into ./vendor
```

Then I can use for example the rake task to preview things while writing this post.

```
$ ./bin/rake preview
Starting to watch source with Jekyll and Compass. Starting Rack on port 4000
directory source/stylesheets/
   create source/stylesheets/screen.css
[2014-05-07 21:46:35] INFO  WEBrick 1.3.1
[2014-05-07 21:46:35] INFO  ruby 2.1.1 (2014-02-24) [x86_64-linux]
[2014-05-07 21:46:35] INFO  WEBrick::HTTPServer#start: pid=10815 port=4000
```

## Conclusion

I've used Chef before it was even released. As the project has evolved, and as the Ruby community around it has established new best practices installing and maintaining Ruby development environments, I've followed along. I've used all the version managers listed above. I've spent untold hours getting the right set of gems installed just to have to upgrade everything again and debug my workstation. I've written blog posts, [wiki](http://wiki.opscode.com) pages, and helped countless users do this on their own systems.

Now, we have an all-in-one environment that provides a great solution. Give ChefDK a whirl on your workstation - I think you'll like it!
