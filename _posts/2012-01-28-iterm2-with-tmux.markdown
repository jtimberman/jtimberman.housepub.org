---
layout: post
title: "iTerm2 with tmux"
date: 2012-01-28 15:03
comments: true
categories: [development]
---

A new "which tool is best" battle is raging in the internets amongst
developers and system administrators. The contestants are [screen](http://www.gnu.org/software/screen/)
and [tmux](http://tmux.sourceforge.net/), and the
[jury is still out](https://www.google.com/search?q=tmux+vs+screen).
This is very much an argument over what color to paint the bikeshed,
but with the latest version of
[iTerm2](http://www.iterm2.com/), I think tmux is even more
compelling. Personally, I chose tmux awhile ago.

At my [day job](http://opscode.com), I worked with a customer that
uses tmux for remote pairing between developers. At the time, tmux had
better customizability, and better split-pane support (screen didn't
yet have vertical split). I stuck with tmux ever since, and was very
pleased when an iTerm2 update announced integration with tmux.

# iTerm2

For those who aren't aware, iTerm2 is an alternate terminal program
for Mac OS X. It is actually an updated codebase from the original,
[iTerm](http://iterm.sourceforge.net/), which is effectively
[unmaintained](http://iterm.sourceforge.net/news.shtml). iTerm2 offers
a lot of excellent features like split panes, Growl support, and
[many more](http://www.iterm2.com/#/section/features).

One of the excellent new features is integration with tmux.

# iTerm2's tmux integration

If you already have iTerm2 installed, you may have seen the update
check prompt you to update. You also need to install a special version
of tmux that has the integration patched. The iTerm2 author is working
with the tmux author to get this into the latest tmux codebase, so
hopefully the custom compiled version won't be necessary soon.

Using the new feature is relatively straightforward. Start up iTerm2
like normal. Then run `tmux -C` to open a new iTerm2 window that works
like tmux.

![Launch tmux in iTerm2](http://img.skitch.com/20120128-cxjbh9hf9feagn5p5ieg574uce.png)

Use the tmux menu in iTerm2 to open new windows in tmux. Note that there
are keyboard shortcuts for each of these, and they are not the same as
the tmux window commands.

![Use tmux menu to open buffers](http://img.skitch.com/20120128-diexhy69d8t3b9da6g65yjmsum.png)

You can also attach to a tmux session running in iTerm2. In the
screenshot, this is running on the same system, for example purposes.
However, since OS X has SSH, this can be useful if you want to SSH to
another system in the local network and connect to the running
session. For example, the system shown below is my wife's iMac over
screensharing, but I wouldn't need to use screenshare (or participate
in its lag) to connect to this anymore. The same holds true for
connecting to my work laptop if necessary.

![Attach to tmux session in iTerm2](http://img.skitch.com/20120128-txr67q3jftmcw26n84jm5hpgm.png)

In this final example screenshot, you can see that I have multiple
panes split in one iTerm2 tab. These correspond to the split windows
in the attached tmux in the other window. Also, the two tabs in the
iTerm2 window are separate tmux windows (`0:zsh` and `1:zsh`).

![iTerm2 panes are panes in tmux](http://img.skitch.com/20120128-miaa1dkeatt2hebxcxst8sydy1.png)

And now, I can SSH to that system and attach to the tmux session
started by iTerm2.

![SSH to remote and run tmux attach](http://img.skitch.com/20120129-gjgajqnekq93da59m4r86ewh2k.png)

![tmux is attached to iTerm2 session](http://img.skitch.com/20120129-j2x4iir5557nt68n5jmr64f8dp.png)

# Automating Installation with Chef

Installing OS X apps is quite easy, but I automate them
[with Chef](/blog/2011/04/03/managing-my-workstations-with-chef/)
anyway. While it is a simple "install update and restart", with a
couple commands to install the update, I do have three systems I want
this on. I updated my
[iterm2 cookbook](http://community.opscode.com/cookbooks/iterm2) to
support installing the tmux integration for iTerm2. This is disabled
by default, so it needs to be enabled via a node attribute. For
example, I have this in my `workstation` role applied to my OS X
workstations.

``` ruby
name "workstation"
description "Mac OS X workstations"
run_list("recipe[tmux]")
default_attributes(
  "iterm2" => {
    "tmux_enabled" => true
  }
)
```

Check out the iterm2 cookbook's README for more information.
