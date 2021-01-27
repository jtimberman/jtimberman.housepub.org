---
layout: post
title: "Chef Gem Compile Time Compatibility"
date: 2015-03-20 08:46:47 -0600
comments: true
categories: [chef, ruby]
---

TL;DR, if you're using Chef 11 and `chef-sugar`, upgrade `chef-sugar` to version 3.0.1. If you cannot upgrade, use the following in your `chef_gem` resources in your recipes:

```ruby
compile_time true if Chef::Resource::ChefGem.instance_methods(false).include?(:compile_time)
```

As you may be aware, [Chef 12.1.0](https://www.chef.io/blog/2015/03/03/chef-12-1-0-released/) introduces a change to the `chef_gem` resource that prints out warning messages like this:

    WARN: chef_gem[chef-vault] chef_gem compile_time installation is deprecated
    WARN: chef_gem[chef-vault] Please set `compile_time false` on the resource to use the new behavior.
    WARN: chef_gem[chef-vault] or set `compile_time true` on the resource if compile_time behavior is required.

These messages are just warnings, but if you're installing a lot of gems in your recipes, you may be annoyed by the output. As the warning indicates, you can set `compile_time true` property. This doesn't work on versions of Chef before 12.1, though:

    NoMethodError: undefined method `compile_time' for Chef::Resource::ChefGem

So, as a workaround, we can ask whether we respond to the `compile_time` method in the `chef_gem` resource:

```ruby
chef_gem 'chef-vault' do
  compile_time false if respond_to?(:compile_time)
end
```

This appears to get around the problem for most cases. However, if you're using `chef-sugar`, you'll note that until version 3.0.0, `chef-sugar` includes a `compile_time` DSL method that gets injected into `Chef::Resource` (and `Chef::Recipe`). This has been modified to `at_compile_time` in `chef-sugar` version 3.0.0 to work around Chef's introduction of a `compile_time` method in the `chef_gem` resource. The simple thing to do is make sure that your `chef-sugar` gem/cookbook are updated to v3.0.1. However if that isn't an option for some reason, you can use this conditional check:

```ruby
chef_gem 'chef-vault' do
  compile_time true if Chef::Resource::ChefGem.instance_methods(false).include?(:compile_time)
end
```

Hat tip to Anthony Scalisi, who added this in [a pull request for the aws cookbook](https://github.com/opscode-cookbooks/aws/pull/110). The `instance_methods` method comes from Ruby's [Module class](http://ruby-doc.org/core-2.2.1/Module.html#method-i-instance_methods). Per the documentation:

> Returns an array containing the names of the public and protected instance methods in the receiver. For a module, these are the public and protected methods; for a class, they are the instance (not singleton) methods. If the optional parameter is false, the methods of any ancestors are not included.

If we look at this in, e.g., `chef-shell` under Chef 12.1.0:

```
chef:recipe > Chef::Resource::ChefGem.instance_methods(false)
 => [:gem_binary, :compile_time, :after_created]
```

And in Chef 12.0.3 or 11.18.6:

```
chef > Chef::Resource::ChefGem.instance_methods(false)
 => [:gem_binary, :after_created]
```
