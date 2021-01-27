---
layout: post
title: "Preview Chef Client Local Mode"
date: 2013-10-10 23:39
comments: true
categories: [chef]
---

Opscode Developer John Keiser
[mentioned](https://twitter.com/jkeiser2/status/388460927026085888)
that a feature for Chef Zero he's been working on, "local mode," is
now in Chef's master branch. This means it should be in the next
release (11.8). I took the liberty to check this *unreleased* feature
out.

Let's just say, it's super awesome and John has done some amazing work
here.

## PREVIEW

This is a preview of an unreleased feature in Chef. All standard
disclaimers apply :).

## Install

This is in the master branch of Chef, not released as a gem yet.
You'll need to get the source and build a gem locally. This totally
assumes you've installed a sane ruby and bundler on your system.

```sh
git clone git://github.com/opscode/chef.git
cd chef
bundle install
bundle exec rake gem
gem install  pkg/chef-11.8.0.alpha.0.gem
```

**Note** Alpha!

## Setup

Next, point it at a local repository. I'll use a simple example.

```sh
git clone git://github.com/opscode/chef-repo.git
cd chef-repo
knife cookbook create zero -o ./cookbooks
vi cookbooks/zero/recipes/default.rb
```

I created a fairly trivial example recipe to show that this will
support search, and data bag items:

```ruby
a = search(:node, "*:*")
b = data_bag_item("zero", "fluff")

file "/tmp/zerofiles" do
  content a[0].to_s
end

file "/tmp/fluff" do
  content b.to_s
end
```

This simply searches for all nodes, and uses the content of the first
node (the one we're running on presumably) for a file in /tmp. It also
loads a data bag item (which I created) and uses it for the content of
another file in /tmp.

```sh
mkdir -p data_bags/zero
vi data_bags/zero/fluff.json
```

The data bag item:

```json
{
  "id": "fluff",
  "clouds": "Are fluffy"
}
```

## Converge!

Now, converge the node:

```sh
chef-client -z -o zero
```

The `-z`, or `--local-mode` argument is the magic that sets up Chef
Zero, and loads all the contents of the repository. The `-o zero`
tells Chef to use a one time run list of the "zero" recipe.

```
[2013-10-10T23:53:32-06:00] WARN: No config file found or specified on command line, not loading.
Starting Chef Client, version 11.8.0.alpha.0
[2013-10-10T23:53:36-06:00] WARN: Run List override has been provided.
[2013-10-10T23:53:36-06:00] WARN: Original Run List: [recipe[zero]]
[2013-10-10T23:53:36-06:00] WARN: Overridden Run List: [recipe[zero]]
resolving cookbooks for run list: ["zero"]
Synchronizing Cookbooks:
  - zero
Compiling Cookbooks...
Converging 2 resources
Recipe: zero::default
  * file[/tmp/zerofiles] action create
    - create new file /tmp/zerofiles
    - update content in file /tmp/zerofiles from none to 0a038a
        --- /tmp/zerofiles      2013-10-10 23:53:36.368059768 -0600
        +++ /tmp/.zerofiles20131010-6903-10cvytu        2013-10-10 23:53:36.368059768 -0600
        @@ -1 +1,2 @@
        +node[jenkins.int.housepub.org]
  * file[/tmp/fluff] action create
    - create new file /tmp/fluff
    - update content in file /tmp/fluff from none to d46bab
        --- /tmp/fluff  2013-10-10 23:53:36.372059683 -0600
        +++ /tmp/.fluff20131010-6903-1l3i1h     2013-10-10 23:53:36.372059683 -0600
        @@ -1 +1,2 @@
        +data_bag_item[fluff]
Chef Client finished, 2 resources updated
```

The diff output from each of the file resources shows that the content
does in fact come from the search (a node object was returned) and a
data bag item (a data bag item object was returned).

## What's Next?

Since this is a feature of Chef, it will be documented and released,
so look for that in the next version of Chef.

I can see this used for testing purposes, especially for recipes that
make use of combinations of data bags and search, such as Opscode's
[nagios cookbook](http://community.opscode.com/cookbooks/nagios).

## Questions

* Does it work with Berkshelf?

I don't know. Probably not (yet).

* Does it work with Test Kitchen?

I don't know. Probalby not (yet). Provisioners in test-kitchen
would need to be (re)written.

* Should I use this in production?

This is an unreleased feature in the master branch. What do you think?
:)

* When will this be released?

I don't know the schedule for 11.8.0. Soon?

* Where do I find out more, or get involved?

Join #chef-hacking in irc.freenode.net, the chef-dev mailing list, or
attend the Chef Community Summit (November 12-13, 2013 in Seattle).
