---
layout: post
title: "Quick Tip: Deleting Attributes"
date: 2014-12-24 10:00:40 -0700
comments: true
categories: [chef, quicktips]
---

I have a new goal for 2015, and that is to write at least one "Quick Tip" per week about Chef. I've added the category "[quicktips](/blog/categories/quicktips)" to make these easier to find.

In this quick tip, I want to talk about a new feature of Chef 12. The new feature is the ability to remove an attribute from all levels (default, normal, override) on a node so it doesn't get saved back to the Chef Server. This was brought up in [Chef RFC 23](https://github.com/opscode/chef-rfc/blob/master/rfc023-chef-12-attributes-changes.md#global-level-removals). The reason I don't want to save the attribute in question back to the server is that it is a secret that I have in a [Chef Vault item](https://github.com/Nordstrom/chef-vault).

I'm using [Datadog](https://www.datadoghq.com) for my home systems, and the wonderful folks at Datadog have a [cookbook](https://supermarket.chef.io/cookbooks/datadog) to set it up. The documentation requires that you set two attributes to authenticate, the API key, and the application key:

```ruby
node.default['datadog']['api_key'] = 'Secrets In Plain Text Attributes??'
node.default['datadog']['application_key'] = 'It is probably fine.'
```

I prefer to use chef-vault because [I think it's the best way](http://jtimberman.housepub.org/blog/2013/09/10/managing-secrets-with-chef-vault/) to manage shared secrets in Chef recipes. I still need to set the attributes for Datadog's recipe to work, however. In order to accomplish the goal here, I will use a custom cookbook, `housepub-datadog`. It has one recipe that looks like this:

```ruby
include_recipe 'chef-vault'

node.default['datadog']['api_key'] = chef_vault_item(:secrets, 'datadog')['data']['api_key']
node.default['datadog']['application_key'] = chef_vault_item(:secrets, 'datadog')['data']['chef']

include_recipe 'datadog::dd-agent'

ruby_block 'smash-datadog-auth-attributes' do
  block do
    node.rm('datadog', 'api_key')
    node.rm('datadog', 'application_key')
  end
  subscribes :create, 'template[/etc/dd-agent/datadog.conf]', :immediately
end
```

Let's take a closer look at the recipe.

```ruby
include_recipe 'chef-vault'
```

Here, the `chef-vault` recipe is included to ensure everything works, and I have a dependency on `chef-vault` in my cookbook's metadata. Next, we see the attributes set:

```ruby
node.default['datadog']['api_key'] = chef_vault_item(:secrets, 'datadog')['data']['api_key']
node.default['datadog']['application_key'] = chef_vault_item(:secrets, 'datadog')['data']['chef']
```

The `secrets/datadog` item looks like this in plaintext:

```json
{
  "id": "datadog",
  "data": {
    "api_key": "My datadog API key",
    "chef": "Application key for the 'chef' application"
  }
}
```

When Chef runs, it will load the vault-encrypted data bag item, and populate the attributes that will be used in the template. This template comes from the `datadog::dd-agent` recipe, which is included next. The template from that recipe looks like this:

```ruby
template '/etc/dd-agent/datadog.conf' do
  owner 'root'
  group 'root'
  mode 0644
  variables(
    :api_key => node['datadog']['api_key'],
    :dd_url => node['datadog']['url']
  )
end
```

Now, for the grand finale of this post, I delete the attributes that were set using a `ruby_block` resource. The timing here is important, because these attributes must be deleted after Chef has converged the template. This does get updated every run, because the ruby block is not convergent, and this is okay because the attributes are updated every run, too. I could write additional logic to make this convergent, but I'm okay with the behavior. The `subscribes` ensures that as soon as the template is written, the node object is updated to remove the attributes. Otherwise, this happens next after the `dd-agent` recipe.

```ruby
ruby_block 'smash-datadog-auth-attributes' do
  block do
    node.rm('datadog', 'api_key')
    node.rm('datadog', 'application_key')
  end
  subscribes :create, 'template[/etc/dd-agent/datadog.conf]', :immediately
end
```

Let's see this in action:

```
managed-node$ chef-client
...
Recipe: housepub-datadog::default
  * ruby_block[smash-datadog-auth-attributes] action run
    - execute the ruby block smash-datadog-auth-attributes
...
workstation% knife node show managed-node -a datadog.api_key -a datadog.application_key
managed-node:
  datadog.api_key:
  datadog.application_key:
```

**Bonus quick tip!** `knife node show` can take the `-a` option multiple times to display more attributes. I just discovered this in writing this post, and I don't know when it was added. For sure in Chef 12.0.3, so you should just upgrade anyway ;).

*Update* This [feature was added](https://github.com/opscode/chef/commit/4133160972a9972a9a062579504faa40eaa4c8db) by [Awesome Chef Ranjib Dey](https://twitter.com/RanjibDey).
