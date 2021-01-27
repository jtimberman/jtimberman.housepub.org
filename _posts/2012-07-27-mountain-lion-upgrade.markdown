---
layout: post
title: "Mountain Lion Upgrade"
date: 2012-07-27 16:25
comments: true
categories: [osx, workstation]
---

I upgraded my work laptop to Mountain Lion today. It was not as smooth
as previous OS X upgrades have been for me, despite my efforts in
managing my workstation(s) with Chef.

I received a replacement laptop for the one that was damaged at
ChefConf a couple weeks ago (champagne spill - long story, maybe for
another blog post). As this is a new laptop, it is eligible for the
free Mountain Lion upgrade. So on ML release day, I submitted my
information to Apple for the redemption code, which I received this
morning. As I'm actually on vacation this week, I thought there was no
better time to upgrade.

You may recall that I have managed my
[workstations](/blog/2011/04/03/managing-my-workstations-with-chef/)
[with Chef](/blog/2011/09/04/update-to-managing-my-workstations/) for
quite some time. This has been all well and good so far, though the
Mountain Lion installation didn't seem to like something in my
preferences along the way.

After the installation finished and the system rebooted, I logged in,
expecting to be greeted with my already configured system. This was
not the case, however! My desktop was that light grey of the OS X boot
screen, and the Dock was not running at all. I put on my
[sysadmin hat](http://sysadminday.com) (like I ever take it off?!),
and started debugging. I found the issue pretty quickly.

```
Jul 27 08:42:35 champagne.local Dock[423]: -[__NSCFBoolean isEqualToString:]: unrecognized selector sent to instance 0x7fff75d0fab0
Jul 27 08:42:35 champagne.local Dock[423]: *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[__NSCFBoolean isEqualToString:]: unrecognized selector sent to instance 0x7fff75d0fab0'
        *** First throw call stack:
        (
                0   CoreFoundation                      0x00007fff8fca9716 __exceptionPreprocess + 198
                1   libobjc.A.dylib                     0x00007fff8dc63470 objc_exception_throw + 43
                2   CoreFoundation                      0x00007fff8fd3fd5a -[NSObject(NSObject) doesNotRecognizeSelector:] + 186
                3   CoreFoundation                      0x00007fff8fc97c3e ___forwarding___ + 414
                4   CoreFoundation                      0x00007fff8fc97a28 _CF_forwarding_prep_0 + 232
                5   Dock                                0x000000010da92786 Dock + 681862
                6   Dock                                0x000000010d9f10b2 Dock + 20658
                7   Dock                                0x000000010dab9aed Dock + 842477
                8   libdyld.dylib                       0x00007fff852c17e1 start + 0
        )
Jul 27 08:42:35 champagne com.apple.launchd.peruser.501[267] (com.apple.Dock.agent[423]): Job appears to have crashed: Abort trap: 6
Jul 27 08:42:35 champagne com.apple.launchd.peruser.501[267] (com.apple.Dock.agent): Throttling respawn: Will start in 1 seconds
Jul 27 08:42:35 champagne.local ReportCrash[302]: Saved crash report for Dock[423] version 1.8 (1168) to /Users/jtimberman/Library/Logs/DiagnosticReports/Dock_2012-07-27-084235_champagne.crash
```

This happened every second, constantly crashing and restarting, and
generating a new crash report. What is the problem? Well, let's look
at one of the reports:

```
Process:         Dock [21816]
Path:            /System/Library/CoreServices/Dock.app/Contents/MacOS/Dock
Identifier:      Dock
Version:         1.8 (1168)
Code Type:       X86-64 (Native)
Parent Process:  launchd [1039]
User ID:         501

Date/Time:       2012-07-27 11:26:58.819 -0600
OS Version:      Mac OS X 10.8 (12A269)
Report Version:  10

Crashed Thread:  0  Dispatch queue: com.apple.main-thread

Exception Type:  EXC_CRASH (SIGABRT)
Exception Codes: 0x0000000000000000, 0x0000000000000000

Application Specific Information:
*** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[__NSCFBoolean isEqualToString:]: unrecognized selector sent to instance 0x7fff7b2a2ab0'
abort() called
terminate called throwing an exception

Application Specific Backtrace 1:
0   CoreFoundation                      0x00007fff8cd77716 __exceptionPreprocess + 198
1   libobjc.A.dylib                     0x00007fff8e6df470 objc_exception_throw + 43
2   CoreFoundation                      0x00007fff8ce0dd5a -[NSObject(NSObject) doesNotRecognizeSelector:] + 186
3   CoreFoundation                      0x00007fff8cd65c3e ___forwarding___ + 414
4   CoreFoundation                      0x00007fff8cd65a28 _CF_forwarding_prep_0 + 232
5   Dock                                0x000000010d1b6786 Dock + 681862
6   Dock                                0x000000010d1150b2 Dock + 20658
7   Dock                                0x000000010d1ddaed Dock + 842477
8   libdyld.dylib                       0x00007fff8d7827e1 start + 0
```

I'll spare you the detail of the rest of the file. Suffice to say, it
is less informative than one might want for troubleshooting the issue.
I (foolishly) spent about an hour and a half trying to get to the root
of the problem. I found out that the problem was isolated to my own
user, and something between `~/Library/Preferences` and
`~/Library/Application Support`. I'm not sure what the problem was - I
eventually decided to just do this:

```
sudo rm -rf ~/Library/Preferences ~/Library/Application\ Support
```

Then I logged back in, and everything was well. I have a back up from
before the upgrade (Yay, TimeMachine!), so I wasn't concerned with
losing anything important, and I knew that Chef would bring back most
of my settings anyway.

The first thing I did was, of course, `/usr/bin/chef-client`. This
worked great. Until the Dock was restarted as part of my recipes. The
symptom was the same as before - the desktop went light grey, the dock
wasn't running and launchd was respawning it contnually, with the same
errors from above.

I decided to have some quality time with my configuration and get to
the bottom of the problem. I went through all the settings I modify
through `recipe[mac_os_x::settings]` attributes, and did a careful
comparison of manual settings through the OS X system preferences, and
the files changed in `~/Library/Preferences`.

**Side note**: This handy hint comes from
  [Ben Bleything](http://twitter.com/bleything):

```
cd ~/Library/Preferences
git init
git add .
git commit -m 'initial commit'
```

Then, make a change in system preferences, you can use `git status` to
see what plist files are updated. Of course, most of the plist files
are binary so you can't really diff them, but the filename will
indicate which domain to use with the `defaults(1)` command. Handy,
thanks Ben! **End side note.**

Anyway, it took some time, but I tuned all my configuration for the
things I wanted to ensure happened on any new system, and nothing
else. I believe what happened is that some setting I had is no longer
supported on Mountain Lion and its presence makes Dock.app grumpy, but
that is purely speculation. The end result now, though, is that I have
a pretty sane set of configuration that is automatically applied in a
more data driven way, and I'm not using settings that I don't know
100% what they do.

That aside, Mountain Lion is nice so far. The GCC Installer for 10.7
seems to be working just fine, though I haven't tried installing a new
Ruby under it yet. I am looking forward to wider use and adoption of
the Notification Center as a replacement for Growl.

I hope this post is helpful in some way. Unfortunately I don't have an
answer to the Dock crash problem itself, but I have now remedied the
issue for my own use. I'm going to write a new post about how I'm
managing my workstations, to bring the
[information posted](/blog/2011/04/03/managing-my-workstations-with-chef/)
[previously up to date](/blog/2011/09/04/update-to-managing-my-workstations/),
so stay tuned.
