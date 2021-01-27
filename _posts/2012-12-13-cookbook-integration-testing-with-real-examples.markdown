---
layout: post
title: "Cookbook Integration Testing With Real Examples"
date: 2012-12-13 20:54
comments: true
categories: [chef, development]
---

This blog post starts with a [gist](https://gist.github.com/4281314),
and a
[tweet](https://twitter.com/jtimberman/status/279380189287436289).
However, that isn't the whole story. Read on...

Today I released version 1.6.0 of
[Opscode's apt cookbook](http://community.opscode.com/cookbooks/apt). The
cookbook itself needed better coverage for testing in Test Kitchen.
This post will describe these additions to the cookbook, including how
one of the test recipes can actually be used for actual production
use. My goal is to explain a bit about how we go about testing with
Test Kitchen, and provide some real world examples.

TL;DR -
[This commit has all the code](https://github.com/opscode-cookbooks/apt/commit/9996510bab27a9ea5f17e2f6362151dcbe708e89).

Kitchenfile
===========

First, the Kitchenfile for the project looked like this:

```ruby
cookbook "apt" do
  runtimes []
end
```

This is outdated as far as Kitchenfiles goes. It still has the empty
array runtimes setting which prevents Test Kitchen from attempting to
[run additional tests under RVM](http://tickets.opscode.com/browse/KITCHEN-4).
We'll remove this line, and update it for supporting the
configurations of the recipes and features we want to test. The
cookbook itself has three recipes:

* default.rb
* cacher-client.rb
* cacher-ng.rb

By default, with no configurations defined in a Kitchenfile,
test-kitchen will run the default recipe (using Chef Solo under
Vagrant). This is useful in the common case, but we also want to
actually test other functionality in the cookbook. In addition to the
recipes, we want to verify that the LWRPs will do what we intend.

I updated the Kitchenfile with the following content:

```ruby
cookbook "apt" do
  configuration "default"
  configuration "cacher-ng"
  configuration "lwrps"
end
```

A configuration can correspond to a recipe (`default`, `cacher-ng`),
but it can also be arbitrarily named. This is a name used by kitchen
test. The `cacher-client` recipe isn't present because
`recipe[apt::cacher-ng]` includes it, and getting the test to work,
where the single node is a cacher client to itself, was prone to
error. "I assure you, it works" :-). We'll look at this later anyway.

With the above Kitchenfile, kitchen test will start up the Vagrant
VMs and attempt to run Chef Solo with the recipes named by the
configuration. This is a good start, but we want to actually run some
minitest-chef tests. These will be created inside a "test" cookbook
included with this cookbook. I created a cookbook named `apt_test`
under `./test/kitchen/cookbooks` using:

```text
knife cookbook create apt_test -o ./test/kitchen/cookbooks
```

This creates the cookbook scaffolding like normal. I cleaned up the
contents of the directory to contain what I needed to start:

```text
test/kitchen/cookbooks//apt_test/metadata.rb
test/kitchen/cookbooks//apt_test/README.md
test/kitchen/cookbooks//apt_test/recipes
test/kitchen/cookbooks//apt_test/recipes/cacher-ng.rb
test/kitchen/cookbooks//apt_test/recipes/default.rb
test/kitchen/cookbooks//apt_test/recipes/lwrps.rb
```

The metadata.rb is as you'd expect, it contains the name, a version,
maintainer information and a description. The README simply mentions
that this is a test cookbook for the parent project. The recipes are
the interesting part. Let's address them in the order of the
configurations in the Kitchenfile.

Configuration: default
======================

First, the default recipe in the test cookbook. This is simply going
to perform an `include_recipe "apt::default"`. The way test kitchen
runs, it will actually have the following run list for Chef Solo:

```text
[test-kitchen::default, minitest-handler, apt_test::default, apt::default]
```

`test-kitchen` sets up some essential things for Test Kitchen itself.
`minitest-handler` is the recipe that sets up minitest-chef-handler to
run post-convergence tests. `apt_test::default` is the "test" recipe
for this configuration, and finally `apt::default` is the cookbook's
recipe for this configuration named "default".

Had we not done anything else here, the results are the same as simply
running test kitchen with the original Kitchenfile (with runtimes,
instead of configurations defined).

## Minitest: default recipe

There are now minitest-chef tests for each configuration. The default
recipe provides some "apt-get update" executes, and also creates a
directory that can be used for preseeding packages. We'll simply test
that the preseeding directory exists. We could probably check that the
cache is updated, but since this cookbook has worked for almost 4
years w/o issue for `apt-get update` we'll trust it continues working
:-). Here's the test (ignoring the boilerplate):

```ruby
  it 'creates the preseeding directory' do
    directory('/var/cache/local/preseeding').must_exist
  end
```

When Chef runs, it will run this test:

```text
apt_test::default#test_0001_creates_the_preseeding_directory = 0.00 s = .
Finished tests in 0.007988s, 125.1808 tests/s, 125.1808 assertions/s.
1 tests, 1 assertions, 0 failures, 0 errors, 0 skips
```

Configuration: cacher-ng
========================

Next, Test Kitchen runs the `cacher-ng` configuration. The recipe in
the `apt_test` cookbook simply includes the `apt::cacher-ng` recipe.
The run list in Chef Solo looks like this:

```text
[test-kitchen::default, minitest-handler, apt_test::cacher-ng, apt::cacher-ng]
```

The `apt::cacher-ng` recipe also includes the client recipe, but
basically does nothing unless the `cacher_ipaddress` attribute is set,
or if we can search using a Chef Server (which Solo can't, of course).

## Minitest: cacher-ng recipe

The meat of the matter for the `cacher-ng` recipe is running the
`apt-cacher-ng` service, so we've written a minitest test for this:

```ruby
  it 'runs the cacher service' do
    service("apt-cacher-ng").must_be_running
  end
```

And when Chef runs the test:

```text
apt_test::default#test_0001_runs_the_cacher_service = 0.06 s = .
Finished tests in 0.067250s, 14.8698 tests/s, 14.8698 assertions/s.
1 tests, 1 assertions, 0 failures, 0 errors, 0 skips
```

Configuration: lwrps
====================

Finally, we have our custom configuration that doesn't correspond to a
recipe in the `apt` cookbook, `lwrps`. This configuration instead is
to do a real-world integration test that the LWRPs actually do what
they're supposed to.

## Recipe: apt_test::lwrps

The recipe itself looks like this:

```ruby
include_recipe "apt"

apt_repository "opscode" do
  uri "http://apt.opscode.com"
  components ["main"]
  distribution "#{node['lsb']['codename']}-0.10"
  key "2940ABA983EF826A"
  keyserver "pgpkeys.mit.edu"
  action :add
end

apt_preference "chef" do
  pin "version 10.16.2-1"
  pin_priority "700"
end
```

The `apt` recipe is included because otherwise, we may not be able to
notify the `apt-get update` resource to execute when the new sources.list
is dropped off.

Next, we use Opscode's very own apt repository as an example because
we can rely on that existing. When Test Kitchen runs, it will actually
write out the apt repository configuration file to
`/etc/apt/sources.list.d/opscode.list`, but more on that in a minute.

Finally, we're going to write out an apt preferences file for pinning
the Chef package. Currently, Chef is actually packaged at various
versions in Ubuntu releases:

```text
% rmadison chef
      chef | 0.7.10-0ubuntu1.1 | lucid/universe | source, all
      chef | 0.8.16-4.2 | oneiric/universe | source, all
      chef |  10.12.0-2 | quantal/universe | source, all
      chef |  10.12.0-2 | raring/universe | source, all
```

So by adding the Opscode APT repository, and pinning Chef, we can
ensure that we're going to have the correct version of Chef installed
as a package, if we were installing Chef as a package from APT :).

When Chef Solo runs, here is the run list:

```text
[test-kitchen::default, minitest-handler, apt_test::lwrps]
```

Notice it doesn't have "`apt::lwrps`", since that isn't a recipe in
the apt cookbook.

## Minitest: lwrps recipe

The minitest tests for the lwrps configuration and recipe look like this:

```ruby
  it 'creates the Opscode sources.list' do
    file("/etc/apt/sources.list.d/opscode.list").must_exist
  end

  it 'adds the Opscode package signing key' do
    opscode_key = shell_out("apt-key list")
    assert opscode_key.stdout.include?("Opscode Packages <packages@opscode.com>")
  end

  it 'creates the correct pinning preferences for chef' do
    chef_policy = shell_out("apt-cache policy chef")
    assert chef_policy.stdout.include?("Package pin: 10.16.2-1")
  end
```

The first test simply asserts that the Opscode APT sources.list is
present. We could elaborate on this by verifying that its content is
correct, but for now we're going to trust that the declarative
resource in the recipe is, ahem, declared properly.

Next, we run the `apt-key` command to show the available GPG keys in
the APT trusted keyring. This will have the correct Opscode Packages
key if it was added correctly.

Finally, we test that the package pinning for the Chef package is
correct. Successful output of the tests looks like this:

```text
apt_test::default#test_0003_creates_the_correct_pinning_preferences_for_chef = 0.05 s = .
apt_test::default#test_0002_adds_the_opscode_package_signing_key = 0.05 s = .
apt_test::default#test_0001_creates_the_opscode_sources_list = 0.00 s = .
Finished tests in 0.112725s, 26.6133 tests/s, 26.6133 assertions/s.
3 tests, 3 assertions, 0 failures, 0 errors, 0 skips
```

The Real World Bits
===================

The tests hide some of the detail. What does this actually look like
on a real system? Glad you asked!

Here's the sources.list for Opscode's APT repository.

```text
vagrant@ubuntu-12-04:~$ cat /etc/apt/sources.list.d/opscode.list
deb     http://apt.opscode.com precise-0.10 main
```

Next, the apt-key content:

```text
vagrant@ubuntu-12-04:~$ sudo apt-key list
(snip, ubuntu's keys)
pub   1024D/83EF826A 2009-07-24
uid                  Opscode Packages <packages@opscode.com>
sub   2048g/3B6F42A0 2009-07-24
```

And the grand finale, the pinning preferences:

```text
vagrant@ubuntu-12-04:~$ apt-cache policy chef
chef:
  Installed: 10.14.4-2.ubuntu.11.04
  Candidate: 10.16.2-1
  Package pin: 10.16.2-1
  Version table:
     10.16.2-1 700
        500 http://apt.opscode.com/ precise-0.10/main amd64 Packages
 *** 10.14.4-2.ubuntu.11.04 700
        100 /var/lib/dpkg/status
```

I used [Opscode's bento box](https://github.com/opscode/bento) for
Ubuntu 12.04, which comes with the 'omnibus' Chef package version
10.14.4(-2.ubuntu.11.04). In order to install the newer Chef package
and demonstrate the pinning, I'll first remove it:

```text
vagrant@ubuntu-12-04:~$ sudo dpkg --purge chef
```

Then, I install from the Opscode APT repository:

```text
vagrant@ubuntu-12-04:~$ sudo apt-get install chef
...
Setting up chef (10.16.2-1) ...
...
```

And the package is installed:

```text
vagrant@ubuntu-12-04:~$ apt-cache policy chef
chef:
  Installed: 10.16.2-1
  Candidate: 10.16.2-1
  Package pin: 10.16.2-1
  Version table:
 *** 10.16.2-1 700
        500 http://apt.opscode.com/ precise-0.10/main amd64 Packages
        100 /var/lib/dpkg/status
```

Currently the omnibus packages are *NOT* in the APT repository, since
they do not have additional dependencies they are installed simply
with dpkg. Don't use this particular recipe if you're using the
Omnibus packages. Instead, just marvel at the utility of this. Perhaps
instead, use the LWRPs in the apt cookbook to set up your own local
APT repository and pinning preferences.

Conclusion
==========

Test Kitchen is a framework for isolated integration testing in
individual projects. As such, it has a lot of features, capabilities
and also moving parts. Hopefully this post helps you understand some
of them, and see how it works, and how you may be able to use it for
yourself. Or, if you want, simply grab the `apt_test::lwrps` recipe's
contents and stick them in your own cookbook that manages Chef package
installation and move along. :-)

All the code used in this post is available in the [Opscode Cookbook's
organization "apt" repository](https://github.com/opscode-cookbooks/apt/).

Further Reading
---------------

* [Learn about Test Kitchen](https://github.com/opscode/test-kitchen)
* [More about Minitest-Chef](https://github.com/calavera/minitest-chef-handler)
* [apt cookbook commit](https://github.com/opscode-cookbooks/apt/commit/9996510bab27a9ea5f17e2f6362151dcbe708e89)
* [BDD Testing Infrastructure](http://www.opscode.com/blog/2012/07/20/on-the-level-testing-your-infrastructure/)
