---
layout: post
title: "Changing Class Names"
date: 2012-02-25 09:39
comments: true
categories: [ruby, development]
---

If you change a class name in your library, *do a major version
change*! You don't know who is using your library, even the
undocumented parts.

## Background

Recently, we had a post on the
[Chef mailing list](http://lists.opscode.com) that using bluepill for
the Chef Server daemon process(es) was broken. Upon further
investigation of the output from
[Chef debug output](https://gist.github.com/1847128), it appeared to
be an issue with
[Bluepill itself](https://github.com/arya/bluepill/issues/132), but
after looking into that ticket, the [daemons gem](http://rubygems.org/gems/daemons) had made a change.

Out of curiosity, I was compelled to find out what the source of the
change in the daemons gem was. This lead to a yak shave, as first, I
had to look up the [daemons gem](http://rubygems.org/gems/daemons) on
RubyGems.org to try and find the source. The author of the gem still
uses RubyForge rather than GitHub. That's fine, but it means I have to
do some link-spelunking to find where the [source code lives](http://rubyforge.org/projects/daemons).

Now I take a look at the change log:

    Release 1.1.8: February 7, 2012
    rename to daemonization.rb to daemonize.rb (and Daemonization to Daemonize) to ensure compatibility.

    Release 1.1.7: February 6, 2012
    start_proc: Write out the PID file in the newly created proc to avoid race conditions.
    daemonize.rb: remove to simplify licensing (replaced by daemonization.rb).

    Release 1.1.6: January 18, 2012
    Add the :app_name option for the "call" daemonization mode.

    Release 1.1.5: December 19, 2011
    Catch the case where the pidfile is empty but not deleted and restart
    the app (thanks to Rich Healey)

I then went to the ticket tracker to find out what the source of the
changes might be. Fortunately, there was an open issue that I could
[reference](http://rubyforge.org/tracker/index.php?func=detail&aid=29516&group_id=524&atid=2084
).

My question (which I posted to the ticket) is why wouldn't renaming a
class cause the author to do a new major version? This way other Gems
that rely on this as a dependency could use the paranoia operator,
`~>` so the broken class name wouldn't break usage elsewhere.

I'm glad that the daemons gem author did the right thing and yanked
the broken version. Open source worked well here. The process of
finding this was a bit slower than it should have been, and I think
that the bluepill maintainers moved too quickly to "resolve" the
issue, rather than post their concern about the class naming. Kudos to
thuehlinger for fixing his gem, though.
