---
layout: post
title: "Process Supervision: Solved Problem"
date: 2012-12-29 21:41
comments: true
categories: [unix, daemontools, process supervision]
---

**TL;DR**: Use [runit](http://smarden.org/runit/index.html); skip to
"This is a Solved Problem" and "Additional Resources" sections at the
end of this post.

Recently on my twitter stream, I saw a link to
[a question](http://stackoverflow.com/questions/3295737/how-to-start-jetty-properly)
on Stack Overflow about how to properly start Jetty. While the
question is over 2.5 years old, the question stems from the common
problem: How do I start a long running process, and *keep it running*?
That an accepted answer is to run it under "nohup" (or screen?!) tells
me that for some reason, this isn't perceived as a solved problem.

In my opinion, process supervision *is* a solved problem. However,
this wheel keeps getting reinvented, or reimplemented with solutions
that are not easily manageable or scalable. In this post, I will
clarify some terminology, describe commonly understood goals of
process supervision, explore some of the common approaches, and how
they don't meet those goals, and finally explain why I feel this is a
solved problem.

*Note* This is a Unix/Linux centric post. Windows has its own
  methods for running services, but the problem is definitely solved
  there too; Microsoft gave developers and administrators APIs that
  seem to be commonly used.

# Process Supervision is Not Service Management

What exactly is process supervision?

One of the reasons for running servers is to provide a service of some
kind. That is, an application that provides business value. A service
is made up of one or more running processes on computer systems
somewhere. Those processes are typically long-lived running daemons.

Process supervision is simply the concept of starting a daemon and
keeping it running.

Note that this is *not* the same as more general service management.
That may imply multiple services, which may be running on separate
physical or virtual servers in a distributed system. That is outside
the scope of this post.

This is also *not* service monitoring, a la graphing (munin, graphite)
and/or alerting (nagios, sensu). That is also outside the scope of
this post.

# Goals and Benefits of Process Supervision

The
[Wikipedia page on Process Supervision](http://en.wikipedia.org/wiki/Process_supervision)
describe its benefits as follows:

- Ability to restart services which have failed
- The fact that it does not require the use of "pidfiles"
- Clean process state
- Reliable logging, because the master process can capture the stdout/stderr of the service process and route it to a log
- Faster (concurrent) and ability to start up and stop

To this, I add:

- Manage processes with Unix signals
- Simple setup that is configuration management-friendly (I'm obviously
  [biased](http://opscode.com/chef))
- Dependency management between services on the same machine

For sake of argument, these combined lists of goals and benefits are
my criteria for a process supversion system in this post.

**Spoiler alert**: [runit](http://smarden.org/runit/benefits.html)
covers all of these, as does
[s6](http://skarnet.org/software/s6/why.html).

# Common Approaches

I'm going to talk about some approaches to process supervision, and
how they don't meet the criteria above. This won't be comprehensive. I
want to illustrate the highlights.

## Nohup

First, the approach mentioned in the StackOverflow answer: "nohup."
The "nohup" command will "run a command immune to hangups, with output
to a non-tty." This typically involves logging into a system and
manually invoking the command, such as:

```
nohup jar -jar start.jar
```

This doesn't provide the ability to restart if it fails. The process
state is contaminated with whatever the user has in their login shell
profile(s). It will log to "nohup.out" by default, though it can be
redirected to another file. It's pretty safe to say that in my opinion
that this fails the criteria above, and should not be used for long
running processes, especially those as important as running your Java
application.

## Terminal Multiplexers

Next up, a common approach for running process is to start up screen
(or tmux), and let them run in the foreground. Screen and tmux are
terminal multiplexers. That is, they are "full-screen window
manager[s] that multiplex a physical terminal between several
processes." These are great tools, and
[I use tmux](https://jtimberman.housepub.org/blog/2012/01/28/iterm2-with-tmux/)
for other reasons. However, this fails the criteria for the same
reasons as nohup. Additionally, automating a process running in screen
is not a simple task that can be repeated reliably.

## SysV/BSD Init

Most commonly, process management (and not supervision) is handled on
Unix/Linux systems by plain ol' SysV/BSD
"[init](http://en.wikipedia.org/wiki/Init)." These obviously fail to
meet the criteria above, because two new(er) systems,
"[upstart](http://upstart.ubuntu.com/)" and
"[systemd](http://www.freedesktop.org/wiki/Software/systemd)" have
been written to address the problems. That said, "init" fails pretty
much all the criteria:

1. No ability to restart services which have failed.
2. One of the biggest problems is handling of "pidfiles."
3. Process state is theoretically clean, but then realize the average
init script sources at least two different shell scripts for helper
functions and environment variables, nevermind homegrown scripts that
might read in user shell settings, too.
4. The best one can hope for in logging is that the process writes to
syslog, because many init scripts redirect log output in different,
non-portable ways.
5. Init is 100% sequential startup, no concurrency: "/etc/rc2.d/S*"
6. Sure, you can send signals to the process, but most init scripts
don't support more than "reload" or "restart," so you're left on your
own with picking up the pieces manually.
7. Configuration management is easy, right? Just "ensure running" or
"action :start" - except let's not forget the "/etc/sysconfig" or
"/etc/default" that sets more configuration. And that the package
manager might have started it for you before you're ready.
8. Okay, I'll give you this. As long as the sequential ordering of the
init scripts is all correct to meet the dependencies.

Also, I have a personal pet peeve about init script complexity,
inconsistency and non-portability between distributions of Linux, let
alone Unix. I could (and probably will) write a post about that. For a
taste, see [CHEF-3774](http://tickets.opscode.com/browse/CHEF-3774).

**Note**: I'm generalizing both SysV and BSD here. I admit I don't have
extensive experience with BSD systems, but my observation is it fails
in very similar ways to SysV.

## Systemd/Upstart

The newer init-replacement systems, systemd and upstart are worth
their own section, though I'll be brief. Other people have
[posted](http://blog.mywarwithentropy.com/2010/10/upstart-better-init-or-more-painful-one.html)
[about](http://monolight.cc/2011/05/the-systemd-fallacy/) these, and
they're pretty well covered on the
[s6 comparison](http://skarnet.org/software/s6/why.html).

Mainly, I see both of these as reinventing the solution that follows.
However, a couple points I'd like to make:

1. Both systems are primarily focused on desktop systems, rather than
server systems. This is mostly evident in their use of D-Bus
(*Desktop* bus), goals of faster boot time, and that their roots are
in primarily desktop-oriented Linux distributions (Fedora and Ubuntu).
2. They both completely replace init, which isn't necessarily bad.
However, they both operate differently from init, and each other, thus
being a non-portable major difference between Linux distributions.

## Other Process Supervision Systems

There are a lot of process supervision systems out there. In no
particular order, an incomplete list:

- [Bluepill](https://github.com/arya/bluepill)
- [God](http://godrb.com/)
- [Foreman](http://ddollar.github.com/foreman/) (not to be confused
  with [The Foreman](http://theforeman.org/))
- [Supervisor](http://supervisord.org/)
- [Monit](http://mmonit.com/monit/)

I have varying degrees of experience with all of these. I have written
[significant](http://community.opscode.com/cookbooks/bluepill)
[amounts](http://community.opscode.com/cookbooks/god) of automation
code for operating some of them.

I think that with perhaps the exception of Monit(*), they are redundant
and unnecessary.

(*): I don't have as much experience with Monit as the others, and it
seems to have a lot of nice additional features. I've also heard
[it goes well](https://twitter.com/nrr/status/293564015827894272) with
my favorite solution.

# This Is a Solved Problem

Earlier I mentioned [runit](http://smarden.org/runit/) meets all the
criteria I listed above. In my opinion, it is the solution to the
process supervision problem. While the
[runit website](http://smarden.org/runit/benefits.html) itself lists
its benefits, it gets a nod from the
[s6 project](http://skarnet.org/software/s6/why.html), too. The
underlying solution is actually the foundation both runit and s6 build
on: [Dan J Bernstein's daemontools](http://cr.yp.to/daemontools.html).
The
[merits of DJB and daemontools](http://www.skarnet.org/software/skalibs/djblegacy.html)
are *very* well stated by the author of s6. I strongly recommend
reading it, as he sums up my thoughts about DJB, too. It is worth
noting that I do like s6 itself, but it isn't currently packaged
anywhere and adheres fairly strictly to the "slash package"
convention, which isn't compatible with the more popular
[Filesystem Hierarchy Standard](http://www.pathname.com/fhs/).

Anyway, the real point of this post is to talk about why I like runit.
I think the best way to explain it is to talk about how it meets the
criteria above.

### Restart Failed Services

The `runsv` program
[supervises services](http://smarden.org/runit/benefits.html#supervision),
and will restart them if they fail. While it doesn't provide any
notification that the service failed, other than possibly writing to
the log, this means that if a configuration issue caused a service to
fail, it will automatically start when the configuration file is
corrected.

### No PID files

Each service managed by `runsv` has a "service directory" where all
its files are kept. Here, a "supervise" directory is managed by runsv,
and a "pid" file containing the running PID is stored. However this
isn't the same as the pidfile management used in init scripts, and it
means program authors don't have to worry about managing a pidfile.

### Clean Process State

Runit's benefits page
[describes how](http://smarden.org/runit/benefits.html#state) it
guarantees clean process state. I won't repeat it here.

### Reliable Logging

Likewise, Runit's benefits page describes how it provides
[reliable logging](http://smarden.org/runit/benefits.html#log).

### Parallel Start/Stop

One of the goals and benefits lauded by systemd and upstart is that
they reduce system boot time because various services can be started
in parallel. Runit also starts up all the services it manages in
parallel. More about this under dependency management, too.

### Manage Processes (with Unix Signals)

The `sv` [program](http://smarden.org/runit/sv.8.html) is used to send
signals to services, and for general management of the services. It is
used to start, stop and restart services. It also implements a number of
commands that can be used for signals like TERM, CONT, USR1. `sv` also
includes "LSB-init" compatibility, so the binary can be linked to
`/etc/init.d/service-name` so "init style" commands can be used:

```
sudo /etc/init.d/service-name status
sudo /etc/init.d/service-name restart
```

And so forth.

### Simple Setup, Configuration Management Friendly

One of the benefits listed is that runit is
[packaging friendly](http://smarden.org/runit/benefits.html#packaging).
This is interesting because that also makes it configuration
management friendly. Setting up a new service under runit is
fairly simple:

1. Create a "service directory" for the service.
2. Write a "run" script that will start the service.
3. Create a symbolic link from the service directory to the directory
of supervised services.

As an example, suppose we want to run a git daemon. By convention,
we'll create the service directory in `/etc/sv`, and the supervised
services are linked in `/etc/service`.

```
sudo mkdir /etc/sv/git-daemon
sudo vi /etc/sv/git-daemon/run
sudo chmod 0755 /etc/sv/git-daemon/run
sudo ln -s /etc/sv/git-daemon /etc/service
```

The run script may look like this
([chpst](http://smarden.org/runit/chpst.8.html) is a program that
comes with runit that changes the process state, such as the user it
runs as):

```
#!/bin/sh
exec 2>&1
exec chpst -ugitdaemon \
  "$(git --exec-path)"/git-daemon --verbose --reuseaddr \
    --base-path=/var/cache /var/cache/git
```

Within a few seconds, the git daemon will be running:

```
root      6236  0.0  0.0    164     4 ?        Ss   19:03   0:00 runsv git-daemon
119      12093  0.0  0.0  11460   812 ?        S    23:46   0:00 /usr/lib/git-core/git-daemon --verbose --reuseaddr --base-path=/var/cache /var/cache/git
```

The [documentation](http://smarden.org/runit/faq.html) contains a
lot more information and usesp

**Note**: As evidence that this is packaging friendly, this is
  provided by the very simple `git-daemon-run` package on
  [Debian](http://packages.debian.org/search?keywords=git-daemon-run)
  and
  [Ubuntu](http://packages.ubuntu.com/search?keywords=git-daemon-run).

### Dependency Management

Many services require that other services are available before they
can start. A common example is that the database filesystem must be
mounted before the database can be started.

Depending on the services, this can be addressed simply by `runsv`
restarting services that fail. For example, if the startup of the
database fails because its file system isn't mounted and the process
exits with a return code greater than 0, then perhaps restarting will
eventually work once the filesystem is mounted. Of course, this is an
oversimplified naive example.

The runit [FAQ](http://smarden.org/runit/faq.html#depends) addresses
this issue by use of the program `sv`, mentioned earlier. Simply put,
use the `sv start` command on the required service.

## A Few Notes

I've used runit for a few years now. We used it at HJK Solutions to
manage all system and application services that weren't packaged with
an init script. We use it at Opscode to manage all the services that
run [Opscode Private Chef](http://opscode.com/private-chef).

1. Manage services that run in the foreground. If a service doesn't
support running in the foreground, you'll have a bad time with it in
runit, as `runsv` cannot supervise it.
2. Use [svlogd](http://smarden.org/runit/svlogd.8.html) to capture log
output. It automatically rotates the log files, and can capture both
STDOUT and STDERR. It can also be configured (see the man page).
3. The author of runit is also the package maintainer for
Debian/Ubuntu. This means runit works extremely well on these
distributions.
4. I don't replace init with runit, so I can't speak to that.
5. [Ian Meyer](https://twitter.com/ianmeyer) maintains an
[RPM spec](https://github.com/imeyer/runit-rpm) for runit packages
that work well. It will be included in Opscode's
[runit cookbook](http://community.opscode.com/cookbooks/runit)
[soon](http://tickets.opscode.com/browse/COOK-2164).
6. If you use Chef, use Opscode's
[runit cookbook](http://community.opscode.com/cookbooks/runit). It
will [soon](http://tickets.opscode.com/browse/CHEF-154) have a
resource/provider for managing runit services, instead of the definition.

# Conclusion

Use runit.

But not just because I said so. Use it because it meets the criteria
for a process supervision system, and it builds on the foundation
pioneered by an excellent
[software engineer](www.skarnet.org/software/skalibs/djblegacy.html).

After all, I'm not [the only one who thinks so](http://rubyists.github.com/2011/05/02/runit-for-ruby-and-everything-else.html).

# Additional Resources

* [s6](http://skarnet.org/software/s6/) - by Laurent Bercot
* [runit](http://smarden.org/runit/) - by Gerrit Pape
* [daemontools](http://cr.yp.to/daemontools.html) - by Dan J Bernstein
* [Runit for Ruby (and everything else)](http://rubyists.github.com/2011/05/02/runit-for-ruby-and-everything-else.html)
* [runit cookbook for Chef](http://community.opscode.com/cookbooks/runit)
  (nice changes coming in
  [CHEF-154](http://tickets.opscode.com/browse/CHEF-154), and
  [COOK-2164](http://tickets.opscode.com/browse/COOK-2164) too!)
