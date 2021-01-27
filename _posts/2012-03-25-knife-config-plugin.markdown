---
layout: post
title: "Knife Config Plugin"
date: 2012-03-25 14:47
comments: true
categories: [chef, plugins]
---

I created a plugin for knife that will display a specified option from
Chef's configuration object, `Chef::Config`. It operates with the
scope of the automatically detected
[knife configuration file](http://wiki.opscode.com/display/chef/Knife#Knife-ConfiguringYourSystemForKnife),
or by passing the `-c` option with a configuration file.

## Installation

It's a gem.

```
% gem install knife-config
```

You can get the [source on GitHub](https://github.com/jtimberman/knife-config).

## Examples

Without any options:

```
% knife config
WARNING: No knife configuration file found
```

If an config option were specified, its default value would be printed.

With a single option, `chef_server_url`, but no configuration file available:

```
% knife config chef_server_url
WARNING: No knife configuration file found
chef_server_url:  http://localhost:4000
```

In a "Chef Repository" with a `.chef/knife.rb`:

```
% cd chef-repo
% knife config chef_server_url
chef_server_url:  https://api.opscode.com/organizations/MY_ORG
```

With a few more options:

```
% cd chef-repo
% knife config chef_server_url node_name validation_client_name
chef_server_url:  https://api.opscode.com/organizations/MY_ORG
node_name:        jtimberman
validation_client_name:  housepub-validator
```

With a different config file (`/etc/chef/client.rb` on the same
system):

```
% knife config node_name -c /etc/chef/client.rb
node_name:  imac.int.example.com
```

With `--all`, to show all the options:

```
% knife config --all
amqp_consumer_id:               default
amqp_host:                      0.0.0.0
amqp_pass:                      testing
amqp_port:                      5672
amqp_user:                      chef
amqp_vhost:                     /chef
authorized_openid_identifiers:
authorized_openid_providers:
cache_options:
  path:  /Users/jtimberman/.chef/checksums
cache_type:                     BasicFile
checksum_path:                  /var/chef/checksums
chef_server_url:                https://api.opscode.com/organizations/housepub
client_key:                     /Users/jtimberman/.chef/jtimberman.pem
client_registration_retries:    5
client_url:                     http://localhost:4042
[output truncated, trust me, it shows them all :)]
```

Show the "knife" configuration, which includes things like cloud
provider authentication. This doesn't currently support showing
sub-keys (like `knife[aws_access_key_id]`).

```
% knife config knife
knife:
  aws_access_key_id:      ZOMGAWS_KEY
  aws_secret_access_key:  ZOMGAWS_SECRET
```

## Contributing

If you'd like to contribute, that's awesome! Please create a fork,
branch, and a pull request.
[Reporting issues](https://github.com/jtimberman/knife-config/issues)
is helpful too.
