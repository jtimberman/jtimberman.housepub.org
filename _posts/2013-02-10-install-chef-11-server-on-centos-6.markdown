---
layout: post
title: "Install Chef 11 Server on CentOS 6"
date: 2013-02-10 07:57
comments: true
categories: [chef, rhel]
---

A few months ago, I posted briefly on
[how to install Chef 10 server on CentOS](https://jtimberman.housepub.org/blog/2012/11/17/install-chef-10-server-on-centos/).
This post revisits the process for Chef 11.

These steps were performed on a default CentOS 6.3 server install.

First, navigate to the
[Chef install page](http://www.opscode.com/chef/install) to get the
package download URL. Use the form on the "Chef Server" tab to select
the appropriate drop-down items for your system.

Install the package from the given URL.

```
rpm -Uvh https://opscode-omnitruck-release.s3.amazonaws.com/el/6/x86_64/chef-server-11.0.4-1.el6.x86_64.rpm
```

The package just puts the bits on disk (in `/opt/chef-server`). The
next step is to configure the Chef Server and start it.

```
% chef-server-ctl reconfigure
```

This runs the embedded `chef-solo` with the included cookbooks, and
sets up everything required - Erchef, RabbitMQ, PostgreSQL, etc.

Next, run the Opscode Pedant test suite. This will verify that
everything is working.

```
% chef-server-ctl test
```

Copy the default admin user's key and the validator key to your local
workstation system that you have Chef *client* installed on, and
create a new user for yourself with knife. You'll need version 11.2.0.
The key files on the Chef Server are readable only by root.

```
scp root@chef-server:/etc/chef-server/admin.pem .
scp root@chef-server:/etc/chef-server/chef-validator.pem .
```

Use `knife configure -i` to create an initial `~/.chef/knife.rb` and
new administrative API user for yourself. Use the FQDN of your newly
installed Chef Server, with HTTPS. The validation key needs to be
copied over from the Chef Server from
`/etc/chef-server/chef-validator.pem` to `~/.chef` to use it for
automatically bootstrapping nodes with `knife bootstrap`.

```
% knife configure -i
```

The `.chef/knife.rb` file should look something like this:

```ruby
log_level                :info
log_location             STDOUT
node_name                'jtimberman'
client_key               '/home/jtimberman/.chef/jtimberman.pem'
validation_client_name   'chef-validator'
validation_key           '/home/jtimberman/.chef/chef-validator.pem'
chef_server_url          'https://chef-server.example.com'
syntax_check_cache_path  '/home/jtimberman/.chef/syntax_check_cache'
```

Your Chef Server is now ready to use. Test connectivity as your user
with knife:

```
% knife client list
chef-validator
chef-webui
% knife user list
admin
jtimberman
```

In previous versions of Open Source Chef Server, users were API
clients. In Chef 11, users are separate entities on the Server.

The `chef-server-ctl` command is used on the Chef Server system for
management. It has built-in help (`-h`) that will display the various
sub-commands.
