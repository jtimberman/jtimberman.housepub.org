---
layout: post
title: "Switching to DNSimple"
date: 2012-01-02 13:03
comments: true
categories: articles
tags: 
- dns
- saas
---

<font size="-2">Reminder: this blog reflects my opinions and thoughts, and not
those of my employer, Chef Software, Inc.</font>

Like any good sysadmin, I have my own domain for email and other
purposes. I actually have had a couple, but this post is about my
current one. I originally set it up through Google Apps a couple years
ago, including registering the new domain with Google Apps' default
registrar, GoDaddy. For the most part, it was pretty simple and
painless to set up, including private registration via Domains by
Proxy. Yay!

However, as I automated more components of my home network with Chef,
I found the lack of API driven DNS rather frustrating. At last count,
I had 15 distinct networked devices, counting all the computers,
consoles, mobile devices, etc. This does not count the virtual
machines that I manage as a part of my daily job in doing Chef
cookbook development and testing, which should all have their own DNS
entries, since I'll access them over the network, and remembering IPs
is ridiculous.

# Internal DNS Server

I use [DJB's tinydns](http://cr.yp.to/djbdns.html) as my DNS server,
and it is automated with the
[Opscode Chef Cookbook](http://community.opscode.com/cookbooks/djbdns).
The first incarnation of this setup was a single monolithic template
file containing all the entries in my local network zone, delivered by
the `djbdns::tinydns-internal` recipe, like so:

```ruby
template "#{node[:djbdns][:tinydns_internal_dir]}/root/data" do
  source "tinydns-internal-data.erb"
  mode 0644
  notifies :run, resources("execute[build-tinydns-internal-data]")
end
```

This was great, and simple to manage for this single purpose setup. At
some point, I wrote cookbooks for
[unbound](http://community.opscode.com/cookbooks/unbound) and
[powerdns](http://community.opscode.com/cookbooks/pdns), as I was
evaluating whether one or the other might be easier to modularize. In
the process, I created a data bag of all my DNS entries that I could
step through in templates, so I could use the same data without caring
which software was going to consume it. In the end, I extended the
djbdns cookbook with a lightweight resource and provider, and added
usage to the `djbdns::tinydns-internal` recipe like this:

```ruby
dns = data_bag_item("djbdns", node[:djbdns][:domain].gsub(/\./, "_"))
#
file "#{node[:djbdns][:tinydns_internal_dir]}/root/data" do
  action :create
end
#
%w{ ns host alias }.each do |type|
  dns[type].each do |record|
    record.each do |fqdn,ip|
      #
      djbdns_rr "#{fqdn}.#{dns['domain']}" do
        cwd "#{node[:djbdns][:tinydns_internal_dir]}/root"
        ip ip
        type type
        action :add
        notifies :run, "execute[build-tinydns-internal-data]"
      end
      #
    end
  end
end
```

The data bag itself looks something like this:

```javascript
{
  "id": "int_example_com",
  "domain": "example.com",
  "ns": [
    { "ns: "127.0.0.1" }
  ],
  "alias": [
    { "gw":         "10.10.20.1" },
    { "smb":        "10.10.20.20" },
    { "files":      "10.10.20.20" },
    { "apt":        "10.10.20.120" },
    { "yum":        "10.10.20.120" }
  ],
  "host": [
    { "tavern":          "10.10.20.1" },
    { "cask":            "10.10.20.20" },
    { "cider":           "10.10.20.101" },
    { "merlot":          "10.10.20.103" },
    { "bourbon":         "10.10.20.104" },
    { "iphone":          "10.10.20.105" },
    { "doppelbock":      "10.10.20.106" },
    { "ipad":            "10.10.20.107" },
    { "wii":             "10.10.20.108" },
    { "xbox":            "10.10.20.109" },
    { "htpc":            "10.10.20.110" },
    { "virt1test":       "10.10.20.120" }
  ]
}
```

I am pretty pleased with this approach in that this data can be used
no matter what DNS resolver software I choose. While this is great for
the more static part of my network, as I mentioned I do have some
dynamic usage where I create new virtual machines, and I really want
them to register themselves in DNS automatically.

# Enter DNSimple

A few months ago I decided to switch over to DNSimple. The service was
compelling over the alternatives for a few reasons:

* Low cost ($3/mo) for my size of account.
* Very simple interface
* API for managing records (!)
* Reputation for great service

[Darrin Eden](https://github.com/dje)
wrote a [cookbook](http://community.opscode.com/cookbooks/dnsimple)
for automatically creating records through the API, too, so half my
work for automating with Chef was already done!

However, for various reasons I procrastinated the switchover. After
all, my existing solution worked ok for my purposes. Then after seeing
GoDaddy show up on the SOPA supporters list, and being one a
contributing author to the legislation(*), I decided that was the last
straw and I busted a move to finish the
[switch](http://blog.dnsimple.com/godaddy-sopa-and-you/).

Honestly, from the DNSimple side, it couldn't have been a better
experience. They have one-click services for managing DNS records for
a variety of common services - including Google Apps! It took some
time and hassle to move my domain out of GoDaddy, since their
interface is rather clunky, and I had to unprotect things through
Domains by Proxy to make the move, but after a couple hours everything
was fine. DNSimple has some
[tips for migrating](http://blog.dnsimple.com/things-to-know-about-transferring-a-domain/),
no matter who your current registrar is.

Now for the truly fun part!

# Automated DNS with Chef

Using the dnsimple cookbook is very straightforward. You create an "A"
record like this:

```ruby
dnsimple_record "cask.example.com" do
  name "cask"
  domain "example.com"
  content "10.10.20.20"
  type "A"
  action :create
  username node[:dnsimple][:username]
  password node[:dnsimple][:password]
  domain node[:dnsimple][:domain]
end
```

Yes, that is a private network IP, and yes it is going to be
registered in public DNS. It really doesn't matter that much in my
([and others'](http://serverfault.com/questions/4458/private-ip-address-in-public-dns))
opinion. Especially given that zomg, I just created a DNS entry with
Chef!

By default, the cookbook does assume, and use, node attributes for
storing the username and password. This can be set by a role, but it
means that all nodes will have the data. For my use, I decided to put
these values in a data bag, and because they are sensitive, I used an
[encrypted data bag](http://wiki.opscode.com/display/chef/Encrypted+Data+Bags).
I also wanted to reuse the data bag from my earlier DNS example, so I
wrote a recipe like this:

```ruby
dnsimple = encrypted_data_bag_item("secrets", "dnsimple")
dns = data_bag_item("dns", "int_example_com")
#
%w{ host alias }.each do |type|
  dns[type].each do |record|
    record_type = type =~ /^host$/ ? "A" : "CNAME"
    record.each do |hostname,ip|
      #
      dnsimple_record "#{hostname}.#{dns['domain']}" do
        name hostname
        content ip
        type record_type
        action :create
        username dnsimple['username']
        password dnsimple['password']
        domain dnsimple['domain']
      end
      #
    end
  end
#
end
```

I put that recipe in my "dnsserver" role, ran Chef, and boom, all my
DNS entries are updated on systems I don't have to manage, and all
around the world.

    % host cask.int.example.com 8.8.8.8
    Using domain server:
    Name: 8.8.8.8
    Address: 8.8.8.8#53
    Aliases:

    cask.int.example.com has address 10.10.20.20

What a wonderful redundant distributed key value store :-).

Note that the `encrypted_data_bag_item` method used in the recipe is in a cookbook
library. I wrote about that in an
[earlier blog post](https://jtimberman.housepub.org/blog/2011/08/06/encrypted-data-bag-for-postfix-sasl-authentication/).
It is pretty simple:

```ruby
class Chef
  class Recipe
    #
    def encrypted_data_bag_item(bag, item, secret_file =
        Chef::EncryptedDataBagItem::DEFAULT_SECRET_FILE)
      DataBag.validate_name!(bag.to_s)
      DataBagItem.validate_id!(item)
      secret = EncryptedDataBagItem.load_secret(secret_file)
      EncryptedDataBagItem.load(bag, item, secret)
    rescue Exception
      Log.error("Failed to load data bag item: #{bag.inspect} #{item.inspect}")
      raise
    end
    #
  end
end
```

## Drawbacks

While conveniently reusing the data I already had for my DNS entries,
the `dnsimple_record` provider does take about a second on my internet
connection for each entry to check if it's there. This makes the DNS
server's Chef client run take over a minute, where it used to be less
than 12 seconds. Many of the entries that are in the DNS data bag are
on systems that can add their own records, and will soon once I
refactor things a bit.

# Other Uses

A number of the entries in the DNS data bag are CNAMEs for services
that run on my home LAN server. I have internal services like
Netatalk/Time Machine, Samba, or Munin. I also have external services
like OpenVPN, SSH and Teamspeak. I'll add DNSimple records for each of
these so the recipes can automatically register new DNS entries,
eliminating a step in bringing up a new service.

# Open Source is Awesome

Chef is open source, of course, as is the
[dnsimple cookbook](https://github.com/heavywater/chef-dnsimple/blob/master/LICENSE)
that I'm using. While working with the cookbook as describe above over
the last couple days, I made some improvements and I sent a
[pull request](https://github.com/heavywater/chef-dnsimple/pull/1),
which has been merged and released. Thanks Darrin!

If you use Chef and are looking for an API driven way to manage DNS
entries for your systems, I strongly recommend DNSimple as a provider,
and the `dnsimple` cookbook to tie it all together.

<font size="-2">(*) This isn't a political-blog-soap-box, but this
really was the final motivator.</font>
