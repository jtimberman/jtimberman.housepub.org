---
layout: post
title: "Local-only Knife Configuration"
date: 2013-02-01 10:57
comments: true
categories: [chef, development]
---

In this post I want to discuss briefly an approach to setting up a
shared Knife configuration file for teams using the same Chef
Repository, while supporting customized configuration.

## Background

Most infrastructures managed by Chef have multiple people working on
them. Recently, several people in the Ruby community started working
together on migrating [RubyGems](http://rubygems.org) to
[Amazon EC2](https://github.com/rubygems/rubygems-aws).

The repository has a shared `.chef/knife.rb` which sets some local
paths where cookbooks and roles are located. In addition to this, I
wanted to test building the infrastructure using a Chef Server and my
own EC2 account.

## The Approach

At Opscode, we believe in leveraging internal DSLs. The
`.chef/knife.rb` (and Chef's `client.rb` or `solo.rb`, etc) is no
exception. While you can have a fairly simple configuration like this:

```ruby
node_name        "jtimberman"
client_key       "/home/jtimberman/.chef/jtimberman.pem"
chef_server_url  "https://api.opscode.com/organizations/my_organization"
cookbook_path    "cookbooks"
```

You can also have something like this:

```ruby
log_level     :info
log_location  STDOUT
node_name     ENV["NODE_NAME"] || "solo"
client_key    File.expand_path("../solo.pem", __FILE__)
cache_type    "BasicFile"
cache_options(path: File.expand_path("../checksums", __FILE__))
cookbook_path [ File.expand_path("../../chef/cookbooks", __FILE__) ]
if ::File.exist?(File.expand_path("../knife.local.rb", __FILE__))
  Chef::Config.from_file(File.expand_path("../knife.local.rb", __FILE__))
end
```

This is the `knife.rb` included in the
[RubyGems-AWS repo](https://github.com/rubygems/rubygems-aws).

The main part of interest here is the last three lines.

```ruby
if ::File.exist?(File.expand_path("../knife.local.rb", __FILE__))
  Chef::Config.from_file(File.expand_path("../knife.local.rb", __FILE__))
end
```

This says "if a file `knife.local.rb` exists, then load its
configuration. The `Chef::Config` class is what Chef uses for
configuration files, and the `#from_file` method will load the
specified file.

In this case, the content of my `knife.local.rb` is:

```ruby
node_name                "jtimberman"
client_key               "/Users/jtimberman/.chef/jtimberman.pem"
validation_client_name   "ORGNAME-validator"
validation_key           "/Users/jtimberman/.chef/ORGNAME-validator.pem"
chef_server_url          "https://api.opscode.com/organizations/ORGNAME"
cookbook_path [
  File.expand_path("../../chef/cookbooks", __FILE__),
  File.expand_path("../../chef/site-cookbooks", __FILE__)
]
knife[:aws_access_key_id]      = "Some access key I like"
knife[:aws_secret_access_key]  = "The matching secret access key"
```

Here I'm setting my Opscode Hosted Chef credentials and server. I also
set the `cookbook_path` to include the site-cookbooks directory (this
should probably go in the regular knife.rb). Finally, I set the knife
configuration options for my AWS EC2 account.

The configuration is parsed top-down, so the options here that overlap
the `knife.rb` will be used instead.

## In the Repository

In the repository, commit only the `.chef/knife.rb` and not the
`.chef/knife.local.rb`. I recommend adding the local file to the
.gitignore or VCS equivalent.

```
% echo .chef/knife.local.rb >> .gitignore
% git add .chef/knife.rb .gitignore
% git commit -m 'keep general knife.rb, local config is ignored'
```

## Conclusion

There are many approaches to solving the issue of having shared Knife
configuration for multiple people in a single repository. The real
benefit here is that the configuration file is Ruby, which provides a
lot of flexibility. Of course, when using someone else's configuration
examples, one should always read the code and understand it first :-).
