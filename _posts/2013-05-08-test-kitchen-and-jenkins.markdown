---
layout: post
title: "Test Kitchen and Jenkins"
date: 2013-05-08 23:53
comments: true
categories: [chef, development]
---

I've been working more with test-kitchen 1.0 alpha lately. The most
recent thing I've done is set up a Jenkins build server to run
test-kitchen on cookbooks. This post will describe how I did this for
my own environment, and how you can use my new test-kitchen cookbook
in yours... if you're using Jenkins, anyway.

This is all powered by a relatively simple cookbook, and some
click-click-clicking in the Jenkins UI. I'll walk through what I did
to set up my Jenkins system.

First, I started with Debian 7.0 (stable, released this past weekend).
I installed the OS on it, and then bootstrapped with Chef. The initial
test was to make sure everything installed correctly, and the commands
were functioning. This was done in a VM, and is now handled by
test-kitchen itself (how meta!) in the cookbook, kitchen-jenkins.

The cookbook, [kitchen-jenkins](http://ckbk.it/kitchen-jenkins) is
available on the Chef Community site. I started with a recipe, but
extracted it to a cookbook to make it easier to share with you all.
This is essentially a site cookbook that I use to customize my Jenkins
installation so I can run test-kitchen builds.

I apply the recipe with a role, because I love the roles primitive in
Chef :-). Here is the role I'm using:

```javascript
{
  "name": "jenkins",
  "description": "Jenkins Build Server",
  "run_list": [
    "recipe[kitchen-jenkins]"
  ],
  "default_attributes": {
    "jenkins": {
      "server": {
        "home": "/var/lib/jenkins",
        "plugins": ["git-client", "git"],
        "version": "1.511",
        "war_checksum": "7e676062231f6b80b60e53dc982eb89c36759bdd2da7f82ad8b35a002a36da9a"
      }
    }
  },
  "json_class": "Chef::Role",
  "chef_type": "role"
}
```

The run list is only slightly different here than my actual role, I
have a few other things in the run list, which are other site-specific
recipes. Don't worry about those now. The jenkins attributes are set
to ensure the right plugins I need are available, and the right
version of jenkins is installed.

(I'm going to leave out the details such as uploading cookbooks and
roles, if you're interested in test-kitchen, I'll assume you've got
that covered :-).)

Once Chef completes on the Jenkins node, I can reach the Jenkins UI,
conveniently enough, via "http://jenkins:8080" (because I've made a
DNS entry, of course). The next release of the Jenkins cookbook will
have a resource for managing jobs, but for now I'm just going to
create them in the webui.

For this example, I want to have two kinds of cookbook testing jobs.
The first, is to simply run foodcritic and fail on any correctness
matches. Second, I want to actually run test-kitchen.

A foodcritic job is simple:

1. New job -> Build a free-style software project
   "foodcritic-COOKBOOK".
2. Source Code Management -> Git, supply the repository and the master
   branch.
3. Set a build trigger to Poll SCM every 5 minutes, once an hour,
   whenever you like.
4. Add a build step to execute a shell, "foodcritic . -f correctness"

I created a view for foodcritic jobs, and added them all to the view
for easy organizing.

Next, I create a test-kitchen job:

1. New job -> Copy existing job "foodcritic-COOKBOOK", name the new
   job "test-COOKBOOK".
2. Uncheck Poll SCM, check "Build after other projects are built" and
   enter "foodcritic-COOKBOOK".
3. Replace the foodcritic command in the build shell command with
   "kitchen test".

Now, the test kitchen test will only run if the foodcritic build
succeeds. If the cookbook has any correctness lint errors, then the
foodcritic build fails, and the kitchen build won't run. This will
help conserve resources.

Hopefully the `kitchen-jenkins` cookbook is helpful and this blog post
will give you some ideas how to go about adding cookbook tests to your
CI system, even if it's not Jenkins.
