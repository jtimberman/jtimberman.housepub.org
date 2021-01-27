---
layout: post
title: "Quick Tip: Serverspec spec_helper in Test Kitchen"
date: 2014-12-31 16:01:23 -0700
comments: true
categories: [chef, quicktips]
---

Recently, I've started refactoring some [old](https://supermarket.chef.io/cookbooks/daemontools) [cookbooks](https://supermarket.chef.io/cookbooks/djbdns) [I wrote](https://supermarket.chef.io/cookbooks/ucspi-tcp) ages ago. I'm adding Serverspec coverage that can be run with `kitchen verify`. In this [quicktip](/blog/categories/quicktips), I'll describe how to create a `spec_helper` that can be used in all the specs. This is a convention used by [many](http://pivotallabs.com/spec-helper/) in the Ruby community to add configuration for RSpec.

For Chef, we can run integration tests after convergence using [Test Kitchen](http://kitchen.ci) using Serverspec. To do that, we need to require Serverspec, and then set its backend. In some cookbooks, the author/developer may have written `spec_helper` files in the various `test/integration/SUITE/serverspec/` directories, but this will use a single shared file for them all. Let's get started.

In the `.kitchen.yml`, add the `data_path` configuration directive in the provisioner.

```yaml
provisioner:
  name: chef_zero
  data_path: test/shared
```

Then, create the `test/shared` directory in the cookbook, and create the `spec_helper.rb` in it.

```sh
mkdir test/shared
$EDITOR test/shared/spec_helper.rb
```

Minimally, it should look like this:

```ruby
require 'serverspec'

set :backend, :exec
```

Then in your specs, for example `test/integration/default/serverspec/default_spec.rb`, require the `spec_helper`. On the instances under test, the file will be copied to `/tmp/kitchen/data/spec_helper.rb`.

```ruby
require_relative '../../../kitchen/data/spec_helper'
```

That's it, now when running `kitchen test`, or `kitchen verify` on a converged instance, the helper will be used.
