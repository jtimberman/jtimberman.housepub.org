---
layout: post
title: "Quick Tip: Chef 12 Homebrew User Mixin"
date: 2014-12-29 08:56:01 -0700
comments: true
categories: [chef, osx, workstation, quicktips]
---

OS X is an interesting operating system. It is a Unix, but is primarily used for workstations. As such, many system settings can, and should, be done as a non-privileged user. Some tasks, however, require administrative privileges. OS X uses `sudo` to escalate privileges. This is done by a nice GUI pop-up requesting the user password when done through another GUI element. However, one must use `sudo $COMMAND` when working at the Terminal.

The [Homebrew](http://brew.sh) package manager tries to do everything as a non-privileged user. The [installation script](https://raw.githubusercontent.com/Homebrew/install/master/install) will invoke some commands with `sudo` - namely to create and set the correct permissions on `/usr/local` (its default installation location). Once that is complete, `brew install` will not require privileged access for installing packages. In fact, the [Homebrew project recommends](https://github.com/Homebrew/homebrew/blob/b19d3afccef0ddc31820f1cb7d1a5316017e29df/share/doc/homebrew/FAQ.md#why-does-homebrew-say-sudo-is-bad-) never using `sudo` with the `brew` commands.

In Chef 12 the default provider for the `package` resource is `homebrew`. This originally came from the [homebrew cookbook](https://supermarket.chef.io/cookbooks/homebrew). In order to not use `sudo` when managing packages, there's a helper method (mixin) that attempts to determine what non-privileged user should run the `brew install` command. This is also [ported to Chef 12](https://github.com/opscode/chef/blob/4cb27331d81b394b816278e2bed6b3395b54b9c9/lib/chef/mixin/homebrew_user.rb). The method can also take an argument that specifies a particular user that should run the `brew` command.

When managing an OS X system with Chef, it is often easier to just run `chef-client` as `root`, rather than be around when `sudo` prompts for a password. This means that we need a way to execute other commands for managing OS X as a non-privileged user. We can reuse the mixin to do this. I'll demonstrate this using plain old Ruby with `pry`, which is installed in ChefDK, and I'll start it up with `sudo`. Then, I'll show a short recipe with `chef-apply`.

```
% which pry
/opt/chefdk/embedded/bin/pry
% sudo pry
```

Paste in the following Ruby code:

```ruby
require 'chef'
include Chef::Mixin::HomebrewUser
include Chef::Mixin::ShellOut

find_homebrew_uid #=> 501
```

The method `find_homebrew_uid` is the helper we want. As we can see, rather than returning `0` (for `root`), it returns `501`, which is the UID of the `jtimberman` user on my system. To prove that I'm executing in a process owned by `root`:

```ruby
Process.uid #=> 0
```

Or, I can shell out to the `whoami` command using Chef's `shell_out` method - which is the same method Chef would use to run `brew install`.

```ruby
shell_out('whoami').stdout #=> "root\n"
```

The `shell_out` method can take a `:user` attribute:

```ruby
shell_out('whoami', :user => find_homebrew_uid).stdout #=> "jtimberman\n"
```

So this can be used to install packages with `brew`, and is exactly what Chef 12 does.

```ruby
shell_out('brew install coreutils', :user => find_homebrew_uid)
```

Or, it can be used to run `defaults(1)` settings that require running as a specific user, rather than `root`

```ruby
# Turn off iPhoto face detection, please
shell_out('defaults write com.apple.iPhoto PKFaceDetectionEnabled 0', 
          :user => find_homebrew_uid)
```

```sh
# before...
jtimberman@localhost% defaults read com.apple.iPhoto PKFaceDetectionEnabled
1
# after!
jtimberman@localhost% defaults read com.apple.iPhoto PKFaceDetectionEnabled
0
```

Putting this together in a Chef recipe that gets run by `root`, we can disable face detection in iPhoto like this:

```ruby
Chef::Resource::Execute.send(:include, Chef::Mixin::HomebrewUser)

execute 'defaults write com.apple.iPhoto PKFaceDetectionEnabled 0' do
  user find_homebrew_uid
end
```

The first line makes the method available on all `execute` resources. To make the method available to all resources, use `Chef::Resource.send`, and to make it available across everything in all recipes, use `Chef::Recipe.send`. Otherwise we would get a `NoMethodError` exception.

The `execute` resource takes a `user` attribute, so we use the `find_homebrew_uid` method here to set the user. And we can observe the same results as above:

```
jtimberman@localhost% defaults write com.apple.iPhoto PKFaceDetectionEnabled 1
jtimberman@localhost% defaults read com.apple.iPhoto PKFaceDetectionEnabled
1
jtimberman@localhost% sudo chef-apply nofaces.rb
Recipe: (chef-apply cookbook)::(chef-apply recipe)
* execute[defaults write com.apple.iPhoto PKFaceDetectionEnabled 0] action run
- execute defaults write com.apple.iPhoto PKFaceDetectionEnabled 0
jtimberman@localhost% defaults read com.apple.iPhoto PKFaceDetectionEnabled
0
```

Those who have read the workstation management posts on this blog in the past may be aware that I have a [cookbook](https://supermarket.chef.io/cookbooks/mac_os_x) that can manage OS X "`defaults(1)`" settings. I [plan to make updates](https://github.com/chef-osx/mac_os_x/issues/21) to the resource in that cookbook that will leverage this method.
