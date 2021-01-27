---
layout: post
title: "Quick Tip: Create a Provisioner Node"
date: 2015-02-09 20:50:12 -0700
comments: true
categories: [chef, quicktips, chefconf]
---

This quick tip is brought to you by my preparation for my ChefConf talk about using Chef Provisioning to build a Chef Server Cluster, which is based on my blog post about the same. In the blog post I used chef-zero as my Chef Server, but for the talk I'm using Hosted Chef.

In order for the Chef Provisioning recipe to work the provisioning node - the node that runs chef-client - needs to have the appropriate permissions to manage objects on the Chef Server. This is easy with chef-zero - there are no ACLs at all. However in Hosted Chef, like any regular Chef Server, the ACLs don't allow nodes' API clients to modify other nodes, or API clients.

Fortunately we can do all the work necessary using knife, with the [knife-acl](https://github.com/chef/knife-acl) plugin. In this quick tip, I'll create a group for provisioning nodes, and give that group the proper permissions for the Chef Provisioning recipe to create the machines' nodes and clients.

First of all, I'm using ChefDK, and it's my Ruby environment too, so install the gem:

```sh
chef gem install knife-acl
```

Next, use the `knife group` subcommand to create the new group. Groups are a number of users and/or API clients. By default, an organization on Hosted Chef will have `admins`, `billing-admins`, `clients`, and `users`. Let's create `provisioners` now.

```sh
knife group create provisioners
```

The Role-based access control (RBAC) system in the Chef Server allows us to assign read, create, update, grant, and delete permissions to various objects in the organization. Containers are a special holder of other types of objects, in this case we need to add permissions for the clients and nodes containers. This is what allows the Chef Provisioning recipe's `machine` resources to have their Chef objects created.

```sh
for i in read create update grant delete
do
  knife acl add containers clients $i group provisioners
done

for i in read create update grant delete
do
  knife acl add containers nodes $i group provisioners
done
```

Next, we need the API client that will be used by the Chef Provisioning node to authenticate with the Chef Server, and the node needs to be created as well. By default the client will automatically have permissions for the node object that has the same name.

```sh
knife client create -d chefconf-provisioner > ~/.chef/chefconf-provisioner.pem
knife node create -d chefconf-provisioner
```

Finally, we need to put the new API client into the provisioners group that was created earlier. First we need to get a mapping of the actors in the organization. Then we can add the client to the group.

```sh
knife actor map
knife group add actor provisioners chefconf-provisioner
```

The `knife actor map` command will generate a YAML file like this:

```yaml
---
:user_map:
  :users:
    jtimberman: 12345678901234567890123456780123
  :usags:
    12345678901234567890123456780123: jtimberman
:clients:
  chefconf-provisioner: chefconf-provisioner
  jtimberman-chefconf-validator: jtimberman-chefconf-validator
```

This maps users to their USAG and stores a list of clients. More information about this is in the [knife-acl README](https://github.com/chef/knife-acl/blob/master/README.md)

At this point, we have a node, with the private key in `~/.chef` that can be used with the Chef Server to use Chef Provisioning's `machine` resource. We can also perform additional tasks that require having a node object, such as create secrets as Chef Vault items:

```sh
knife vault create secrets dnsimple -M client -J data_bags/secrets/dnsimple.json -A jtimberman -S 'name:chefconf-provisioner'
```

The entire series of commands is below.

```sh
chef gem install knife-acl
knife group create provisioners

for i in read create update grant delete
do
  knife acl add containers clients $i group provisioners
done

for i in read create update grant delete
do
  knife acl add containers nodes $i group provisioners
done

knife client create -d chefconf-provisioner > ~/.chef/chefconf-provisioner.pem
knife node create -d chefconf-provisioner
knife actor map
knife group add actor provisioners chefconf-provisioner

knife vault create secrets dnsimple -M client -J data_bags/secrets/dnsimple.json -A jtimberman -S 'name:chefconf-provisioner'
```

Hopefully this helps you out with your use of Chef Provisioning, and a non-zero Chef server. If you have further questions, find me at ChefConf!
