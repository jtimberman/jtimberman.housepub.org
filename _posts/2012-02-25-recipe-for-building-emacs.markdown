---
layout: post
title: "Recipe for Building Emacs"
date: 2012-02-25 23:51
comments: true
categories: [emacs, chef]
---

It is no secret that
[I use GNU Emacs](https://jtimberman.housepub.org/blog/2011/05/06/switching-to-gnu-emacs/)
as my default text editor. It is perhaps less evident but no less
relevant that I use Emacs 24. I really like the built-in color theme
support and the package management system for getting the various
modes I like to use.

Recently, I revamped my
[Emacs configuration](https://github.com/jtimberman/.emacs.d). This
post isn't about that topic. Instead, this post is about how I made
sure that all the systems I want to use Emacs on have the latest
version available.

Unfortunately, Emacs 24 is still unreleased, so it is not available as
the default package on the distributions I use for my personal systems
(Ubuntu/Debian flavors). I wrote a recipe to install Emacs
from source. This is easy enough to do, but since I already automate
everything on my home network with Chef, it was a natural fit for a
new recipe. I simply added this to my local `emacs` cookbook, in
`recipes/source24.rb`.

```ruby
srcdir = "#{Chef::Config[:file_cache_path]}/emacs-source"

%w{ git-core build-essential texinfo autoconf libncurses-dev }.each {|prereq| package prereq}

git "#{Chef::Config[:file_cache_path]}/emacs-source" do
  repository "git://git.savannah.gnu.org/emacs.git"
  action :checkout
end

bash "build emacs24" do
  cwd srcdir
  creates "#{srcdir}/src/emacs"
  code <<-EOH
    ./autogen.sh && \
    ./configure --without-x && \
    make bootstrap && \
    make 2>&1 >| make-#{node.name}-#{node['ohai_time']}
  EOH
end

execute "install emacs24" do
  cwd srcdir
  command "make install 2>&1 >| make-#{node.name}-#{node['ohai_time']}"
  creates "/usr/local/bin/emacs"
  only_if "#{srcdir}/src/emacs --version"
end
```

Then I updated my base role to replace `recipe[emacs]` with
`recipe[emacs::source24]` and ran Chef. It took about 25 minutes to do
the build, but now I have the same version of Emacs everywhere, and
there was much rejoicing.

And yes, you're absolutely right, I could just build a package and
install that. However, I don't want to set up and maintain a package
management repository for my small network, as
[easy](http://ckbk.it/reprepro) as [that](http://ckbk.it/apt) may be.

My OS X systems are a special case because I'm using Homebrew, but the
[homebrew cookbook](http://ckbk.it/homebrew) does not [yet?] support
install-time options, and I didn't spend the time adding support for
building the OS X Emacs w/ cocoa support from git. When I tackle that,
I'll make another post, so stay tuned!
