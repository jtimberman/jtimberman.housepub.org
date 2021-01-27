---
layout: post
title: "Xcode Command Line Tools"
date: 2012-02-26 00:07
comments: true
categories: [chef, osx, development]
---

Recently, Apple did the most awesome thing for non-Xcode developers.

They made the development tools available as a
[standalone package!](https://developer.apple.com/library/ios/#documentation/DeveloperTools/Conceptual/WhatsNewXcode/Articles/xcode_4_3.html)

This is awesome news for those of us who only have^Whad Xcode
installed to install RubyGems that compile native extensions, or for
installing software with Homebrew, MacPorts or similar.. You can
download them by logging into the
[Developer Download](https://developer.apple.com/downloads) site.

This appears to be work started by
[Kenneth Reitz](http://kennethreitz.com/xcode-gcc-and-homebrew.html)
with his
[OSX GCC Installer](https://github.com/kennethreitz/osx-gcc-installer/)
project. I did try that project out, but ran into issues I didn't
resolve right away, so I reverted to using Xcode proper. However with
the package from Apple I don't seem to have any issues so far.

If you already have Xcode installed, you may want to remove it first.

```sh
sudo /Developer-3.2.6/Library/uninstall-devtools
sudo /Developer/Library/uninstall-devtools --mode=all /thx
```

My understanding is that you can remove the `/Developer*`
director(y|ies) when complete. I had ollllld Xcode on the system where
I first did this.

Next, download and install the package from Apple. It's about 170M and
takes only a couple minutes to install; sorry I don't have a Chef
recipe for this ;).

I did run into an issue with Homebrew where it wasn't finding the
right gcc binary. I had to run the following commands to fix
[that issue](https://github.com/mxcl/homebrew/issues/10245).

```sh
sudo xcode-select -switch /usr/bin
sudo ln -sf /usr/bin/llvm-gcc-4.2 /usr/bin/gcc-4.2
```

You wouldn't think that something like an 8G installation would matter
in 2012. However, disk space is a precious commodity on MacBook Airs
and systems that have SSDs as the root volume. This is very welcome
change for me, especially since it means that future Mac OS X
installations do not require a large download before I can start doing
things that get my
[system ready to use](http://jtimberman.housepub.org/blog/2011/04/03/managing-my-workstations-with-chef/).
