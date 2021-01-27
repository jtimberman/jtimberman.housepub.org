---
layout: post
title: "Starting ChefSpec Example"
date: 2013-05-09 21:10
comments: true
categories: [chef, development]
---

This is a quick post to introduce what I'm starting on testing with
[ChefSpec](http://acrmp.github.io/chefspec/). This is from Opscode's
Java cookbook. While the recipe tested is really trivial, it actually
has some nuances that require detailed testing.

First off, the whole thing is in
[this gist](https://gist.github.com/jtimberman/5552182). I'm going to
break it down into sections below. The file is `spec/default_spec.rb`
in the java cookbook (not committed/pushed yet).

The chefspec gem is where all the magic comes from. You can read about
ChefSpec on [its home page](http://acrmp.github.io/chefspec/). You'll
need to install the gem, and from there, run `rspec` to run the tests.

```ruby
require 'chefspec'
```

Next, we're going to describe the default recipe. We're using the
regular rspec "let" block to set up the runner to converge the recipe.
Then, because we know/assume that the openjdk recipe is the default,
we can say that this chef run should include the `java::openjdk` recipe.

```ruby
describe 'java::default' do
  let (:chef_run) { ChefSpec::ChefRunner.new.converge('java::default') }
  it 'should include the openjdk recipe by default' do
    chef_run.should include_recipe 'java::openjdk'
  end
```

Next, this cookbook supports Windows. However, we have to set up the
runner with the correct platform and version (this comes from
[fauxhai](https://github.com/customink/fauxhai)), and then set
attributes that are required for it to work.

```ruby
context 'windows' do
    let(:chef_run) do
      runner = ChefSpec::ChefRunner.new(
        'platform' => 'windows',
        'version' => '2008R2'
        )
      runner.node.set['java']['install_flavor'] = 'windows'
      runner.node.set['java']['windows']['url'] = 'http://example.com/windows-java.msi'
      runner.converge('java::default')
    end
    it 'should include the windows recipe' do
      chef_run.should include_recipe 'java::windows'
    end
  end
```

Next are the contexts for other install flavors. The default recipe
will include the right recipe based on the flavor, which is set by an
attribute. So we set up an rspec context for each recipe, then set the
install flavor attribute, and test that the right recipe was included.

```ruby
  context 'oracle' do
    let(:chef_run) do
      runner = ChefSpec::ChefRunner.new
      runner.node.set['java']['install_flavor'] = 'oracle'
      runner.converge('java::default')
    end
    it 'should include the oracle recipe' do
      chef_run.should include_recipe 'java::oracle'
    end
  end
  context 'oracle_i386' do
    let(:chef_run) do
      runner = ChefSpec::ChefRunner.new
      runner.node.set['java']['install_flavor'] = 'oracle_i386'
      runner.converge('java::default')
    end
    it 'should include the oracle_i386 recipe' do
      chef_run.should include_recipe 'java::oracle_i386'
    end
  end
```

Finally, a recent addition to this cookbook is support for
[IBM's Java](http://tickets.opscode.com/browse/COOK-2897). In addition
to setting the install flavor, we must set the URL where the IBM Java
package is (see the README in the commit linked in that ticket for
detail), and we can see that the `ibm` recipe is in fact included.

```ruby
  context 'ibm' do
    let(:chef_run) do
      runner = ChefSpec::ChefRunner.new
      runner.node.set['java']['install_flavor'] = 'ibm'
      runner.node.set['java']['ibm']['url'] = 'http://example.com/ibm-java.bin'
      runner.converge('java::default')
    end
    it 'should include the ibm recipe' do
      chef_run.should include_recipe 'java::ibm'
    end
  end
end
```

This is just the start of the testing for this cookbook. We'll need to
test each individual recipe. However as I've not written that code
yet, I don't have examples. Stay tuned!
