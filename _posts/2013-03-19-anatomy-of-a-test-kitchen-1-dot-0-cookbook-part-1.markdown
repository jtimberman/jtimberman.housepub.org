---
layout: post
title: "Anatomy of a Test Kitchen 1.0 Cookbook (Part 1)"
date: 2013-03-19 09:28
comments: true
categories: [chef, development]
---

**DISCLAIMER** Test Kitchen 1.0 is still in *alpha* at the time of
  this post.

**Update** Remove Gemfile and Vagrantfile

Let's take a look at the anatomy of a cookbook set up with
test-kitchen 1.0-alpha.

**Note** It is outside the scope of this post to discuss how to write
  minitest-chef tests or "test cookbook" recipes. Use the cookbook
  described below as an example to get ideas for writing your own.

This is the full directory tree of Opscode's
"[bluepill](http://ckbk.it/bluepill)" cookbook:

```
├── .kitchen.yml
├── Berksfile
├── CHANGELOG.md
├── CONTRIBUTING
├── LICENSE
├── README.md
├── TESTING.md
├── attributes
│   └── default.rb
├── metadata.rb
├── providers
│   └── service.rb
├── recipes
│   ├── default.rb
│   └── rsyslog.rb
├── resources
│   └── service.rb
├── templates
│   └── default
│       ├── bluepill_init.fedora.erb
│       ├── bluepill_init.freebsd.erb
│       ├── bluepill_init.rhel.erb
│       └── bluepill_rsyslog.conf.erb
└── test
    └── cookbooks
        └── bluepill_test
            ├── README.md
            ├── attributes
            │   └── default.rb
            ├── files
            │   └── default
            │       └── tests
            │           └── minitest
            │               ├── default_test.rb
            │               └── support
            │                   └── helpers.rb
            ├── metadata.rb
            ├── recipes
            │   └── default.rb
            └── templates
                └── default
                    └── test_app.pill.erb
```

I'll assume the reader is familiar with basic components of cookbooks
like "recipes," "templates," and the top-level documentation files, so
let's trim this down to just the areas of concern for Test Kitchen.

```
├── .kitchen.yml
├── Berksfile
└── test
    └── cookbooks
        └── bluepill_test
            ├── attributes
            │   └── default.rb
            ├── files
            │   └── default
            │       └── tests
            │           └── minitest
            │               ├── default_test.rb
            │               └── support
            │                   └── helpers.rb
            ├── recipes
            │   └── default.rb
            └── templates
                └── default
                    └── test_app.pill.erb
```

Note that this cookbook has a "test" cookbook. I'll get to that in a
minute.

First of all, we have the `.kitchen.yml`. This is the project
definition that describes what is required to run test kitchen itself.
This particular file tells Test Kitchen to bring up nodes of the
platforms we're testing with Vagrant, and defines the boxes with their
box names and URLs to download. You can view the full
[`.kitchen.yml` in the Git repo](https://github.com/opscode-cookbooks/bluepill/blob/master/.kitchen.yml).
For now, I'm going to focus on the `suite` stanza in the
`.kitchen.yml`. This defines how Chef will run when Test Kitchen
brings up the Vagrant machine.

```yaml
- name: default
  run_list:
  - recipe[minitest-handler]
  - recipe[bluepill_test]
  attributes: {bluepill: { bin: "/opt/chef/embedded/bin/bluepill" } }
```

Each platform has a recipe it will run with, in this case `apt` and
`yum`. Then the suite's run list is appended, so for example, the final run list of
the Ubuntu 12.04 node will be:

```
["recipe[apt]", "recipe[minitest-handler]", "recipe[bluepill_test]"]
```

We have apt so the apt cache on the node is updated before Chef does
anything else. This is pretty typical so we put it in the default run
list of each Ubuntu box.

The `minitest-handler` recipe existing in the run list means that the
Minitest Chef Handler will be run at the end of the Chef run. In this
case, it will use the tests from the test cookbook, `bluepill_test`.

The bluepill cookbook itself does not depend on any of these
cookbooks. So how does Test Kitchen know where to get them? Enter the
next file in the list above, `Berksfile`. This informs
[Berkshelf](http://berkshelf.com) which cookbooks to download. The
relevant excerpt from the Berksfile is:

```ruby
cookbook "apt"
cookbook "yum"
cookbook "minitest-handler"
cookbook "bluepill_test", :path => "./test/cookbooks/bluepill_test"
```

Based on the
[Berksfile](https://github.com/opscode-cookbooks/bluepill/blob/master/Berksfile),
it will download apt, yum, and minitest-handler from the Chef
Community site. It will also use the
[bluepill_test](https://github.com/opscode-cookbooks/bluepill/tree/master/test/cookbooks/bluepill_test)
included in the bluepill cookbook. This is transparent to the user, as
I'll cover in a moment.

Test Kitchen's Vagrant driver plugin handles all the configuration of
Vagrant itself based on the entries in the `.kitchen.yml`. To get the
Berkshelf integration in the Vagrant boxes, we need to install the
vagrant-berkshelf plugin in Vagrant. Then, we automatically get
Berkshelf's Vagrant integration, meaning all the cookbooks defined in
the Berksfile are going to be available on the box we bring up.

Remember the test cookbook mentioned above? It's the next component.
The default `suite` in `.kitchen.yml` puts `bluepill_test` in the run
list. This particular recipe will include the `bluepill` default
recipe, then it sets up a test service using the `bluepill_service`
LWRP. This means that when the nodes brought up by Test Kitchen via
Vagrant converge, they'll have bluepill installed and set up, and then
a service running that we can test the final behavior. Since Chef will
exit with a non-zero return code if it encounters an exception, we
know that a successful run means everything is configured as defined
in the recipes, and we can run tests against the node.

The tests we'll run are written with the
[Minitest Chef Handler](https://github.com/calavera/minitest-chef-handler/).
These are defined in the test cookbook, `files/default/tests/minitest`
directory. The `minitest-handler` cookbook (also in the default suite
run list) will execute the
[default_test](https://github.com/opscode-cookbooks/bluepill/blob/master/test/cookbooks/bluepill_test/files/default/tests/minitest/default_test.rb)
tests.

In the next post, we'll look at how to run Test Kitchen, and what all
the output means.
