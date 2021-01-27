---
layout: post
title: "Disable AirDrop in Mac OS X Lion"
date: 2012-02-25 23:20
comments: true
categories: [chef, osx]
---

Mac OS X Lion introduced a new nifty feature called AirDrop. This
allows users on a local network to drag and drop files to each other
with Finder.

While it seems that this would be useful, there are security
implications. After looking through
[Google Search Results](https://www.google.com/?q=disable+airdrop#sclient=psy&hl=en&site=&source=hp&q=disable+airdrop&pbx=1&oq=disable+airdrop&aq=f&aqi=&aql=&gs_sm=3&gs_upl=0l0l0l884l0l0l0l0l0l0l0l0ll0l0&bav=on.2,or.r_gc.r_pw.,cf.osb&fp=2c07cc10bcee84ff&biw=1440&bih=786)
on the topic, I found some un-helpful information in a [random forum](http://forums.macrumors.com/showthread.php?t=1191359)
post (unsurprising). A little more
[review of the search results](http://derflounder.wordpress.com/2011/10/07/disabling-airdrop-from-the-command-line/)
resulted in finding the actual `defaults(1)` command to do so:

    defaults write com.apple.NetworkBrowser DisableAirDrop -bool YES

Naturally, I put this in a Chef recipe and applied it on all my Macs
post haste:

```ruby
  mac_os_x_userdefaults "Disable AirDrop" do
    domain "com.apple.NetworkBrowser"
    key "DisableAirDrop"
    value true
    type "bool"
  end
```

The resource above is available in my [mac_os_x cookbook](http://ckbk.it/mac_os_x).
