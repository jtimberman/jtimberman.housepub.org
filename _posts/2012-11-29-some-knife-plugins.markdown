---
layout: post
title: "Some Knife Plugins"
date: 2012-11-29 19:11
comments: true
categories: [chef, ruby, plugins]
---

I've shared my `~/.chef/plugins/knife` directory as a Git repository
on [GitHub](https://github.com/jtimberman/chef-plugins-knife). There's
only a few, but I hope you find them useful. They are licensed under
the Apache 2.0 software license, but please only use them for awesome.

## gem

This plugin will install a gem into the Ruby environment that knife is
executing in. This is handy if you want to install knife plugins that
are gems.

If you have Ruby and Chef/Knife installed in an area where your user
can write:

```
knife gem install knife-config
```

If you're using an Omnibus package install of Chef, or otherwise
require root access to install:

```
knife gem install knife-config
```

**Note** If you're trying to install a gem for _Chef_ to use, you
  should put it in a `chef_gem` resource in a recipe.

## metadata

This plugin prints out information from a cookbook's metadata. It
currently only works with `metadata.rb` files, and not `metadata.json`
files.

In a cookbook's directory, display the cookbook's dependencies:

```
knife metadata dependencies
```

Show the dependencies and supported platforms:

```
knife metadata dependencies platforms
```

Use the `-P` option to pass a path to a cookbook.

```
knife metadata name dependencies -P ~/.berkshelf/cookbooks/rabbitmq-1.6.4
```

## nukular

[I wrote on this blog about this plugin awhile ago](https://jtimberman.housepub.org/blog/2012/02/15/testing-with-fission/).

This plugin cleans up after running `chef-client` on a VMware Fusion machine.

```
knife nukular guineapig base guineapig.int.example.com
```

## plugin_create

This creates a plugin scaffolding in `~/.chef/plugins/knife`. It will
join underscored words as CamelCaseClasses.

For example,

```
knife plugin create awesometown
```

Creates a plugin that is class `Awesometown` that can be executed with:

```
knife awesometown
```

Whereas this,

```
knife plugin create awesome_town
```

Creates a plugin that is class `AwesomeTown` that can be executed
with:

```
knife plugin awesome town
```
