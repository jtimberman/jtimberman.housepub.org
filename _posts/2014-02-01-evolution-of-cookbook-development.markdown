---
layout: post
title: "Evolution of Cookbook Development"
date: 2014-02-01 12:48:49 -0700
comments: true
categories: [chef, cookbooks, development]
---

In this post, I will explore some development patterns that I've seen
(and done!) with Chef cookbooks, and then explain how we can evolve to
a new level of cookbook development. The examples here come from
Chef's new
[chef-splunk cookbook](http://www.getchef.com/blog/2014/01/28/chefs-splunk-cookbook-2/),
which is a refactored version of an old `splunk42` cookbook. While
there is a public `splunk` cookbook on the Chef community site, it
shares some of the issues that I saw with our old one, which are
partially subject matter of this post.

Anyway, on to the evolution!

# Sub-optimal patterns

These are the general patterns I'm going to address.

* Composing URLs from multiple local variables or attributes
* Large conditional logic branches like case statements in recipes
* Not using definitions when it is best to do so
* Knowledge of how node run lists are composed for search, or
  searching for "`role:some-server`"
* Repeated resources across multiple orthogonal recipes
* Plaintext secrets in attributes or data bag items

Cookbook development is a wide and varied topic, so there are many
other patterns to consider, but these are the ones most relevant to
the refactored cookbook.

## Composing URLs

It may seem like a good idea, to compose URL strings as attributes or
local variables in a recipe based on other attributes and local
variables. For example, in our `splunk42` cookbook we have this:

```ruby
splunk_root = "http://download.splunk.com/releases/"
splunk_version = "4.2.1"
splunk_build = "98164"
splunk_file = "splunkforwarder-#{splunk_version}-#{splunk_build}-linux-2.6-amd64.deb"
os = node['os'].gsub(/\d*/, '')
```

These get used in the following `remote_file` resource:

```ruby
remote_file "/opt/#{splunk_file}" do
  source "#{splunk_root}/#{splunk_version}/universalforwarder/#{os}/#{splunk_file}"
  action :create_if_missing
end
```

We reused the filename variable, and composed the URL to the file to
download. Then to upgrade, we can simply modify the `splunk_version`
and `splunk_build`, as Splunk uses a consistent naming theme for their
package URLs (thanks, Splunk!). The filename itself is built from a
case statement (more on that in the next section). We could further
make the version and build attributes, so users can update to newer
versions by simply changing the attribute.

So what is bad about this? Two things.

1. This is in the `splunk42::client` recipe, and repeated again in the
`splunk42::server` recipe with only minor differences (the package
name, splunk vs splunkforwarder).
2. Ruby has excellent libraries for manipulating URIs and paths as
strings, and it is easier to break up a string than compose a new one.

How can this be improved? First, we can set attributes for the full URL.
The actual code for that is below, but suffice to say, it will look
like this (note the version is different because the new cookbook installs
a new Splunk version).

```ruby
default['splunk']['forwarder']['url'] = 'http://download.splunk.com/releases/6.0.1/universalforwarder/linux/splunkforwarder-6.0.1-189883-linux-2.6-amd64.deb'
```

Second, we have
[helper libraries](https://github.com/opscode-cookbooks/chef-splunk/blob/master/libraries/helpers.rb)
distributed with the cookbook that break up the URI so we can return
just the package filename.

```ruby
def splunk_file(uri)
  require 'pathname'
  require 'uri'
  Pathname.new(URI.parse(uri).path).basename.to_s
end
```

The previous `remote_file` resource is rewritten like this:

```ruby
remote_file "/opt/#{splunk_file(node['splunk']['forwarder']['url'])}" do
  source node['splunk']['forwarder']['url']
  action :create_if_missing
end
```

As a bonus, the helper methods are available in other places like
other cookbooks and recipes, rather than the local scope of local
variables.

## Conditional Logic Branches

One of the wonderful things about Chef is that simple Ruby
conditionals can be used in recipes to selectively set values for
resource attributes, define resources that should be used, and other
decisions. One of the horrible things about Chef is that simple Ruby
conditionals can be used in recipes and often end up being far more
complicated than originally intended, especially when handling
multiple platforms and versions.

In the earlier example, we had a `splunk_file` local variable set in a
recipe. I mentioned it was built from a case statement, which looks
like this, in full:

```ruby
splunk_file = case node['platform_family']
  when "rhel"
    if node['kernel']['machine'] == "x86_64"
      splunk_file = "splunkforwarder-#{splunk_version}-#{splunk_build}-linux-2.6-x86_64.rpm"
    else
      splunk_file = "splunkforwarder-#{splunk_version}-#{splunk_build}.i386.rpm"
    end
  when "debian"
    if node['kernel']['machine'] == "x86_64"
      splunk_file = "splunkforwarder-#{splunk_version}-#{splunk_build}-linux-2.6-amd64.deb"
    else
      splunk_file = "splunkforwarder-#{splunk_version}-#{splunk_build}-linux-2.6-intel.deb"
    end
  when "omnios"
    splunk_file = "splunkforwarder-#{splunk_version}-#{splunk_build}-solaris-10-intel.pkg.Z"
  end
```

Splunk itself supports many platforms, and not all of them are covered
by this conditional, so it's easy to imagine how this can get further
out of control and make the recipe even harder to follow. Also
consider that this is just the `client` portion for the
`splunkforwarder` package, this same block is repeated in the `server`
recipe, for the `splunk` package.

So why is this bad? There are three reasons.

1. We have a large block of conditionals that sit in front of a user
reading a recipe.
2. This logic isn't reusable elsewhere, so it has to be duplicated in
the other recipe.
3. This is only the logic for the package filename, but we care about
the entire URL. I've also covered that composing URLs isn't delightful.

What is a better approach? Use the full URL as I mentioned before, and
set it as an attribute. We will still have the gnarly case statement,
but it will be tucked away in the `attributes/default.rb` file, and
hidden from anyone reading the recipe (which is the thing they
probably care most about reading).

```ruby
case node['platform_family']
when 'rhel'
  if node['kernel']['machine'] == 'x86_64'
    default['splunk']['forwarder']['url'] = 'http://download.splunk.com/releases/6.0.1/universalforwarder/linux/splunkforwarder-6.0.1-189883-linux-2.6-x86_64.rpm'
    default['splunk']['server']['url'] = 'http://download.splunk.com/releases/6.0.1/splunk/linux/splunk-6.0.1-189883-linux-2.6-x86_64.rpm'
  else
    default['splunk']['forwarder']['url'] = 'http://download.splunk.com/releases/6.0.1/universalforwarder/linux/splunkforwarder-6.0.1-189883.i386.rpm'
    default['splunk']['server']['url'] = 'http://download.splunk.com/releases/6.0.1/splunk/linux/splunk-6.0.1-189883.i386.rpm'
  end
when 'debian'
  # ...
```

The the complete case block can be viewed in the
[repository](https://github.com/opscode-cookbooks/chef-splunk/blob/master/attributes/default.rb#L46-L66).
Also, since this is an attribute, consumers of this cookbook can set
the URL to whatever they want, including a local HTTP server.

Another example of gnarly conditional logic looks like this, also from
the `splunk42::client` recipe.

```ruby
case node['platform_family']
when "rhel"
  rpm_package "/opt/#{splunk_file}" do
    source "/opt/#{splunk_file}"
  end
when "debian"
  dpkg_package "/opt/#{splunk_file}" do
    source "/opt/#{splunk_file}"
  end
when "omnios"
  # tl;dr, this was more lines than you want to read, and
  # will be covered in the next section.
end
```

Why is this bad? After all, we're selecting the proper package
resource to install from a local file on disk. The main issue is the
conditional creates different resources that can't be looked up in the
resource collection. Our recipe doesn't do this, but perhaps a wrapper
cookbook would. The consumer wrapping the cookbook has to duplicate
this logic in their own. Instead, it is better to select the provider
for a single `package` resource.

```ruby
package "/opt/#{splunk_file(node['splunk']['forwarder']['url'])}" do
  case node['platform_family']
  when 'rhel'
    provider Chef::Provider::Package::Rpm
  when 'debian'
    provider Chef::Provider::Package::Dpkg
  when 'omnios'
    provider Chef::Provider::Package::Solaris
  end
end
```

## Definitions Aren't Bad

Definitions are simply defined as recipe "macros." They are not
actually Chef Resources themselves, they just look like them, and
contain their own Chef resources. This has some disadvantages, such as
lack of metaparameters (like action), which has lead people to prefer
using the "Lightweight Resource/Provider" (LWRP) DSL instead. In fact,
some feel that definitions are bad, and that one should feel bad for
using them. I argue that they have their place. One advantage is their
relative simplicity.

In our `splunk42` cookbook, the client and server recipes duplicate a
lot of logic. As mentioned a lot of this is case statements for the
Splunk package file. They also repeat the same logic for choosing the
provider to install the package. I snipped the content from the `when
"omnios"` block, but it looks like this:

```ruby
cache_dir = Chef::Config[:file_cache_path]
splunk_pkg = splunk_file.gsub(/\.Z/, '')

execute "uncompress /opt/#{splunk_file}" do
  not_if { ::File.exists?(splunk_cmd) }
end

cookbook_file "#{cache_dir}/splunk-nocheck" do
  source "splunk-nocheck"
end

file "#{cache_dir}/splunkforwarder-response" do
  content "BASEDIR=/opt"
end

pkgopts = ["-a #{cache_dir}/splunk-nocheck",
           "-r #{cache_dir}/splunkforwarder-response"]

package "splunkforwarder" do
  source "/opt/#{splunk_pkg}"
  options pkgopts.join(' ')
  provider Chef::Provider::Package::Solaris
end
```

(Note: the logic for setting the provider is required since we're not using the default over-the-network package providers, and installing from a local file on the system.)

This isn't *too* bad on its own, but needs to be repeated again in the
server recipe if one wanted to run a Splunk server on OmniOS. The
actual differences between the client and server package installation
are the package name, `splunkforwarder` vs `splunk`. The earlier URL
attribute example established a `forwarder` and `server` attribute.
Using a definition, named `splunk_installer`, allows us to simplify
the package installation used by the client and server recipes to look
like this:

```ruby
splunk_installer 'splunkforwarder' do
  url node['splunk']['forwarder']['url']
end
splunk_installer 'splunk' do
  url node['splunk']['server']['url']
end
```

How is this better than an LWRP? Simply that there was less ceremony
in creating it. There is less cognitive load for a cookbook developer
to worry about. Definitions by their very nature of containing
resources are already idempotent and convergent with no additional
effort. They also automatically support why-run mode, whereas in an
LWRP that must be done by the developer. Finally, between resources in
the definition and the rest of the Chef run, notifications may be
sent.

Contrast this to an LWRP, we need `resources` and `providers`
directories, and the attributes of the resource need to be defined in
the resource. Then the action methods need to be written in the
provider. If we're using inline resources (which we are) we need to
declare those so any notifications work. Finally, we should ensure
that why-run works properly.

The actual definition is ~40 lines, and can be viewed in the cookbook
[repository](https://github.com/opscode-cookbooks/chef-splunk/blob/master/definitions/splunk_installer.rb).
I don't have a comparable LWRP for this, but suffice to say that it
would be longer and more complicated than the definition.

## Reasonability About Search

Search is one of the killer features of running a Chef Server.
Dynamically configuring load balancer configuration, or finding the
master database server is simple with a search. Because we often think
about the functionality a service provides based on the role it
serves, we end up doing searches that look like this:

```ruby
splunk_servers = search(:node, "role:splunk-server")
```

Then we do something with `splunk_servers`, like send it to a
template. What if someone doesn't like the [role name](http://bikeshed.io)?
Then we have to do something like this:

```ruby
splunk_servers = search(:node, "role:#{node['splunk']['server_role']}")
```

Then consumers of the cookbook can use whatever server role name they
want, and just update the attribute for it. But, the internet has said
that roles are bad, so we shouldn't use them (even though they
aren't ;)). So instead, we need something like one of these queries:

```ruby
splunk_servers = search(:node, "recipes:splunk42\:\:server")
#or
splunk_servers = search(:node, "#{node['splunk']['server_search_query']}")
```

The problem with the first is similar to the problem with the first
(`role:splunk-server`), we need knowledge about the run list in order
to search properly. The problem with the second is that we now have to
worry about constructing a query properly as a string that gets
interpolated correctly.

How can we improve this? I think it is more "Chef-like" to use an
attribute on the server's node object itself that informs queries the
intention that the node is in fact a Splunk server. In our
`chef-splunk` cookbook, we use `node['splunk']['is_server']`. The
query looks like this:

```ruby
splunk_servers = search(:node, "splunk_is_server:true")
```

This reads clearly, and the `is_server` attribute can be set in one of
15 places (for good or bad, but that's a different post).

## Repeating Resources, Composable Recipes

In the past, it was deemed okay to repeat resources across recipes
when those recipes were not included on the same node. For example,
client and server recipes that have similar resource requirements, but
may pass in separate data. Another example is in the
[haproxy](http://community.opscode.com/cookbooks/haproxy)) cookbook I
wrote where one recipe statically manages the configuration files, and
the other uses a Chef search to populate the configuration.

As I have mentioned above, a lot of code was duplicated between the
client and server recipes for our `splunk42` cookbook: user and group,
the case statements, package resources, execute statements (that
haven't been shared here), and the service resource. It is definitely
important to ensure that all the resources needed to converge a recipe
are defined, particularly when using notifications. That is why
sometimes a recipe will have a `service` resource with no actions like
this:

```ruby
service 'mything'
```

However Chef 11will generate a warning about
[cloned resources](http://tickets.opscode.com/browse/CHEF-3694) when
they are repeated in the same Chef run.

Why is this bad? Well, CHEF-3694 explains in more detail that
particular issue, of cloned resources. The other reason is that it
makes recipes harder to reuse when they have a larger scope than
absolutely necessary. How can we make this better? A solution to this
is to write small, composable recipes that contain resources that may
be optional for certain use cases. For example, we can put the service
resource in a recipe and include that:

```ruby
service 'splunk' do
  supports :status => true, :restart => true
  provider Chef::Provider::Service::Init
  action :start
end
```

Then when we need to make sure we have the `service` resource
available (e.g., for notifications):

```ruby
template "#{splunk_dir}/etc/system/local/outputs.conf" do
  source 'outputs.conf.erb'
  mode 0644
  variables :splunk_servers => splunk_servers
  notifies :restart, 'service[splunk]'
end
include_recipe 'chef-splunk::service'
```

Note that the service is included *after* the resource that notifies
it. This is a feature of the notification system, where the notified
resource can appear anywhere in the resource collection, and brings up
another excellent practice, which is to declare service resources
after other resources which affect their configuration. This prevents
a race condition where, if a bad config is deployed, the service would
attempt to start, fail, and cause the Chef run to exit before the
config file could correct the problem.

Making recipes composable in this way means that users can pick and
choose the ones they want. Our `chef-splunk` cookbook has a
prescriptive default recipe, but the client and server recipes mainly
include the others they need. If someone doesn't share our opinion on
this for their use case, they can pick and choose the ones they want.
Perhaps they have the `splunk` user and group created on systems
through some other means. They won't need the `chef-splunk::user`
recipe, and can write their own wrapper to handle that. Overall this
is good, though it does mean there are multiple places where a user
must look to follow a recipe.

## Plaintext Secrets

Managing secrets is one of the hardest problems to solve in system
administration and configuration management. In Chef, it is very easy
to simply set attributes, or use data bag items for authentication
credentials. Our old `splunk42` cookbook had this:

```ruby
splunk_password = node[:splunk][:auth].split(':')[1]
```

Where `node[:splunk][:auth]` was set in a role with the
`username:password`. This isn't particularly *bad* since our Chef
server runs on a private network and is secured with HTTPS and RSA
keys, but a defense in depth security posture has more controls in
place for secrets.

How can this be improved? At Chef, we started using
[Chef Vault](https://github.com/Nordstrom/chef-vault) to manage
secrets. I wrote a
[post about chef-vault](http://www.getchef.com/blog/2013/09/19/managing-secrets-with-chef-vault/)
a few months ago, so I won't dig too deep into the details here. The
current `chef-splunk` cookbook loads the authentication information
like this:

```ruby
splunk_auth_info = chef_vault_item(:vault, "splunk_#{node.chef_environment}")['auth']
user, pw = splunk_auth_info.split(':')

execute "#{splunk_cmd} edit user #{user} -password '#{pw}' -role admin -auth admin:changeme" do
  not_if { ::File.exists?("#{splunk_dir}/etc/.setup_#{user}_password") }
end

file "#{splunk_dir}/etc/.setup_#{user}_password" do
  content 'true\n'
  owner 'root'
  group 'root'
  mode 00600
end
```

The first line loads the authentication information from the
encrypted-with-chef-vault data bag item. Then we make a couple of
convenient local variables, and change the password from Splunk's
built-in default. Then, we control convergence of the execute by
writing a file that indicates that the password has been set.

The advantage of this over attributes or data bag items is that the
content is encrypted. The advantage over regular encrypted data bags
is that we don't need to distribute the secret key out to every
system, we can update the list of nodes that have access with a knife
command.

# Conclusion

Neither Chef (the company), nor I are here to tell anyone how to
write cookbooks. One of the benefits of Chef (the product) is its
flexibility, allowing users to write blocks of Ruby code in recipes
that quickly solve an immediate problem. That's how we got to where we
were with `splunk42`, and we certainly have other cookbooks that can
be refactored similarly. When it comes to sharing cookbooks with the
community, well-factored, easy to follow, understand, and use code is
preferred.

Many of the ideas here came from community members like Miah Johnson,
Noah Kantrowitz, Jamie Winsor, and Mike Fiedler. I owe them thanks for
challenging me over the years on a lot of the older patterns that I
held onto. Together we can build better automation through cookbooks,
and a strong collaborative community. I hope this information is
helpful to those goals.
