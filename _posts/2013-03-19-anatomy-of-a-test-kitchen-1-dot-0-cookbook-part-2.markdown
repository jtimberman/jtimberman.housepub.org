---
layout: post
title: "Anatomy of a Test Kitchen 1.0 Cookbook (Part 2)"
date: 2013-03-19 09:28
comments: true
categories: [chef, development]
---

**DISCLAIMER** Test Kitchen 1.0 is still in *alpha* at the time of
  this post.

**Update** We're no longer required to use bundler, and in fact
  recommend installing the required RubyGems in your globalRuby
  environment (#3 below).

**Update** The log output from the various kitchen commands is not
  updated with the latest and greatest. Play along at home, it'll be
  okay :-).

This is a continuation from [part 1](/blog/2013/03/19/anatomy-of-a-test-kitchen-1-dot-0-cookbook-part-1/)

In order to run the tests then, we need a few things on our machine:

1. VirtualBox and Vagrant (1.1+)
2. A compiler toolchain with XML/XSLT development headers (for building Gem dependencies)
3. A sane, working Ruby environment (Ruby 1.9.3 or greater)
4. Git

It is outside the scope of this post to cover how to get all those
installed.

Once those are installed:

```
% vagrant plugin install vagrant-berkshelf
% gem install berkshelf
% gem install test-kitchen --pre
% gem install kitchen-vagrant
```

Test Kitchen combines the suite (default) with the platform names
(e.g., ubuntu-12.04). To run all the suites on all platforms, simply do:

```
% kitchen test
```

This will take awhile, especially if you don't already have the
Vagrant boxes on your system, as it will download each one. To make
this faster, we'll just run Ubuntu 12.04:

```
% kitchen test default.*1204
```

Test Kitchen 1.0 can take a regular expression for the instances to
test. This will match the box `default-ubuntu-12.04`. I could also
just say `12` as that will match the single entry in my kitchen list
(above).

It will take a few minutes to run Test Kitchen. Those familiar with
Chef know that if it encounters an unhandled exception, it exits with
a non-zero return code. This is important, because we know at the end
of a successful run, Chef did the right thing, assuming our recipe is
the right thing :-).

To recap the [previous post](/blog/2013/03/19/anatomy-of-a-test-kitchen-1-dot-0-cookbook-part-1/), we have a run list like this:

```
["recipe[apt]", "recipe[minitest-handler]", "recipe[bluepill_test]"]
```

Let's break down the output of our successful run. I'll show the
output first, and explain it after:

```
Starting Kitchen
Cleaning up any prior instances of <default-ubuntu-1204>
Destroying <default-ubuntu-1204>
Finished destroying <default-ubuntu-1204> (0m0.00s).
Testing <default-ubuntu-1204>
Creating <default-ubuntu-1204>
```

This is basic setup to ensure that "The Kitchen" is clean beforehand
and we don't have existing state interfering with the run.

```
[vagrant command] BEGIN (vagrant up default-ubuntu-1204 --no-provision)
[default-ubuntu-1204] Importing base box 'canonical-ubuntu-12.04'...
[default-ubuntu-1204] Matching MAC address for NAT networking...
[default-ubuntu-1204] Clearing any previously set forwarded ports...
[default-ubuntu-1204] Forwarding ports...
[default-ubuntu-1204] -- 22 => 2222 (adapter 1)
```

This will look familiar to Vagrant users, we're just getting some
basic setup from Vagrant initializing the box defined in the
`.kitchen.yml` (passed to the Vagrantfile by the kitchen-vagrant
plugin). This step does a `vagrant up --no-provision`.

```
[Berkshelf] installing cookbooks...
[Berkshelf] Using bluepill (2.2.2) at path: '/Users/jtimberman/Development/opscode/cookbooks/bluepill'
[Berkshelf] Using apt (1.8.4)
[Berkshelf] Using yum (2.0.0)
[Berkshelf] Using minitest-handler (0.1.2)
[Berkshelf] Using bluepill_test (0.0.1) at path: './test/cookbooks/bluepill_test'
[Berkshelf] Using rsyslog (1.5.0)
[Berkshelf] Using chef_handler (1.1.0)
```

Remember from the [previous post](/blog/2013/03/19/anatomy-of-a-test-kitchen-1-dot-0-cookbook-part-1/) that we're using Berkshelf? This is
the integration with Vagrant that ensures that the cookbooks are
available. The first four, `apt`, `yum`, `minitest-handler` and
bluepill_test are defined in the Berksfile. The next, `rsyslog` is a
dependency of the `bluepill` cookbook (for rsyslog integration), and the
last, `chef_handler` is a dependency of `minitest-handler`. Berkshelf
extracts the dependencies from the cookbook metadata of each cookbook
defined in the Berksfile.

```
[default-ubuntu-1204] Creating shared folders metadata...
[default-ubuntu-1204] Clearing any previously set network interfaces...
[default-ubuntu-1204] Running any VM customizations...
[default-ubuntu-1204] Booting VM...
[default-ubuntu-1204] Waiting for VM to boot. This can take a few minutes.
[default-ubuntu-1204] VM booted and ready for use!
[default-ubuntu-1204] Setting host name...
[default-ubuntu-1204] Mounting shared folders...
[default-ubuntu-1204] -- v-root: /vagrant
[default-ubuntu-1204] -- v-csc-1: /tmp/vagrant-chef-1/chef-solo-1/cookbooks
[vagrant command] END (0m48.76s)
Vagrant instance <default-ubuntu-1204> created.
Finished creating <default-ubuntu-1204> (0m53.12s).
```

Again, this is familiar output to Vagrant users, where Vagrant is
making the cookbooks available to the instance.

```
Converging <default-ubuntu-1204>
[vagrant command] BEGIN (vagrant ssh default-ubuntu-1204 --command 'should_update_chef() {\n...')
Installing Chef Omnibus (11.4.0)
Downloading Chef 11.4.0 for ubuntu...
Installing Chef 11.4.0
Selecting previously unselected package chef.
g database ...        60513 files and directories currently installed.)
Unpacking chef (from .../chef_11.4.0_amd64.deb) ...
Setting up chef (11.4.0-1.ubuntu.11.04) ...
Thank you for installing Chef!
[vagrant command] END (0m34.85s)
[vagrant command] BEGIN (vagrant provision default-ubuntu-1204)
[Berkshelf] installing cookbooks...
[Berkshelf] Using bluepill (2.2.2) at path: '/Users/jtimberman/Development/opscode/cookbooks/bluepill'
[Berkshelf] Using apt (1.8.4)
[Berkshelf] Using yum (2.0.0)
[Berkshelf] Using minitest-handler (0.1.2)
[Berkshelf] Using bluepill_test (0.0.1) at path: './test/cookbooks/bluepill_test'
[Berkshelf] Using rsyslog (1.5.0)
[Berkshelf] Using chef_handler (1.1.0)
```

This part is interesting, in that we're going to install the Full
Stack Chef (Omnibus) package. This means it doesn't matter what the
underlying base box has installed, we get the right version of Chef.
This is defined in the `.kitchen.yml`. This is done through `vagrant
ssh` (second line). Then, Test Kitchen does `vagrant provision`. The
provisioning step is where Berkshelf happens, so we do see this happen
again (perhaps a bug?).

```
[default-ubuntu-1204] Running provisioner: Vagrant::Provisioners::ChefSolo...
[default-ubuntu-1204] Generating chef JSON and uploading...
[default-ubuntu-1204] Running chef-solo...
INFO: *** Chef 11.4.0 ***
INFO: Setting the run_list to ["recipe[apt]", "recipe[minitest-handler]", "recipe[bluepill_test]"] from JSON
INFO: Run List is [recipe[apt], recipe[minitest-handler], recipe[bluepill_test]]
INFO: Run List expands to [apt, minitest-handler, bluepill_test]
INFO: Starting Chef Run for default-ubuntu-1204.vagrantup.com
```

This is the start of the actual Chef run, using Chef Solo by Vagrant's
provisioner. Note that we have our suite's run list. I'm going to skip
a lot of the Chef output because it isn't required. Note that a few
resources in the minitest--handler will report as failed, but they can
be ignored because it means that those tests were simply not implemented.

```
INFO: Processing directory[/var/chef/minitest/bluepill_test] action create (minitest-handler::default line 50)
INFO: directory[/var/chef/minitest/bluepill_test] created directory /var/chef/minitest/bluepill_test
INFO: Processing cookbook_file[tests-bluepill_test-default] action create (minitest-handler::default line 53)
INFO: cookbook_file[tests-bluepill_test-default] created file /var/chef/minitest/bluepill_test/default_test.rb
INFO: Processing remote_directory[tests-support-bluepill_test-default] action create (minitest-handler::default line 60)
INFO: remote_directory[tests-support-bluepill_test-default] created directory /var/chef/minitest/bluepill_test/support
INFO: Processing cookbook_file[/var/chef/minitest/bluepill_test/support/helpers.rb] action create (dynamically defined)
INFO: cookbook_file[/var/chef/minitest/bluepill_test/support/helpers.rb] mode changed to 644
INFO: cookbook_file[/var/chef/minitest/bluepill_test/support/helpers.rb] created file /var/chef/minitest/bluepill_test/support/helpers.rb
```

These are the relevant parts of the minitest-handler recipe, where it
has copied the tests from the `bluepill_test` cookbook into place.

```
INFO: Processing gem_package[i18n] action install (bluepill::default line 20)
INFO: Processing gem_package[bluepill] action install (bluepill::default line 24)
INFO: Processing directory[/etc/bluepill] action create (bluepill::default line 34)
INFO: directory[/etc/bluepill] created directory /etc/bluepill
INFO: directory[/etc/bluepill] owner changed to 0
INFO: directory[/etc/bluepill] group changed to 0
INFO: Processing directory[/var/run/bluepill] action create (bluepill::default line 34)
INFO: directory[/var/run/bluepill] created directory /var/run/bluepill
INFO: directory[/var/run/bluepill] owner changed to 0
INFO: directory[/var/run/bluepill] group changed to 0
INFO: Processing directory[/var/lib/bluepill] action create (bluepill::default line 34)
INFO: directory[/var/lib/bluepill] created directory /var/lib/bluepill
INFO: directory[/var/lib/bluepill] owner changed to 0
INFO: directory[/var/lib/bluepill] group changed to 0
INFO: Processing file[/var/log/bluepill.log] action create_if_missing (bluepill::default line 41)
INFO: entered create
INFO: file[/var/log/bluepill.log] owner changed to 0
INFO: file[/var/log/bluepill.log] group changed to 0
INFO: file[/var/log/bluepill.log] mode changed to 755
INFO: file[/var/log/bluepill.log] created file /var/log/bluepill.log
```

Recall from the [previous post](/blog/2013/03/19/anatomy-of-a-test-kitchen-1-dot-0-cookbook-part-1/) that the `bluepill_test` recipe includes
the `bluepill` recipe. This is the basic setup of bluepill.

```
INFO: Processing package[nc] action install (bluepill_test::default line 4)
INFO: Processing template[/etc/bluepill/test_app.pill] action create (bluepill_test::default line 16)
INFO: template[/etc/bluepill/test_app.pill] updated content
INFO: Processing bluepill_service[test_app] action enable (bluepill_test::default line 18)
INFO: Processing bluepill_service[test_app] action load (bluepill_test::default line 18)
INFO: Processing bluepill_service[test_app] action start (bluepill_test::default line 18)
INFO: Processing link[/etc/init.d/test_app] action create (/tmp/vagrant-chef-1/chef-solo-1/cookbooks/bluepill/providers/service.rb line 30)
INFO: link[/etc/init.d/test_app] created
INFO: Chef Run complete in 81.099185824 seconds
```

And this is the rest of the `bluepill_test` recipe. It sets up a test
service that will basically be a netcat process listening on a port.
Let's take a moment here and discuss what we have.

First, we have successfully converged the default recipe in the
`bluepill` cookbook via its inclusion in `bluepill_test`. This is
awesome, because we know the recipe works exactly as we defined it,
since Chef resources are declarative, and Chef exits if there's a
problem.

Second, we have successfully setup a service managed by bluepill
itself using the LWRP included in the `bluepill` cookbook,
`bluepill_service`. This means we know that the underlying provider
configured all the resources correctly.

At this point, we could say "Ship it!" and release the cookbook,
knowing it will do what we require. However, this may be disingenuous
because we don't know if the behavior of the system after all this
runs is actually correct. Therefore we look to the next segment of
output from Chef, from minitest:

```
INFO: Running report handlers
Run options: -v --seed 38794
\# Running tests:
recipe::bluepill_test::default#test_0001_the_default_log_file_must_exist_cook_1295_ =
0.00 s = .
recipe::bluepill_test::default::create a bluepill configuration file#test_0001_anonymous =
0.00 s = .
recipe::bluepill_test::default::create a bluepill configuration file#test_0002_must_be_valid_ruby =
0.06 s = .
recipe::bluepill_test::default::runs the application as a service#test_0001_anonymous =
0.72 s = .
recipe::bluepill_test::default::runs the application as a service#test_0002_anonymous =
0.71 s = .
recipe::bluepill_test::default::spawn a netcat tcp client repeatedly#test_0001_should_receive_a_tcp_connection_from_netcat =
2.24 s = .
Finished tests in 3.746002s, 1.6017 tests/s, 1.8687 assertions/s.
6 tests, 7 assertions, 0 failures, 0 errors, 0 skips
```

This is performed by the minitest-handler, which runs the tests copied
from the `bluepill_test` cookbook before. It's outside the scope of
this post to describe how to write minitest-chef tests, but we can
talk about the output.

We have 6 separate tests that perform 7 assertions, and they all
passed. The tests are asserting:

1. The log file is created, and by the full name of the test, this is
to check for a regression from
[COOK-1295](http://tickets.opscode.com/browse/COOK-1295).
2. The `.pill` config file for the service must exist and be valid
Ruby.
3. The bluepill service must actually be enabled and running, thereby
testing that those actions in the LWRP work.
4. The running service, which listens on a TCP port, must be up and
available, thereby testing that bluepill started the service
correctly.

```
[vagrant command] END (1m29.24s)
Finished converging <default-ubuntu-1204> (2m15.45s).
Setting up <default-ubuntu-1204>
Finished setting up <default-ubuntu-1204> (0m0.00s).
Verifying <default-ubuntu-1204>
Finished verifying <default-ubuntu-1204> (0m0.00s).
Destroying <default-ubuntu-1204>
[vagrant command] BEGIN (vagrant destroy default-ubuntu-1204 -f)
[default-ubuntu-1204] Forcing shutdown of VM...
[Berkshelf] cleaning Vagrant's shelf
[default-ubuntu-1204] Destroying VM and associated drives...
[vagrant command] END (0m3.68s)
Vagrant instance <default-ubuntu-1204> destroyed.
Finished destroying <default-ubuntu-1204> (0m4.04s).
Finished testing <default-ubuntu-1204> (3m12.62s).
Kitchen is finished. (3m12.62s)
```

This output shows Test Kitchen cleaning up after itself. We destroy
the Vagrant instance on a successful convergence and test run in Chef,
because further investigation is not required. If the test failed for
some reason, Test Kitchen leaves it running so you can log into the
machine and poke around to find out what went wrong. Then simply
correct the required part of the cookbook (recipes, tests, etc) and
rerun Test Kitchen. For example:

```
% bundle exec kitchen login 1204
vagrant@ubuntu-1204$ ... run some commands
vagrant@ubuntu-1204$ ^D
% bundle exec kitchen converge 1204
```

My goal with these posts is to get some information out for folks to
consider when examining Test Kitchen 1.0 alpha for their own projects.
There's a lot more to Test Kitchen, such as managing non-cookbook
projects, or even using other kinds of tests. We'll have more
documentation and guides as we get the 1.0 release out.

Enjoy!
