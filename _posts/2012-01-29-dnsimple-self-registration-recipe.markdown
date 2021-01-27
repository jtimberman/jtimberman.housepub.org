---
layout: post
title: "DNSimple Self Registration Recipe"
date: 2012-01-29 19:00
comments: true
categories: [dns, saas, chef]
---

Earlier this month, I completed a
[switch to DNSimple](/blog/2012/01/02/switching-to-dnsimple/) for my
domain's DNS provider. I am still happy with the switch, and finally,
just now, got around to writing a recipe to have my systems
automatically register themselves in DNS.

In the post, I described automatically adding the DNS entries with the
[dnsimple cookbook](http://community.opscode.com/cookbooks/dnsimple).
I did this as a proof of concept, but I didn't add it to all my nodes,
instead using my existing data bag-driven solution.

That said, this post serves as a brief document on how you can mimic
this behavior with your own environment.

# Encrypted Data Bag

I put my DNSimple credentials in an
[encrypted data bag](http://wiki.opscode.com/display/chef/Encrypted+Data+Bags).
Since I have to decrypt and read the entire thing anyway, I also store
the relevant data there. I keep my encrypted data bags in a bag called
secrets. The structure looks like this:

```javascript
{
  "id": "dnsimple",
  "api_token": "DNSimple API Token Here",
  "domain": "your-domain.example.com",
  "username": "DNSimple username",
  "password": "DNSimple password"
}
```

Replace the values with your values. Encrypting the data is optional,
but requires that you create a secret key or key file. Read my
[previous post on the topic](/blog/2011/08/06/encrypted-data-bag-for-postfix-sasl-authentication/)
for more information.

```
% knife data bag from file secrets dnsimple.json
% knife data bag from file secrets dnsimple.json --secret-file /etc/chef/encrypted_data_bag_secret
```

The first command just uploads the data bag item, the second encrypts
it. Note that I manage my workstation with Chef, so I will use the
same secret file as the Chef default. The secret file needs to be
copied to each system that will need it.

# Recipe

The recipe looks like this:

```ruby
dnsimple = encrypted_data_bag_item("secrets", "dnsimple")
dnsimple_record "#{node['hostname']}.int.#{dnsimple['domain']}" do
  content node['ipaddress']
  type "A"
  action :create
  username dnsimple['username']
  password dnsimple['password']
  domain dnsimple['domain']
end
```

As
[mentioned in the previous blog post](/blog/2011/08/06/encrypted-data-bag-for-postfix-sasl-authentication/),
the `encrypted_data_bag_item` method is in a library. Either add that
library to your cookbook, or use the class directly.

```ruby
dnsimple = Chef::EncryptedDataBagItem.load("secrets", "dnsimple")
```

If you're not using an encrypted data bag, then the item can be
accessed with the normal method:

```ruby
dnsimple = data_bag_item("bagname", "dnsimple")
```

The real work happens in the `dnsimple_record` LWRP, which will add an
"A" record for the system running the recipe. Note that the actual
entry is going to use the `int` subdomain, and it will use the domain
stored in the data bag item. It also will use the default IP address
of the node, which means the IP for the interface with the default
route.
