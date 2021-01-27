---
layout: post
title: "Install Chef 10 Server on Ubuntu With Ruby 1.9"
date: 2012-11-17 11:27
comments: false
categories: [chef, ruby, debian]
---

I wanted to capture the minimal steps required to install a Chef
version 10 Server using RubyGems under Ruby 1.9 (1.9.3-p0) on
Ubuntu 12.04.

Similar steps are also
[available on the Chef wiki](http://wiki.opscode.com/display/chef/Installing+Chef+Server+using+Chef+Solo),
too.

This does not include any customization to the
installation and takes everything as default. This will all change
when
[Chef 11 is released](http://wiki.opscode.com/display/chef/Chef+11+Server+Preview).

These steps were performed on a default server install of Ubuntu
12.04.

Update apt sources.

```
sudo apt-get update
```

Install required packages.

```
sudo apt-get install ruby1.9.1-full build-essential wget ssl-cert curl
```

Update RubyGems.

```
REALLY_GEM_UPDATE_SYSTEM=yes sudo -E gem update --system
```

Install Chef.

```
sudo gem install chef --no-ri --no-rdoc
```

Configure chef-solo.

```
sudo mkdir -p /etc/chef
sudo vi /etc/chef/solo.rb
```

The config file should look like this:

```ruby
file_cache_path "/tmp/chef-solo"
cookbook_path "/tmp/chef-solo/cookbooks"
recipe_url "http://s3.amazonaws.com/chef-solo/bootstrap-latest.tar.gz"
```

Run chef-solo with the `chef-server::rubygems-install` recipe.

```
sudo chef-solo -o chef-server::rubygems-install
```

The WebUI is not enabled by default. This can be done by setting an
attribute, see the `chef-server` cookbook for more information.

Next, create a new API client that will be used to upload cookbooks.

```sh
sudo knife configure -i
```

Change the `chef_server_url` to use the FQDN or IP address of the Chef
Server. Change the path for the validation key. The `.chef/knife.rb`
file should look something like this:

```ruby
log_level                :info
log_location             STDOUT
node_name                'jtimberman'
client_key               '/home/jtimberman/.chef/jtimberman.pem'
validation_client_name   'chef-validator'
validation_key           '/home/jtimberman/.chef/validation.pem'
chef_server_url          'http://10.1.1.100:4000'
cache_type               'BasicFile'
cache_options( :path => '/home/jtimberman/.chef/checksums' )
```

Copy the validation key (`/etc/chef/validation.pem`) to the `.chef` directory.

```
sudo cp /etc/chef/validation.pem ~/.chef
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
