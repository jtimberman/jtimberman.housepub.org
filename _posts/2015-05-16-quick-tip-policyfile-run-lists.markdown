---
layout: post
title: "Quick Tip: Policyfile Run Lists"
date: 2015-05-16 21:52:17 -0600
comments: true
categories: [quicktips, chef]
---

As I indicated on [Twitter earlier tonight](https://twitter.com/jtimberman/status/599779453456097280), I'm working with the new Policyfile feature of [ChefDK](https://github.com/chef/chef-dk/blob/master/POLICYFILE_README.md). While converting my personal systems' repository to use Policyfile instead of roles, I found myself writing this Policyfile:

```ruby
name 'home_server'
default_source :community
run_list(
         'build-essential',
         'packages',
         'users',
         'sudo',
         'runit',
         'ntp',
         'openssh',
         'postfix'
        )

cookbook 'build-essential'
cookbook 'packages', git: 'https://github.com/mattray/packages-cookbook', branch: 'multipackage'
cookbook 'users', path: '../housepub-chef-repo/cookbooks/users'
cookbook 'sudo'
cookbook 'runit'
cookbook 'ntp'
cookbook 'openssh'
cookbook 'postfix'
```

No big deal, but I found the repetition... redundant. Several of these cookbooks are fine floating on the latest version from Supermarket - everything but `packages` and `users`. So I thought, "wouldn't it be great if entries in the run list were automatically added as dependencies?"

Then, I added `chef-client-runit` to the run list, but I didn't add it as a `cookbook` entry, performed the `chef update`, and reran my `chef provision` command, and wound up with `chef-client-runit` being converged.

To illustrate this with a really simple example, I confirmed with `zsh`:

    % cd ~/Development/sandbox/test
    % chef generate policyfile

Then edit the `Policyfile.rb`:

```ruby
name "example_application"
default_source :community
run_list "zsh"
```

Note that there is no `cookbook` line here.

    % chef install
    Building policy example_application
    Expanded run list: recipe[zsh]
    Caching Cookbooks...
    Using      zsh 1.0.1

    Lockfile written to /Users/jtimberman/Development/sandbox/test/Policyfile.lock.json

And if we examine the `Policyfile.lock.json`, we see:

```json
{
  "revision_id": "e8b5b48d35f4a8efcd037ef6c9cc8e34f901561ffef160bd0a57ca1b612a1179",
  "name": "example_application",
  "run_list": [
    "recipe[zsh::default]"
  ],
  "cookbook_locks": {
    "zsh": {
      "version": "1.0.1",
      "identifier": "b512ef33af29b8d34ad7e4e9b6ad38d42dea4945",
      "dotted_decimal_identifier": "50967789358229944.59473511204894381.62483954551109",
      "cache_key": "zsh-1.0.1-supermarket.chef.io",
      "origin": "https://supermarket.chef.io/api/v1/cookbooks/zsh/versions/1.0.1/download",
      "source_options": {
        "artifactserver": "https://supermarket.chef.io/api/v1/cookbooks/zsh/versions/1.0.1/download",
        "version": "1.0.1"
      }
    }
  } <SNIP>
```

Yay! Shoutout to [Dan DeLeo](https://twitter.com/kallistec) for preemptively implementing features I didn't even know I wanted yet :-).
