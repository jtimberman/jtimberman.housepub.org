---
layout: post
title: "Quick Tip: Define Resources to Notifiy in LWRPs"
date: 2015-01-17 22:37:06 -0700
comments: true
categories: [chef, quicktips]
---

In this quick tip, I'll explain why you may need to create resources to notify in a provider, even if the resource exists in a recipe, when using `use_inline_resources` in Chef's [LWRP DSL](http://docs.chef.io/lwrp.html).

I'll use an example cookbook, `notif`, to illustrate. First, I've created `cookbooks/notif/resources/default.rb`, with the following content.

```ruby
actions :write
default_action :write
```

Then, I have written `cookbooks/notif/providers/default.rb` like this:

```ruby
use_inline_resources

action :write do
  log 'notifer' do
    notifies :create, 'file[notified]'
  end
end
```

Then the default recipe, where I'll use the resource automatically generated from the resource directory, `notif`.

```ruby
file 'notified' do
  content 'something'
  action :nothing
end

notif 'doer'
```

When I run Chef, I'll get an error like this:

```
 Recipe: notif::default
   * file[notified] action nothing (skipped due to action :nothing)
   * notif[doer] action write

     ================================================================================
     Error executing action `write` on resource 'notif[doer]'
     ================================================================================

     Chef::Exceptions::ResourceNotFound
     ----------------------------------
     resource log[notifer] is configured to notify resource file[notified] with action create, but file[notified] cannot be found in the resource collection. log[notifer] is defined in /tmp/kitchen/cookbooks/notif/providers/default.rb:4:in `block in class_from_file'

     Resource Declaration:
     ---------------------
     # In /tmp/kitchen/cookbooks/notif/recipes/default.rb

      12: notif 'doer'

     Compiled Resource:
 ------------------
     # Declared in /tmp/kitchen/cookbooks/notif/recipes/default.rb:12:in `from_file'

     notif("doer") do
       action :write
       retries 0
       retry_delay 2
       default_guard_interpreter :default
       declared_type :notif

       recipe_name "default"
     end
```

To fix this, I define the `file` resource in the provider:

```ruby
use_inline_resources

action :write do
  log 'notifer' do
    notifies :create, 'file[notified]'
  end

  file 'notified' do
    content new_resource.name
  end
end
```

Then when I run Chef, it will converge and notify the file resource to be configured.

```
Recipe: notif::default
  * file[notified] action nothing (skipped due to action :nothing)
  * notif[doer] action write
    * log[notifer] action write

    * file[notified] action create
      - create new file notified
      - update content in file notified from none to 935e8e
      --- notified       2015-01-18 05:47:49.186399317 +0000
      +++ ./.notified20150118-15795-om5fiw       2015-01-18 05:47:49.186399317 +0000
      @@ -1 +1,2 @@
      +doer
    * file[notified] action create (up to date)

Running handlers:
Running handlers complete
Chef Client finished, 3/4 resources updated in 1.298990565 seconds
```

## Why does this happen?

The reason for this is because `use_inline_resources` tells Chef that in this provider, we're using inline resources that will be added to their own run context, with their own resource collection. We don't have access to the resource collection from the recipe. Even though the `file[notified]` resource exists from the recipe, it doesn't actually get inherited in the provider's run context, raising the error we saw before.

We can turn off `use_inline_resources` by removing it, and the custom resource will be configured:

```ruby
action :write do
  log 'notifer' do
    notifies :create, 'file[notified]'
  end
end
```

Then run Chef:

```
Recipe: notif::default
  * file[notified] action nothing (skipped due to action :nothing)
  * notif[doer] action write (up to date)
  * log[notifer] action write
  * file[notified] action create
    - update content in file notified from 935e8e to 3fc9b6
    --- notified 2015-01-18 05:47:49.186399317 +0000
    +++ ./.notified20150118-16159-r18q7z 2015-01-18 05:50:57.832140405 +0000
    @@ -1,2 +1,2 @@
    -doer
    +something
```

Notice that the `file[notified]` resource wasn't updated at the start of the run, when it was encountered in the recipe, but it was when notified by the log resource in the provider action, changing the content.

## Use inline compile mode!

The `use_inline_resources` method in the lightweight provider DSL is strongly recommended. It makes it easier to send notifications from the custom resource itself to other resources in the recipe's resource collection. Read more about the [inline compile mode](http://docs.chef.io/lwrp.html#inline-compile-mode) in the Chef docs.

Also, define the resources that you need to notify when you're doing this in your provider's actions. A common example is within a provider that writes configuration for a service, and needs to tell that service to restart.
