---
layout: post
title: "load_current_resource and chef-shell"
date: 2014-07-21 09:50:56 -0600
comments: true
categories: [chef]
---

This post will illustrate `load_current_resource` and a basic use of chef-shell.

The `chef-shell` is an `irb`-based [REPL (read-eval-print-loop)](http://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop). Everything I do is Ruby code, just like in Chef recipes or other cookbook components. I'm going to use a `package` resource example, so need privileged access (`sudo`).

```
% sudo chef-shell
```

The chef-shell program loads its configuration, determines what session type, and displays a banner. In this case, we're taking all the defaults, which means no special configuration, and a `standalone` session.

```
loading configuration: none (standalone session)
Session type: standalone
Loading...done.

This is the chef-shell.
 Chef Version: 11.14.0.rc.2
 http://www.opscode.com/chef
 http://docs.opscode.com/

run `help' for help, `exit' or ^D to quit.

Ohai2u jtimberman@jenkins.int.housepub.org!
```

To evaluate resources as we'd write them in a recipe, we need to switch to recipe mode.

```
chef > recipe_mode
```

I can do anything here that I can do in a recipe. I could paste in my own recipes. Here, I'm just going to add a `package` resource to manage the `vim` package. Note that this works like the "compile" phase of a `chef-client` run. The resource will be added to the `Chef::ResourceCollection` object. We'll look at this in a little more detail shortly.

```
chef:recipe > package "vim"
 => <package[vim] @name: "vim" @noop: nil @before: nil @params: {} @provider: nil @allowed_actions: [:nothing, :install, :upgrade, :remove, :purge, :reconfig] @action: :install @updated: false @updated_by_last_action: false @supports: {} @ignore_failure: false @retries: 0 @retry_delay: 2 @source_line: "(irb#1):1:in `irb_binding'" @guard_interpreter: :default @elapsed_time: 0 @sensitive: false @candidate_version: nil @options: nil @package_name: "vim" @resource_name: :package @response_file: nil @response_file_variables: {} @source: nil @version: nil @timeout: 900 @cookbook_name: nil @recipe_name: nil>
```

I'm done adding resources/writing code to test, so I'll initiate a Chef run with the `run_chef` method (this is a special method in `chef-shell`).

```
chef:recipe > run_chef
[2014-07-21T09:04:51-06:00] INFO: Processing package[vim] action install ((irb#1) line 1)
[2014-07-21T09:04:51-06:00] DEBUG: Chef::Version::Comparable does not know how to parse the platform version: jessie/sid
[2014-07-21T09:04:51-06:00] DEBUG: Chef::Version::Comparable does not know how to parse the platform version: jessie/sid
[2014-07-21T09:04:51-06:00] DEBUG: package[vim] checking package status for vim
vim:
  Installed: 2:7.4.335-1
  Candidate: 2:7.4.335-1
  Version table:
 *** 2:7.4.335-1 0
        500 http://ftp.us.debian.org/debian/ testing/main amd64 Packages
        100 /var/lib/dpkg/status
[2014-07-21T09:04:51-06:00] DEBUG: package[vim] current version is 2:7.4.335-1
[2014-07-21T09:04:51-06:00] DEBUG: package[vim] candidate version is 2:7.4.335-1
[2014-07-21T09:04:51-06:00] DEBUG: package[vim] is already installed - nothing to do
```

Let's take a look at what's happening. Note that we have INFO and DEBUG output. By default, `chef-shell` runs with `Chef::Log#level` set to `:debug`. In a normal Chef Client run with `:info` output, we see the first line, but not the others. I'll show each line, and then explain what Chef did.

```
[2014-07-21T09:04:51-06:00] INFO: Processing package[vim] action install ((irb#1) line 1)
```

There is a timestamp, the resource, `package[vim]`, the action `install` Chef will take, and the location in the recipe where this was encountered. I didn't specify one in the resource, that's the default action for package resources. The `irb#1 line 1` just means that it was the first line of the `irb` in recipe mode.

```
[2014-07-21T09:04:51-06:00] DEBUG: Chef::Version::Comparable does not know how to parse the platform version: jessie/sid
[2014-07-21T09:04:51-06:00] DEBUG: Chef::Version::Comparable does not know how to parse the platform version: jessie/sid
```

Chef chooses the default provider for each resource based on a mapping of platforms and their versions. It uses an internal class, `Chef::Version::Comparable` to do this. The system I'm using is a Debian "testing" system, which has the codename `jessie`, but it isn't a specific release number. Chef knows that for all `debian` platforms to use the `apt` package provider, and that'll do here.

```
[2014-07-21T09:04:51-06:00] DEBUG: package[vim] checking package status for vim
vim:
  Installed: 2:7.4.335-1
  Candidate: 2:7.4.335-1
  Version table:
 *** 2:7.4.335-1 0
        500 http://ftp.us.debian.org/debian/ testing/main amd64 Packages
        100 /var/lib/dpkg/status
[2014-07-21T09:04:51-06:00] DEBUG: package[vim] current version is 2:7.4.335-1
[2014-07-21T09:04:51-06:00] DEBUG: package[vim] candidate version is 2:7.4.335-1
```

This output is the `load_current_resource` method implemented in the [apt package provider](https://github.com/opscode/chef/blob/c507822d7b3fe0b34d02719dea0ab483ec292195/lib/chef/provider/package/apt.rb#L33-L38).

The `check_package_state` method does all the heavy lifting. It runs `apt-cache policy` and parses the output looking for the version number. If we used the `:update` action, and the installed version wasn't the same as the candidate version, Chef would install the candidate version.

Chef resources are convergent. They only get updated if they need to be. In this case, the `vim` package is installed already (our implicitly specified action), so we see the following line:

```
[2014-07-21T09:04:51-06:00] DEBUG: package[vim] is already installed - nothing to do
```

Nothing to do, Chef finishes its run.

## Modifying Existing Resources

We can manipulate the state of the resources in the resource collection. This isn't common in most recipes. It's required for certain kinds of development patterns like "wrapper" cookbooks. As an example, I'm going to modify the resource object so I don't have to log into the system again and run `apt-get remove vim`, to show the next section.

First, I'm going to create a local variable in the context of the recipe. This is just like any other variable in Ruby. For its value, I'm going to use the `#resources()` [method to look up](http://docs.opscode.com/chef/dsl_recipe.html#resources) a resource in the resource collection.

```
chef:recipe > local_package_variable = resources("package[vim]")
 => <package[vim] @name: "vim" @noop: nil @before: nil @params: {} @provider: nil @allowed_actions: [:nothing, :install, :upgrade, :remove, :purge, :reconfig] @action: :install @updated: false @updated_by_last_action: false @supports: {} @ignore_failure: false @retries: 0 @retry_delay: 2 @source_line: "(irb#1):1:in `irb_binding'" @guard_interpreter: :default @elapsed_time: 0.029617095 @sensitive: false @candidate_version: nil @options: nil @package_name: "vim" @resource_name: :package @response_file: nil @response_file_variables: {} @source: nil @version: nil @timeout: 900 @cookbook_name: nil @recipe_name: nil>
```

The return value is the package resource object:

```
chef:recipe > local_package_variable.class
 => Chef::Resource::Package
```

(`#class` is a method on the Ruby `Object` class that returns the class of the object)

To remove the `vim` package, I use the `#run_action` method (available to all `Chef::Resource` subclasses), specifying the `:remove` action as a symbol:

```
chef:recipe > local_package_variable.run_action(:remove)
[2014-07-21T09:11:50-06:00] INFO: Processing package[vim] action remove ((irb#1) line 1)
[2014-07-21T09:11:52-06:00] INFO: package[vim] removed
```

There is no additional debug to display. Chef will run `apt-get remove vim` to converge the resource with this action.

## Load Current Resource Redux

Now that the package has been removed from the system, what happens if we run Chef again? Well, Chef is convergent, and it takes idempotent actions on the system to ensure that the managed resources are in the desired state. That means it will install the `vim` package.

```
chef:recipe > run_chef
[2014-07-21T09:11:57-06:00] INFO: Processing package[vim] action install ((irb#1) line 1)
```

We'll see some familiar messages here about the version, then:

```
[2014-07-21T09:11:57-06:00] DEBUG: package[vim] checking package status for vim
vim:
  Installed: (none)
  Candidate: 2:7.4.335-1
  Version table:
     2:7.4.335-1 0
        500 http://ftp.us.debian.org/debian/ testing/main amd64 Packages
[2014-07-21T09:11:57-06:00] DEBUG: package[vim] current version is nil
[2014-07-21T09:11:57-06:00] DEBUG: package[vim] candidate version is 2:7.4.335-1
```

This is `load_current_resource` working as expected. As we can see from the `apt-cache policy` output, the package is not installed, and as the action to take is `:install`, Chef will do what we think:

```
Reading package lists...
Building dependency tree...
Reading state information...
The following packages were automatically installed and are no longer required:
  g++-4.8 geoclue geoclue-hostip geoclue-localnet geoclue-manual
  geoclue-nominatim gstreamer0.10-plugins-ugly libass4 libblas3gf libcolord1
  libcolorhug1 libgeoclue0 libgnustep-base1.22 libgnutls28 libminiupnpc8
  libpoppler44 libqmi-glib0 libstdc++-4.8-dev python3-ply xulrunner-29
Use 'apt-get autoremove' to remove them.
Suggested packages:
  vim-doc vim-scripts
The following NEW packages will be installed:
  vim
0 upgraded, 1 newly installed, 0 to remove and 28 not upgraded.
Need to get 0 B/905 kB of archives.
After this operation, 2,088 kB of additional disk space will be used.
Selecting previously unselected package vim.
(Reading database ... 220338 files and directories currently installed.)
Preparing to unpack .../vim_2%3a7.4.335-1_amd64.deb ...
Unpacking vim (2:7.4.335-1) ...
Setting up vim (2:7.4.335-1) ...
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vim (vim) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vimdiff (vimdiff) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/rvim (rvim) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/rview (rview) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vi (vi) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/view (view) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/ex (ex) in auto mode
```

This should be familiar to anyone that uses Debian/Ubuntu, it's standard `apt-get install` output. Of course, this is a development system so I have some cruft, but we'll ignore that ;).

If we run_chef again, we get the output we saw in the original example in this post:

```
[2014-07-21T09:50:06-06:00] DEBUG: package[vim] is already installed - nothing to do
```
