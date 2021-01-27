---
layout: post
title: "Install Chef 10 Server on CentOS"
date: 2012-11-17 19:49
comments: false
categories: [chef, rhel]
---

In addition to capturing the minimal steps required to [install a Chef
10 Server on Ubuntu](/blog/2012/11/17/install-chef-10-server-on-ubuntu-with-ruby-1-dot-9/), I also wanted to capture the steps for CentOS.
Unfortunately, Ruby 1.9 isn't available in the default distribution
repositories, and almost all third party repositories are quite out of
date. As such, I'll use the Ruby 1.8.7 that comes with CentOS, despite
being an EOL version of Ruby.

Similar steps are also
[available on the Chef wiki](http://wiki.opscode.com/display/chef/Installing+Chef+Server+using+Chef+Solo),
too.

This does not include any customization to the installation and takes
everything as default. This will all change when
[Chef 11 is released](http://wiki.opscode.com/display/chef/Chef+11+Server+Preview).

These steps were performed on a default server install of CentOS 6.3.

Install required packages.

```
yum install ruby ruby-devel rubygems wget gcc gcc-c++ kernel-devel make
```

Update RubyGems.

```
gem update --system
```

Install Chef.

```
gem install chef --no-ri --no-rdoc
```

Configure chef-solo.

```
mkdir -p /etc/chef
vi /etc/chef/solo.rb
```

The config file should look like this:

```ruby
file_cache_path "/tmp/chef-solo"
cookbook_path "/tmp/chef-solo/cookbooks"
recipe_url "http://s3.amazonaws.com/chef-solo/bootstrap-latest.tar.gz"
```

Run chef-solo with the `chef-server::rubygems-install` recipe.

```
chef-solo -o chef-server::rubygems-install
```

The WebUI is not enabled by default. This can be done by setting an
attribute, see the `chef-server` cookbook for more information.

Next, create a new API client that will be used to upload cookbooks.

```sh
knife configure -i
```

Change the `chef_server_url` to use the FQDN or IP address of the Chef
Server. Change the path for the validation key. The `.chef/knife.rb`
file should look something like this:

```ruby
log_level                :info
log_location             STDOUT
node_name                'jtimberman'
client_key               '/root/.chef/jtimberman.pem'
validation_client_name   'chef-validator'
validation_key           '/root/.chef/validation.pem'
chef_server_url          'http://10.1.1.100:4000'
cache_type               'BasicFile'
cache_options( :path => '/root/.chef/checksums' )
```

Copy the validation key (`/etc/chef/validation.pem`) to the `.chef` directory.

```
cp /etc/chef/validation.pem ~/.chef
```

Verify that the API client and configuration file work.

```
knife client list
  chef-validator
  chef-webui
  jtimberman
```

The newly created API client should be listed. Mine is `jtimberman`.
Now the Chef Server is ready to use.
