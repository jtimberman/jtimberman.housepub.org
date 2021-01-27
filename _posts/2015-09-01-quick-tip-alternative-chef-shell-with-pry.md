---
layout: post
title: 'Quick Tip: Alternative Chef Shell With Pry'
date: 2015-09-01 16:22 -0700
---
This quick tip brought to you by the letters “p,” “r,” and “y.”

You can start up a pry session in the context of a Chef recipe easily by using `chef-apply`. The pry gem is bundled with the Chef omnibus package, so it’s immediately available.

```ruby
chef-apply -e 'require "pry"; binding.pry'
```

This will result in a prompt like this:

```
2.1.6 (#<Chef::Recipe>):0 >
```

This is a recipe! Use `pry`’s `ls` command:

```
2.1.6 (#<Chef::Recipe>):0 > ls
Chef::EncryptedDataBagItem::CheckEncrypted#methods: encrypted?
Chef::DSL::DataQuery#methods: data_bag  data_bag_item  search
Chef::DSL::PlatformIntrospection#methods: platform?  platform_family?  value_for_platform  value_for_platform_family
Chef::DSL::IncludeRecipe#methods: include_recipe  load_recipe  require_recipe
*** SNIP ***

2.1.6 (#<Chef::Recipe>):0 > cd node
2.1.6 (node[localhost.example.com]):1 > ls
*** SNIP ***
2.1.6 (node[localhost.example.com]):1 > exit
2.1.6 (#<Chef::Recipe>):0 >
```

Write a resource:

```
2.1.6 (#<Chef::Recipe>):0 > file "/tmp/hello_world" do
2.1.6 (#<Chef::Recipe>):0 *   content "I'm in pry!"
2.1.6 (#<Chef::Recipe>):0 * end
```

This will return the `file[/tmp/hello_world]` resource. It doesn’t run Chef, but we can do that in one of two ways: exit pry, or send the create action to the resource.

```
2.1.6 (#<Chef::Recipe>):0 > resources("file[/tmp/hello_world]").run_action :create
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * file[/tmp/hello_world] action create
    - create new file /tmp/hello_world
    - update content in file /tmp/hello_world from none to 3a5417
    --- /tmp/hello_world  2015-09-01 08:35:18.000000000 -0600
    +++ /tmp/.hello_world20150901-42665-11bwurx   2015-09-01 08:35:18.000000000 -0600
    @@ -1 +1,2 @@
```

If we write multiple resources, we’d have to send that action to every one of them. Exiting pry will work, but then we are, of course, no longer in the pry session. This is not ideal, but hey, it’s not like we’re in `chef-shell`.

The `chef-apply` program runs in “solo mode.”

```
2.1.6 (#<Chef::Recipe>):0 > Chef::Config.solo
=> true
```

However, it may be useful to debug things through the Chef Server API. We will want to do two things. First, load a config file like `.chef/knife.rb`. We can verify the Chef Server we want is configured by checking `Chef::Config[:chef_server_url]`.

```
2.1.6 (#<Chef::Recipe>):0 > Chef::Config[:chef_server_url]
=> "https://localhost:443"
2.1.6 (#<Chef::Recipe>):0 > Chef::Config.from_file('.chef/knife.rb')
=> "client"
2.1.6 (#<Chef::Recipe>):0 > Chef::Config[:chef_server_url]
=> "https://api.opscode.com/organizations/joshtest"
```

Cool. Now let’s borrow `chef-shell`’s helper methods for interacting with the API.

```
require 'chef/shell/ext'
Chef::Shell::Extensions.extend_context_object(self)
```

And now we can use, for example, `api.get` or `nodes.all`.

```
2.1.6 (#<Chef::Recipe>):0 > api.get('/users')
=> [{"user"=>{"username"=>"joshtest"}}, {"user"=>{"username"=>"jtimberman"}}]
```

Of course, we can get a lot of this functionality by starting up `chef-shell` and loading pry, but I think this was more fun :–).

Learn more about [pry](https://pry.github.io/).