---
layout: post
title: "OS X Workstation Management with Chef"
date: 2012-07-29 12:36
comments: true
categories: [chef, workstation, osx]
---
**Update: This post is old and outdated. I'll have another post in 2015 about workstation management with Chef.**

I have written
[twice](/blog/2011/04/03/managing-my-workstations-with-chef/)
[before](/blog/2011/09/04/update-to-managing-my-workstations/) about
managing my OS X workstations with Chef. The first post has one of my
highest hit counts of any blog post, so it is certainly a topic of
interest to people.

This post is a rewrite of the original, and now has an
[accompanying Chef Repository](https://github.com/jtimberman/workstation-chef-repo)
where all the code I talk about is available.

## Background

The current incarnation of this repository lives in the more general
private Chef Repository I use for my home network, since I manage more
than just workstations with Chef. I have used the main recipe
(workstation::default) with great success on several Mac OS X systems:
Macbook Pro, iMac, and Macbook Air running various versions between
10.6 and 10.8. I recently used it to configure a replacement Macbook
Pro for work, and then again after upgrading to
[Mountain Lion](/blog/2012/07/27/mountain-lion-upgrade/).

## The Setup

Also known as "bootstrapping Chef", the system needs to be set up to
run Chef.

* Opscode Hosted Chef is my Chef Server
* I run everything as a non-privileged user

### Installation

I use the Opscode
[full-stack installer](http://opscode.com/chef/install) on all my
systems, including OS X, because it includes everything Chef needs,
including Ruby.

Whether you're unpacking a brand new Mac, or using an existing system,
use this command:

``` sh
curl -L https://opscode.com/chef/install.sh | sudo bash
```

Symbolic links are created for Chef's binaries in `/usr/bin`.

**Mountain Lion Note**: I did this on Lion before upgrading to
  Mountain Lion. Apple removed X11 from Mountain Lion, and the
  installer opens an xterm, so I don't know how/if this works the
  same on a brand new Mountain Lion system.

### Configuration

Next, create a configuration file for the Chef Server, and copy the
validation key into place. If this is a new Mac, you'll need to get
your validation key copied to the system.

``` sh
sudo mkdir -p /etc/chef
sudo vi /etc/chef/client.rb
cp ~/Downloads/ORGNAME-validation.pem /etc/chef/validation.pem
```

I am using Opscode Hosted Chef, and this is my `/etc/chef/client.rb`.
The path options are so Chef writes its files in a location my user
has write access.

``` ruby
base_dir = "/Users/USERNAME/.chef"
chef_server_url         'https://api.opscode.com/organizations/ORGNAME'
validation_client_name  'ORGNAME-validator'
checksum_path           "#{base_dir}/checksum"
file_cache_path         "#{base_dir}/cache"
file_backup_path        "#{base_dir}/backup"
cache_options({:path => "#{base_dir}/cache/checksums", :skip_expires => true})
```

## The Repository

I have made
[the repository available on GitHub](https://github.com/jtimberman/workstation-chef-repo).

``` sh
git clone git://github.com/jtimberman/workstation-chef-repo.git
```

Normally, systems that are configured with Chef wouldn't have the Chef
Repository on them. For the purpose of this post, clone the
repository to the local system. Presumably, one might do further
development to it.

Before we upload it and run Chef, let's explore what is included.

## Data Bags

The repository contains two data bags with a single item each. One is
for the local user, the other is for the workstation setup.

The USERNAME should be changed to the local user that is being
configured on the workstation. To ensure that the correct value is
used, run the following and use the value returned.

```
% ruby -retc -e 'puts Etc.getlogin'
jtimberman
```

Thus, I use `jtimberman` for my systems.

The user data bag item is used in two cookbooks, users and
workstation. This is described below under Cookbooks.

The workstation data bag item contains various data about the
workstation itself, software that should be installed, property list
files dropped off, etc. The JSON file in the repository contains several
examples. Modify this as required for your own system.

## Roles

There are three roles.

### base

This is the role I apply on *all* my systems, not just workstations.
Aside from the contents of the role file in the repository, I also set
attributes across my systems for a variety of other purposes like
postfix, munin, ntp and so forth. For the workstation setup purposes,
it contains the attributes I use for installing Ruby under Rbenv, and
the gems I want available on all my systems that aren't project
specific (I use [bundler](http://gembundler.com) for those).

``` ruby
name "base"
description "Base role for all nodes"
override_attributes(
  "ruby_build" => {
    "git_ref" => "v20120524",
    "upgrade" => true,
    "install_pkgs" => []
  },
  "rbenv" => {
    "install_pkgs" => [],
    "user_installs" => [
      {
        "user" => "USERNAME",
        "rubies" => ["1.9.3-p194"],
        "global" => "1.9.3-p194",
        "gems" => {
          "1.9.3-p194" => [
            {"name" => "bundler", "version" => "1.1.1"},
            {"name" => "git-up"},
          ]
        }
      }
    ]
  }
)
```

Edit the list of gems as required for your preferences. The ones
included in the role are what I find useful or required for my day to
day work on Chef and Chef-related projects (like
[opscode-cookbooks](https://github.com/opscode-cookbooks)).

The base role does not have a run list. It is included instead by OS
specific roles that I apply to Ubuntu or OS X systems respectively. As
this is a post for my OS X workstations, let's look at that role next.

### mac_os_x

The `mac_os_x` role is applied on all my OS X systems. Of note, it
includes the base role, and the homebrew recipe. The homebrew cookbook
includes a package resource provider that replaces the Chef default
for OS X, macports, with homebrew.

``` ruby
name "mac_os_x"
description "Role applied to all Mac OS X systems."
run_list(
  "role[base]",
  "recipe[build-essential]",
  "recipe[homebrew]"
)
```

This role probably doesn't need to be edited.

### workstation

This is the role of interest, which contains the workstation specific
run list and attributes.

The role itself is very long, so I won't include it here. You can view it
[in the repository](https://github.com/jtimberman/workstation-chef-repo/tree/master/roles/workstation.rb).

Do note that `recipe[mac_os_x::firewall]` requires root access, and
will prompt for the sudo password (and pause the whole run until entered).

Edit the `mac_os_x` settings as required for your own preferences.
Edit other attributes as required for software you wish to use or
install.

## Cookbooks

The repository uses a number of cookbooks, most of which are published
on the Chef Community site as well. I'm not going to describe all the
cookbooks in this post, just the ones that are most relevant for
workstation setup.

### Development Essentials

These are:

* build-essential
* homebrew
* git
* ruby_build
* rbenv

On OS X, `build-essential` will install
[Kenneth Reitz's OS X GCC installer](https://github.com/kennethreitz/osx-gcc-installer)
- XCode is *not required*. Of course, you may have other reasons why
  you want to have XCode, and that is outside the scope of this
  repository. If so, remove the build-essential recipe from the roles.

The homebrew cookbook uses
[Homebrew](http://mxcl.github.com/homebrew/) as the default package
provider on OS X. The default recipe will install Homebrew, Git with homebrew, and
ensure that the formulae are updated.

**IMPORTANT NOTE** Homebrew recently "broke"
  ([in my opinion](https://github.com/mxcl/homebrew/issues/13299)) the
  output of `brew info`. I manually patched [my local copy of Library/Homebrew/cmd/info.rb](https://github.com/mxcl/homebrew/pull/13300/files) after discovering
  this halfway through setup of my new system.

The git cookbook installs git using
[the Git OS X installer](https://code.google.com/p/git-osx-installer/),
and the git binary will be `/usr/bin/git`. This is redundant with git
installed from homebrew, but at some point I had issues and I don't
remember if they were resolved. If you wish to use git from homebrew,
use `/usr/local/bin/git` instead.

The ruby_build and rbenv cookbooks are by Fletcher Nichol, and are
quite excellent for installing per-user Rubies of a specific version,
and gems using the `rbenv_gem` LWRP. The `base` role has the
attributes set up for how I like this, YMMV.

### users

I use an older, modified version of Opscode's `users` cookbook, pre
`users_manage` LWRP. The recipe adds capability for distributing
arbitrary files, such as dotfiles for users. To use this, create a
"files" section of the users data bag item. The USERNAME.json item
includes examples of this. Each file needs to be copied to
`cookbooks/users/files/default/USERNAME/` as the source file name used
in the data bag item.

### workstation

The workstation cookbook has a recipe that does all the work of
reading the workstation data bag item and setting up the system per
the data available.

The README.md in the cookbook contains detailed information about its
use, and the data bag item already has the structure to get started.

If the `plists` array is used, then each plist file should be copied
into the `files/default/` directory.

### mac_os_x

My [mac_os_x](http://community.opscode.com/cookbooks/mac_os_x)
cookbook has two LWRPs that I use elsewhere in this repository:

* `mac_os_x_plist` - drops off property list (plist) files in `~/Library/Preferences`
* `mac_os_x_userdefaults` - writes OS X user settings with the `defaults(1)` system

The plist files used by `mac_os_x_plist` should be added in the
`files/default` directory of the cookbook where the resource is used
in a recipe.

The `mac_os_x::settings` recipe will read the
`node['mac_os_x']['settings']` attribute for user defaults to apply.

See the `mac_os_x` cookbook's README for more information.

### Applications

The following are application specific cookbooks that I use:

* iterm2
* virtualbox
* ghmac
* 1password
* xquartz

The iTerm2 cookbook will set up iTerm 2 and optionally add tmux
integration. I
[wrote about this awhile back](/blog/2012/01/28/iterm2-with-tmux/).

We use [Vagrant](http://vagrantup.com) extensively at Opscode, which
requires VirtualBox. This recipe will install it per the attributes
set in the workstation role.

Install [GitHub for Mac](http://mac.github.com) with the ghmac
cookbook. Local setup for it is on your own.

I have 1password in here because I used to install it from the zip
file, but I may remove this at some point since I install it from the
Mac App Store now.

Note that the versions of these apps may be old, but they have
Sparkle.framework or can otherwise update themselves to newer versions
easily. Click the buttons, it's cool.

I don't have a recipe for managing application installation through
the Mac App Store. It's really not that hard to fire up the app and
click the "Install" button next to the apps you want though.
Seriously, it would take longer to figure out a command-line or API
way to do this, if it's even possible. Just click the button.

### Others

The other cookbooks in the repository are there as dependencies and
may or may not be used specifically.

## Upload Repository, Run Chef

Once it is cloned, all the components need to be uploaded to the Chef
Server with Knife. As that is installed with Chef, it will be
available. The knife config file and user key do need to be copied to
`.chef` in the chef-repo. If necessary, download them from Opscode
Hosted Chef (or your Chef Server).

``` sh
cd workstation-chef-repo
mkdir .chef
cp ~/Downloads/knife.rb .chef
cp ~/Downloads/USERNAME.pem .chef
```

Make your changes to the data bags and roles. I'll wait here.

Then, upload everything.

``` sh
knife data bag create users
knife data bag create apps
knife data bag from file users USERNAME.json
knife data bag from file apps workstation.json
knife role from file base.rb mac_os_x.rb workstation.rb
knife cookbook upload -a
```

Finally, run Chef!
```
% whoami
jtimberman
% chef-client
INFO: *** Chef 10.12.0 ***
... loads of output, hooray ...
INFO: Chef Run complete in 45.116912 seconds
```

## FAQ

These aren't necessarily questions anyone asked, but a more preemptive
FAQ :).

### This seems heavyweight, why all this effort?

As a sysadmin, I want to do something once and automate it afterward.
That includes all the stuff I need to do to have a useful, usable work
environment. This means that when I get a new computer, or have to
wipe and reinstall (rare, but happens), I can get back to a productive
environment very quickly.

I have three OS X systems I use regularly (work laptop, personal
laptop, family iMac). Having them in a Chef Server gives me access to
information about these systems easily with knife.

Also, this post is focused specifically on OS X, however this setup
works pretty much as is on Linux. I simply don't use Linux as a
desktop OS, but I do have "workstation-like" systems that I SSH into,
and this is generally fine for those.

### Why Chef Server? Why not Chef Solo?

Honestly, I don't actually use Chef Solo except as a way to setup a
[Chef Server](http://wiki.opscode.com/display/chef/Installing+Chef+Server+using+Chef+Solo).
Since I use Chef Client/Chef Server so often, it is second nature for
me to do it. You're free to adapt this to work with Solo.

### Will you support Windows with this repository?

No. I don't use Windows as a workstation/desktop anymore.

It might just work on Windows though. It did once, but I haven't tried
in a few months.

### I want to make this moar awesome, will you merge my pull request?

Thank you. I appreciate that you want to help me, or other members of
the community. However I consider this pretty much "feature complete",
as it meets all my needs, and I don't plan to merge any pull requests.

For individual cookbooks, they have their own repositories linked from
their pages on the Chef Community site.

### Why do you have redundancy or inconsistent use?

Such as plist file location, dmg installation, etc.

Because: Reasons. This codebase has been developed over ~2 years. It
works for me.

### How can I get help?

You can [email me](mailto:opensource@housepub.org). However, as I said
before this is a free time project, so I might not respond right away.
If you're an Opscode Hosted or Private Chef customer, please contact
[Opscode support](http://www.opscode.com/support/). Finally, community
based support is available through our
[community resources](http://community.opscode.com).

# Further resources

If this is a topic of interest to you, I'd also like to point out a
few similar projects that may be interesting. They have inspired me
and things I have implemented in my own setup, so thank you Ben, Corey
and Matthew and Brian at Pivotal!

* [Ben Bleything's OS X Bootstrap](https://github.com/bleything/bootstrap)
* [Corey Donohoe's Cinderella](http://www.atmos.org/cinderella/)
* [Pivotal Labs workstation setup](https://github.com/pivotal/pivotal_workstation/)
