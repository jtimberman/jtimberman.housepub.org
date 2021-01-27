---
layout: post
title: "Testing with fission"
date: 2012-02-15 22:53
comments: true
categories: [virtualization, ruby, development]
---

In this post, I'm going to talk about the fission gem:

* [http://rubygems.org/gems/fission](http://rubygems.org/gems/fission)
* [https://github.com/thbishop/fission](https://github.com/thbishop/fission)

I primarily use this to manage the VMware Fusion virtual machines I
use for testing [Opscode's Chef Cookbooks](http://community.opscode.com/users/opscode).

This post is rather light on specific details about things that are
either "common" knowledge or documented elsewhere. Particularly, I
won't tell you how to set up the virtual machines, other than a few
notes that I think make it easier to manage virtual machines in this
way. In other words, you're smart and can figure them out.

## Install and Configure

The first step to use the fission RubyGem is to install it. If you
don't like RubyGems, then create a package or grab the source from the
GitHub link, above.

    gem install fission

Test that it is detecting your VMware Fusion VMs:

    fission status

Fission has a configuration file, `~/.fissionrc`, which is yaml
format. If the status command fails, you may need to configure fission
to find the `vmrun` command. Here's the example from my system:

    ---
    vmrun_bin: /Applications/VMware Fusion.app/Contents/Library/vmrun

## Install an OS on a VM

If you're reading this, I presume you know how to install an OS in a
VMware virtual machine. I do a number of tasks during the installation
to make it easy and consistent to work with all my test VMs.

* Use bridged networking with DHCP

This usually results in the least amount of hassle for connecting to
the VM without any tunneling or port forwarding tomfoolery.

* Give it a simple VM name with alphanumeric characters only.
* Use the same hostname during installation as the VM name

In the examples below, I use my "guineapig" system. I also have other
systems like "ubuntu1110", "freebsd82" and "centos6". This is the name
you'll use to refer to the VM with fission, so it should be short,
easy to type and clearly identifiable.

* Set the root password, even if the OS doesn't use the root account
* Make sure SSH as root is enabled

Some Linux distributions such as Ubuntu do not enable the root login.
This is for testing, so I really don't care, and I can always write a
Chef recipe to lock things down (as I would in production) if
required.

I also set a simple password that I can use with `-P` to `knife
bootstrap` without the shell doing anything with special characters.

* Use NTP

Install the NTP package for your operating system. The workflow here
(and the whole point really) is to make heavy use of VMware Fusion
snapshots and rollback, so it is important that the system time is
correct. I customized the bootstrap templates I use to add `ntpdate`.

## Using Fission

And now the moment you've been waiting for. First, see the fission
README for full detail on the commands available. I'm going to focus
here on how I use it.

After the install and post install tasks are done, I create a
new snapshot for the VM.

    % fission status
    guineapig        [running]
    % fission snapshot create guineapig base

The name is "base" because thats a good name for a baseline. It can be
useful to create specifically named snapshots for particular purposes.

I use Opscode Hosted Chef as my server and I already have my local
workstation set up with the validation key, a Chef repository and have
uploaded the cookbook(s) I use for testing. I'll use "knife
bootstrap" to kick off a run on my VM:

    % knife bootstrap 10.1.1.129 -x root -Pvanilla -r 'recipe[apache2]'
    ...
    INFO: service[apache2] restarted
    INFO: Chef Run complete in 44.324473 seconds

Sweet, it worked. However, if the Chef run fails, I can log in as
root, fix the bug and rerun, or whatever else may need to be done.
Then once I'm ready to reset the VM, fission comes back to play.

    % fission snapshot revert guineapig base
    Reverting to snapshot 'base'
    Reverted to snapshot 'base'

Note that fission will poweroff the VM when reverting the snapshot.
Turn it on again with the start command.

    % fission status
    guineapig        [not running]

    % fission start guineapig
    Starting 'guineapig'
    VM 'guineapig' started

    % fission status
    guineapig        [running]

And after logging in, we can see that apache2 is not installed as it
should not be after the snapshot is restored.

    % ssh root@guineapig
    root@guineapig's password:
    root@guineapig:~# dpkg -l apache2
    No packages found matching apache2.

The VM is now ready to do my bidding once again.

# Cleanup

Note that reverting the snapshot doesn't delete the Chef node or
client objects. Since fission is a Ruby library, a simple knife plugin
can wrap up all the fission revert, restart and Chef cleanup, though.
I called mine nukular.

    % knife nukular guineapig base guineapig.int.example.com

And here's the plugin I'm using:

```ruby
require 'chef/knife'

  module KnifePlugins
    class Nukular < Chef::Knife
      deps do
        require 'fission'
        require 'chef/node'
        require 'chef/api_client'
      end

      banner "knife nukular VM SNAPSHOT [NODE]"

      def run
        vm, snapshot = @name_args
        node = @name_args[2].nil? ? vm : @name_args[2]
        Fission::Command::SnapshotRevert.new(args=[vm, snapshot]).execute
        Fission::Command::Start.new(args=[vm]).execute
        Chef::Node.load(node).destroy
        Chef::ApiClient.load(node).destroy
      end
    end
  end
```

The command-line usage takes 2 or 3 arguments. The first two must be
the VM name and the snapshot name, e.g. `guineapig` and `base`. If the
node name is different than the VM name, then specify it.

    % knife nukular guineapig base guineapig.int.example.com

Note that the plugin has *zero* error handling or any other sensible
things. You may want to modify it before you use it. Or not, these are
just test systems after all.

## Full example

Minus the output from the commands, here is the full output of testing
the Opscode apache2 cookbook on my guinea pig. Assume that all the
required things from my Chef Repository have been uploaded to the Chef
Server and the knife configuration is correct. Also, the `chef-full`
bootstrap template specified here is a customized version of the
template in the
[Chef source master branch template](https://github.com/opscode/chef/blob/master/chef/lib/chef/knife/bootstrap/chef-full.erb)
that has `ntpdate -u pool.ntp.org` in it.

```
% fission status
% fission snapshot create guineapig testing-apache2
% knife bootstrap 10.1.1.129 -x root -P vanilla \
  -r 'recipe[apache2]' -d chef-full
% knife nukular guineapig testing-apache2 guineapig.int.housepub.org
```

That's it. I hope you find this helpful!
