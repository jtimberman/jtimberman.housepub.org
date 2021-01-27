---
layout: post
title: "Using Multiple Provisioners in Vagrant"
date: 2012-08-10 11:39
comments: true
categories: [virtualization, chef]
---

**Update** Chef 10.14 is
  [released](http://www.opscode.com/blog/2012/09/07/chef-10-14-0-released/).
  I removed the "`--pre`" from the gem install commands but otherwise
  left this post, since it was written by Past Me.

As you may be aware, the next release of Chef, 10.14, is in
[beta testing](http://lists.opscode.com/sympa/arc/chef-dev/2012-07/msg00021.html).
Most publicly available baseboxes for [Vagrant](http://vagrantup.com)
aren't built for the beta. But perhaps you want to try it out?

Vagrant to the rescue! Besides the Chef solo and client provisioners,
Vagrant has a shell provisioner, too. It allows you to pass in an
inline command, or a shell script. This example builds on my
[MultiVM Vagrantfile for Chef post](/blog/2012/03/18/multivm-vagrantfile-for-chef/).
You may wish to refer to the full file in that post for complete
context.

First, the shell provisioner line looks like this:

```ruby
config.vm.provision :shell, :inline => "sudo /opt/chef/embedded/bin/gem install chef --no-ri --no-rdoc"
```

It goes in the config block. Remember that `cookbook_testers` is a
hash of data about the multiple VMs I'm using, so they would all get
launched with this provisioner.

```ruby
Vagrant::Config.run do |global_config|
  cookbook_testers.each_pair do |name, options|
    global_config.vm.define name do |config|
      ### Use shell for great justice!
      config.vm.provision :shell, :inline => "sudo /opt/chef/embedded/bin/gem install chef --no-ri --no-rdoc"
      ### chef-client (allthethings)
      config.vm.provision :chef_client do |chef|
        # ... chef provisioner is the same as before
      end
    end
  end
end
```

And now, a `vagrant up` on my Ubuntu 12.04 VM, with the less
interesting output truncated:

```
% vagrant up precise
...
[precise] Running provisioner: Vagrant::Provisioners::Shell...
Successfully installed chef-10.14.0.beta.3
1 gem installed
[precise] Running provisioner: Vagrant::Provisioners::ChefClient...
...
[precise] Running chef-client...
...
[Fri, 10 Aug 2012 17:50:23 +0000] INFO: *** Chef 10.14.0.beta.3 ***
...
[Fri, 10 Aug 2012 17:50:37 +0000] INFO: Chef Run complete in 11.323208919 seconds
```

One of the most compelling features of Chef 10.14 is the introduction
of
["no-op" or "dry-run" mode](http://tickets.opscode.com/browse/CHEF-13),
also called
"[Why Run Mode](http://lists.opscode.com/sympa/arc/chef/2012-07/msg00025.html)".
However, at this time, neither arbitrary command-line arguments nor why-run
mode itself are supported in Vagrant. I opened a
[pull request](https://github.com/mitchellh/vagrant/pull/1067) for the
former, which will allow the latter easily. Of course, vagrant is
highly valuable for doing testing by *actually running* your recipes,
but I think arbitrary command-line options will be a welcome feature
for other purposes.

Another use case for this particular method of using Vagrant
provisioners is to update RubyGems inside an "Omnibus" full stack
install, to resolve
[this bug](http://tickets.opscode.com/browse/CHEF-3295), or
[this request](http://tickets.opscode.com/browse/CHEF-2871).

```ruby
config.vm.provision :shell, :inline => "sudo /opt/chef/embedded/bin/gem update --system"
```

These two example commands can be combined in a single `:inline`, or
as a shell script.

```ruby
config.vm.provision :shell, :path => "gem-updater.sh"
```

And the shell script, `gem-updater.sh` in the same directory as the Vagrantfile:

```sh
#!/bin/sh
sudo /opt/chef/embedded/bin/gem update --system
sudo /opt/chef/embedded/bin/gem install chef --no-ri --no-rdoc
```
