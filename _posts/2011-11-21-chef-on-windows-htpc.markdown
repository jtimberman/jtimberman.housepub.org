---
layout: post
title: "Chef on Windows HTPC"
date: 2011-11-21 07:15
comments: true
categories: [chef, windows]
---

Over the past 20 years, I always had a Windows (or DOS!) PC as the
main system I use at home. The primary purpose was for gaming,
although building my own systems is also a hobby. In 2007, I wanted to
build a new system to use as a home theater PC. Originally, I built it
with Windows Vista - Windows Media Center *was* Vista's killer app! I
painstakingly installed software, tweaked system settings and tuned
the registry. Then Windows 7 came along with further
improvements. After almost 20 years of reinstalling from scratch for
new versions of Windows, I treated this upgrade as no
different.

Unfortunately, I didn't capture all the system tweaks and settings. I
didn't get all the software reinstalled. I did what I remembered, and
W7 did have some improvements that made extra software and tweaks
unnecessary. The thing I was lacking in the rebuild, and up until
recently, was configuration management for Windows.

## Enter Chef

Recently, my employer Opscode
[announced broader Windows support](http://www.opscode.com/press-releases/opscode-delivers-cloud-infrastructure-automation-to-windows-environments/)
for Chef. The awesome Chef community, and our development team have
added some key features to Chef in the form of core libraries and
cookbooks. While I did get Chef running on my HTPC a few months ago, I
had only used it to test out the
[powershell cookbook](http://community.opscode.com/cookbooks/powershell),
and also wrote a
[silverlight cookbook](http://community.opscode.com/cookbooks/silverlight)
as a proof of concept. In the mean time, the
[windows cookbook](http://community.opscode.com/cookbooks/windows)
cookbook received some serious leveling up thanks to
[Seth Chisamore](http://twitter.com/schisamo).

## Windows Cookbook

The Windows cookbook has a number of excellent "primitives" in the
form of libraries and resources. You can read all about that on
[The Chef Wiki](http://wiki.opscode.com/display/chef/Opscode+LWRP+Resources#OpscodeLWRPResources-windows). For
now, I'll talk about how I captured some of the configuration so I
don't have to remember next time (Windows 8, coming soon...). Of
interest here are the `windows_registry` and `windows_auto_run` resources.

## My HTPC's cookbooks

I wrote two cookbooks this weekend. First, is a general "Windows Media
Center" cookbook called, aptly, "`wmc`". This will capture the
specific registry and other configuration related to Windows Media
Center itself. The idea with this one is it manages only Windows Media
Center. It's not ready for release, but I'll post some snippets
of code here as I go.

The second cookbook I created is specifically for the Home Theater PC
itself, named `htpc`. This is going to contain all my preferences and
biases of how I want my HTPC to look like in order to provide
entertainment in my house.

## Windows Media Center (wmc)

First and foremost, I want to make sure that Windows Media Center is
running at boot time, and the `windows_auto_run` resource fits the
bill. This resource modifies the registry to update the key
`HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`, using the
`windows_registry` resource. It does operate at the *machine* level,
not just the current user, but I only have one user on my HTPC, so
it's okay. Here's the resource:

``` ruby
windows_auto_run "Windows Media Center" do
  program "RunDLL32.exe C:/Windows/ehome/ehuihlp.exe,BootMediaCenter"
  not_if { Registry.value_exists?(AUTO_RUN_KEY, 'Windows Media Center') }
  action :create
end
```

Next, I want to tweak a few other registry settings. Some of these I
discovered through an
[MSDN blog post](http://blogs.msdn.com/b/astebner/archive/2006/04/29/586961.aspx),
others are from setting up WMC itself and observing the "HKCU"
registry. Not the most sophisticated way, but I don't see a great
"diff" tool for the registry. Because the name of the HKCU and HKLM keys for WMC are
long, I set them to a couple local variables, via `cookbooks/attributes/default.rb`.

``` ruby
default['wmc']['reg_hkcu'] = 'HKCU\Software\Microsoft\Windows\CurrentVersion\Media Center'
default['wmc']['reg_hklm'] = 'HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Media Center'
```

Then in the recipe, I simply assign local variables.

``` ruby
hkcu = node['wmc']['reg_hkcu']
hklm = node['wmc']['reg_hklm']
```

Now, for the registry changes. I use a 30 second skip forward and back
with my remote during recorded TV shows, so I can avoid commercials.

``` ruby
windows_registry "#{hkcu}\\Settings\\VideoSettings" do
  values(
    'SkipAheadInterval' => 0x7148,
    'InstantReplayInterval' => 0x7148
  )
end
```

I also optimize the display for LCD/Plasma (I have a RP DLP, but I
digress).

``` ruby
windows_registry hkcu do
  values 'TVSkin' => 0
end
```

For now, that's all the `wmc` cookbook has going for it. I do have
plans to enable additional features of WMC, install useful plugins,
and improve playback quality and performance. Mainly, I need to find
my old notes :-).

## Home Theater PC (htpc)

The `htpc` cookbook will contain additional tweaks and settings for
how I want my HTPC configured. I don't know if I'll release it
publicly, as it is pretty specific to me, but it might be useful as
examples for others, so stay tuned to my
[GitHub account](https://github.com/jtimberman) for updates.

First up are some local performance tweaks. These are done to
Explorer(.exe), and I don't remember the source.

``` ruby
windows_registry "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Policies\\Explorer" do
  values(
    'NoLowDiskSpaceChecks' => 1,
    'NoResolveSearch' => 1,
    'NoResolveTrack' => 1
  )
end
```

Second, since this system is where all my music files live, I have
iTunes installed for
[Home Sharing](http://support.apple.com/kb/HT3819). It doesn't do much
good if iTunes isn't running, however, so I use `windows_auto_run` to
start it up.

``` ruby
windows_auto_run "iTunes" do
  program "C:/Program\ Files/iTunes/iTunes.exe"
  not_if { Registry.value_exists?(AUTO_RUN_KEY, "iTunes") }
  action :create
end
```

It's pretty simple so far, but I have some plans in store for my HTPC cookbook.

## HTPC Cookbook Plans

I do plan to add some new features to the `htpc` cookbook. First, I
want to make sure all the software I need is installed. While the
system mainly sits in WMC (with iTunes in the background), I do have
some additional software and utilities that I like to have for
managing my media library.

* iTunes - I already have the software installed, but it would be nice
  to not have to do that during system setup (Windows 8, I say...).
* Google Chrome - there's no abc.com, nbc.com, etc plugins for WMC
  (yet?), so I need a browser to go to the respective web sites to
  catch up on older episodes.
* CDBurnerXP - decent CD/DVD burning software, rarely used but useful
  for backups.
* Logitech Harmony Remote Software - I have a
  [Harmony 700 universal remote](http://www.logitech.com/en-us/remotes/universal-remotes/devices/6063)
  (and it's awesome!). This one might be tricky, but really I just
  want to make sure the software is installed. Configuration profiles
  are stored in an online support account with Logitech as far as I'm
  aware.
* Netflix plugin for WMC - This will probably go into the `wmc`
  cookbook, though.
* Additional performance tweaks - There are a whole slew of tweaks
  that I *know* I made, I just don't remember them all. These will be
  added with `windows_registry` resources.
* Manage services - Everyone who has a gaming PC knows about
  [Black Viper's Windows Service Guide(s)](http://www.blackviper.com/).
* Windows Features - With the `windows_feature` resource, Chef can
  manage core features included in Windows installations.
* Logging, monitoring - What kind of system administrator would I be
  if I didn't also monitor my system for performance? :-)

Some of the things I want to manage on my HTPC are covered by other
cookbooks I have, or exist on the
[Chef Community Site](http://community.opscode.com).

* Steam - This system is still capable of running some games, like
  StarCraft II, TeamFortress 2 and Left 4 Dead (1 & 2). I have a Steam
  cookbook that works on OS X, it just isn't released (yet).
* Teamspeak 3 - I already have a
  [teamspeak3 cookbook](http://community.opscode.com/cookbooks/teamspeak3),
  I'll just need to add installation of the client for Windows there.
* [A text editor](http://gnu.org/s/emacs) - Since I have games, and
  games have configuration files, it makes sense to ... wait, I should
  put those in Chef too! Or at least, in a Version Control System and
  check out the repository...
* [Git](http://git-scm.org) - For the above. This will likely be a
  contribution to Opscode's
  [git cookbook](http://community.opscode.com/cookbooks/git). Though
  I'll still install a [text editor](http://notepad-plus-plus.org) of
  [some kind](http://www.gnome.org/projects/gedit).

These changes, updates and releases will be announced on this blog,
 mainly through my [twitter](https://twitter.com/jtimberman) and
 [Google+](https://plus.google.com/100567271038100401523/)
 accounts. Stay tuned!
