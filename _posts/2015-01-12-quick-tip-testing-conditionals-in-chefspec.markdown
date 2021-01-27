---
layout: post
title: "Quick Tip: Testing Conditionals in ChefSpec"
date: 2015-01-12 14:07:06 -0700
comments: true
categories: [chef, quicktips]
---

This tip is brought to you by the [homebrew](https://supermarket.chef.io/cookbooks/homebrew) cookbook.

[ChefSpec](http://sethvargo.github.io/chefspec/) is a great way to create tests for Chef recipes to catch regressions. Sometimes recipes end up having branching conditional logic that can have very different outcomes based on external factors - attributes, existing system state, or cross-platform support.

The homebrew cookbook only supports OS X, so we don't have cross-platform support to test there. However, its default recipe has four conditionals to test. You can read the entire [default_spec.rb](https://github.com/opscode-cookbooks/homebrew/blob/master/spec/recipes/default_spec.rb) for full context, I'm going to focus on just one aspect here:

* Installing homebrew should only happen if the `brew` binary does not exist.

This is a common use case in Chef recipes. The best way to go about converging your node to the desired state involves running some arbitrary command. In this case, it's the installation of Homebrew itself. Normally for installations we want to use an idempotent, convergent resource like `package`. However, since homebrew is to be our package management system, we have to do something else. As it turns out the homebrew project provides an installation script and that script will install a binary, `/usr/local/bin/brew`. We will assume that if Chef converged on a node after running the script, and the `brew` binary exists, then we don't need to attempt reinstallation. There's more robust ways to go about it (e.g., running `brew` gives some desired output), but this works for example purposes today.

From [the recipe](https://github.com/opscode-cookbooks/homebrew/blob/master/recipes/default.rb), here's the resource:

```ruby
execute 'install homebrew' do
  command homebrew_go
  user node['homebrew']['owner'] || homebrew_owner
  not_if { ::File.exist? '/usr/local/bin/brew' }
end
```

`command` is a script, called `homebrew_go`, which is a local variable set to a path in `Chef::Config[:file_cache_path]`. It is retrieved in the recipe with `remote_file`. The resource used to have `execute homebrew_go`, but when ChefSpec runs, it does so in a random temporary directory, which we cannot predict the name.

The astute observer will note that the `user` parameter has another conditional (designated by the `||`). That's actually the subject of another post. In this post, I'm concerned only with testing the guard, `not_if`.

The `not_if` is a Ruby block, which means the Ruby code is evaluated inline during the Chef run. How we go about testing that is the subject of this post.

First, we need to mock the return result of sending the `#exist?` method to the `File` class. There are two reasons. First, we want to control the conditional so we can write a test for each outcome. Second, someone running the test (like me) might have already installed homebrew on their local system (which I have), and so `/usr/local/bin/brew` will exist. To do this, in our context, we have a `before` block that stubs the return to false:

```ruby
before(:each) do
  allow_any_instance_of(Chef::Resource).to receive(:homebrew_owner).and_return('vagrant')
  allow_any_instance_of(Chef::Recipe).to receive(:homebrew_owner).and_return('vagrant')
  allow(File).to receive(:exist?).and_return(false)
  stub_command('which git').and_return(true)
end
```

There's some other mocked values here. I'll talk about the `vagrant` user for `homebrew_owner` in a moment, though again, that's the subject of another post.

The actual spec will test that the installation script will actually get executed when we run chef, and as the `vagrant` user.

```ruby
it 'runs homebrew installation as the default user' do
  expect(chef_run).to run_execute('install homebrew').with(
    :user => 'vagrant'
  )
end
```

When rspec runs, we see this is the case:

```
homebrew::default
  default user
    runs homebrew installation as the default user
```


If I didn't mock the user, it would be `jtimberman`, as that is the user that is running Chef via rspec/ChefSpec. The test would fail. If you're looking at the full file, there's some other details we're going to look at shortly. If I didn't mock the return for `File.exist?`, the execute wouldn't run at all.

To test what happens when `/usr/local/bin/brew` exists, I set up a new context in rspec, and create a new `before` block.

```ruby
context '/usr/local/bin/brew exists' do
  before(:each) do
    allow(File).to receive(:exist?).and_return(true)
    stub_command('which git').and_return(true)
  end

  it 'does not run homebrew installation' do
    expect(chef_run).to_not run_execute('install homebrew')
  end
end
```

We don't need the `vagrant` mocks earlier, but we do need to stub `File.exist?`. This test would pass on my system without it, but not on, e.g., a Linux system that doesn't have homebrew.

Then running rspec, we see:

```
homebrew::default
  /usr/local/bin/brew exists
    does not run homebrew installation
  default user
    runs homebrew installation as the default user
```

In a coming post, I will walk through the conditionals related to the `homebrew_owner`.
