---
layout: post
title: "Quick Tip: Stubbing Library Helpers in ChefSpec"
date: 2015-05-30 08:48:47 -0600
comments: true
sidebar: collapse
categories: [quicktips, chef]
---

I'm currently updating my [vagrant cookbook](https://supermarket.chef.io/cookbooks/vagrant), and adding [ChefSpec](https://sethvargo.github.io/chefspec) coverage. Each of the different platform recipes results in slightly different resources to download the package file and install it. To support this, I have [helper methods](https://github.com/jtimberman/vagrant-cookbook/blob/master/libraries/helpers.rb) that calculate the download URI, the package name, and the SHA256 checksum based on the version of Vagrant (`node['vagrant']['version']`), and the platform (`node['os']`, `node['platform_family']`).

The outcomes I want to test are that for a given platform: the correct recipe is included, the correct file is downloaded, and the correct package resource installs the downloaded file. Those tests look like this (using Ubuntu/Debian example first):

```ruby
it 'includes the debian platform family recipe' do
  expect(chef_run).to include_recipe('vagrant::debian')
end

it 'downloads the package from the calculated URI' do
  expect(chef_run).to create_remote_file('/var/tmp/vagrant.deb').with(
    source: 'https://dl.bintray.com/mitchellh/vagrant/vagrant_1.88.88_x86_64.deb'
  )
end

it 'installs the downloaded package' do
  expect(chef_run).to install_dpkg_package('vagrant').with(
    source: '/var/tmp/vagrant.deb'
  )
end
```

I've set the version attribute in the `chef_run` block for the ChefSpec run to `1.88.88` to ensure that it doesn't use the value set in the attributes file, and then I can test this specifically in the source for the calculated URI, even if the attribute changes - hopefully Vagrant doesn't have a 1.88.88 version some day ;).

When I run `rspec spec`, I get exceptions, however.

```
1) vagrant::default debian includes the debian platform family recipe
   Failure/Error: end.converge(described_recipe)
   OpenURI::HTTPError:
     404 Not Found
```

This is because in the `attributes/default.rb`, the checksum is retrieved using the `vagrant_sha256sum` method:

```ruby
default['vagrant']['checksum'] = vagrant_sha256sum(node['vagrant']['version'])
```

The exception is happening when Chef loads the attributes file, and because the version is not valid. The solution here is to stub out the return value from `vagrant_sha256sum`. This can be anything at all really, because that specific attribute is to make sure we don't have to re-download the package to [compare its checksum](http://docs.chef.io/resource_remote_file.html#attributes) on later Chef runs. In this cookbook, the helper methods are not namespaced under a module, they're bare methods in `libraries.helpers.rb`. This [poses some](https://github.com/sethvargo/chefspec/issues/562) [challenges](https://github.com/sethvargo/chefspec/issues/549) [when trying](https://github.com/sethvargo/chefspec/issues/273) [to stub](https://github.com/sethvargo/chefspec/issues/138) [them](https://github.com/sethvargo/chefspec/issues/253) in ChefSpec. I won't rehash all the ways I attempted to get this to work, and instead focus on the final solution that got the tests passing:

```ruby
before(:each) do
  allow_any_instance_of(Chef::Node).to receive(:vagrant_sha256sum).and_return('')
end
```

When Chef loads cookbook attributes files, it is evaluating them in the context of a `Chef::Node`, so those library helper methods are sent to the `Chef::Node` object. Similarly, if this were inside a recipe, I would use `Chef::Recipe`, and if it were inside a resource (e.g., `package`), `Chef::Resource`.

I put this `before` block at the `describe 'vagrant::default'` level, not within any `context` blocks, so it will be done for each of the various per-platform tests. The results in my `debian` context are now:

```
% rspec spec --color -fd

vagrant::default
  debian
    includes the debian platform family recipe
    downloads the package from the calculated URI
    installs the downloaded package

Finished in 7.04 seconds (files took 7.93 seconds to load)
3 examples, 0 failures
```

Six issues have been filed against ChefSpec about this. Hopefully this can result in fewer inquiries.

Happy testing!
