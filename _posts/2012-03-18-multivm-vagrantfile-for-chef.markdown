---
layout: post
title: "MultiVM Vagrantfile for Chef"
date: 2012-03-18 00:16
comments: true
categories: [virtualization, chef, development]
---

Most commonly, [Vagrant's](http://vagrantup.com)
Vagrantfile describes only a single VM. That's fine, but most
environments separate functionality to different servers (e.g.,
database and web app). For this reason, Vagrantfiles can be set up for
[multi-VM arrangements](http://docs.vagrantup.com/v2/multi-machine/index.html).

However, I'm going to describe a different use case for multiple VMs.

## Testing Multiple Distributions

As a cookbook developer for a variety of platforms and platform
versions, I have to ensure changes do not break existing functionality
across supported platforms - namely current releases of Ubuntu and
CentOS (and their parents, Debian and RHEL).

Enter Vagrant's Multi-VM Vagrantfile.

While I [posted recently](/blog/2012/02/15/testing-with-fission/)
about testing with VMware Fusion, I am using Vagrant more. Primarily
because Opscode uses Vagrant internally -
[Seth Chisamore](http://twitter.com/schisamo) built a multi-VM
Vagrantfile for bringing up our full stack. The specifics in his
Vagrantfile are tuned to that particular use case which is different
than mine. However, there are similar patterns that I adapted.

## Vagrantfile

The Vagrantfile configures four virtual machines:

* CentOS 5.7 and 6.2
* Ubuntu 10.04 and 11.10

I built my VMs with [veewee](https://github.com/jedi4ever/veewee)
templates that install Chef via the
[omnibus built chef-full package](http://opscode.com/chef/install).
That way I have a consistent installation that reflects what Opscode
will ship as the easiest and best supported way to install Chef.

Part of the magic of this configuration is that I'm going to reuse my
knife configuration. The Vagrantfile itself goes into my cookbook
testing Chef Repository.

```ruby

    require 'chef'
    require 'chef/config'
    require 'chef/knife'

    current_dir = File.dirname(__FILE__)
    Chef::Config.from_file(File.join(current_dir, '.chef', 'knife.rb'))
```

Next, I'm going to describe data about the virtual machines that I'm
going to run. This is a hash of named VMs, `centos5`, `lucid`, etc. I
assign their hostname, and give them a host only IP address. I also
set an initial run list, since Vagrant will (noisily) complain if the
run list is empty in a Chef provisioner.

*Note* I have a `base` role as a holder, the actual relevant things
 are in the `base_redhat` and `base_debian` roles. The details really
 don't matter, though.

```ruby
cookbook_testers = {
  :centos5 => {
    :hostname => "centos5-cookbook-test",
    :ipaddress => "172.16.13.5",
    :run_list => "role[base],role[base_redhat]"
  },
  :centos6 => {
    :hostname => "centos6-cookbook-test",
    :ipaddress => "172.16.13.6",
    :run_list => "role[base],role[base_redhat]"
  },
  :lucid => {
    :hostname => "lucid-cookbook-test",
    :ipaddress => "172.16.13.10",
    :run_list => "role[base],role[base_debian]"
  },
  :oneiric => {
    :hostname => "oneiric-cookbook-test",
    :ipaddress => "172.16.13.11",
    :run_list => "role[base],role[base_debian]"
  }
}
```

Next, I'm going to set up the `Vagrant::Config` object as "global"
configuration for all the VMs, and then iterate over each of the VMs
described above.

```ruby
Vagrant::Config.run do |global_config|
  cookbook_testers.each_pair do |name, options|
    global_config.vm.define name do |config|
      vm_name = "#{name}-cookbook-test"
      ipaddress = options[:ipaddress]
```

I disable the shared folder, since I'm going to use a Chef Server, and
my recipes will download what they need from remote repositories, not
my local system.

```ruby
config.vm.share_folder("v-root", "/vagrant", ".", :disabled => true)
```

Set up some basic configuration for the box. Modify this to suit your
environment. This section is on a per-VM basis. If particular tunables
were required, I'd create additional config in the `cookbook_testers`
hash above, and use those values here.

*Note* `name` will be a symbol, but only in some contexts of
 execution.

```ruby
config.vm.box = name.to_s
config.vm.boot_mode = :headless
config.vm.host_name = vm_name
config.vm.network :hostonly, ipaddress
```

Now I set up the Chef provisioner. Again, I'm using Chef with a Server
([Opscode Hosted Chef](http://opscode.com/hosted-chef), of course). I
use the `chef_server_url`, and `validation_client_name` settings from
my `knife.rb`.

The nodes' names will be `NAME-cookbook-test`, rather than their FQDN.
I use this with a rake task that nukes them all from orbit
consistently :).

```ruby
chef.chef_server_url = Chef::Config[:chef_server_url]
chef.validation_key_path = "#{current_dir}/.chef/#{Chef::Config[:validation_client_name]}.pem"
chef.validation_client_name = Chef::Config[:validation_client_name]
chef.node_name = vm_name
chef.provisioning_path = "/etc/chef"
chef.log_level = :info
```

The run list is going to be combined from the run lists defined from
the `cookbook_testers` hash above, and a shell environment variable,
`CHEF_RUN_LIST`, which is simply a comma-separated list of run list
items, similar to that used by `knife bootstrap`.

```ruby
run_list = []
run_list << ENV['CHEF_RUN_LIST'].split(",") if ENV.has_key?('CHEF_RUN_LIST')
chef.run_list = [options[:run_list].split(","), run_list].flatten
```

To use the Vagrantfile, I export the shell variable with the
role(s)/recipe(s) I am testing, then run `vagrant up`.

```
% export CHEF_RUN_LIST="recipe[apache2],recipe[apache2::mod_ssl]"
% vagrant up
```

Vagrant will bring up each VM one at a time, going through the full
cycle of provisioning. If there's an unhandled exception that causes
Chef to exit, then Vagrant also halts execution. If `vagrant up` is
rerun, then Vagrant continues to the *next* VM. To reprovision a
failed VM, it can be specified:

```
% vagrant provision centos5
```

Without the VM name, vagrant would reprovision all the VMs. Likewise,
`vagrant ssh NAME` can be used to open an SSH connection to the named
VM. This is useful to reprovision a VM that failed early, while
Vagrant is continuing on with the others.

## Full Vagrantfile

The Vagrantfile is split up in the earlier section, but you can see
the full thing below.

```ruby
require 'chef'
require 'chef/config'
require 'chef/knife'
current_dir = File.dirname(__FILE__)
Chef::Config.from_file(File.join(current_dir, '.chef', 'knife.rb'))
cookbook_testers = {
  :centos5 => {
    :hostname => "centos5-cookbook-test",
    :ipaddress => "172.16.13.5",
    :run_list => "role[base],role[base_redhat]"
  },
  :centos6 => {
    :hostname => "centos6-cookbook-test",
    :ipaddress => "172.16.13.6",
    :run_list => "role[base],role[base_redhat]"
  },
  :lucid => {
    :hostname => "lucid-cookbook-test",
    :ipaddress => "172.16.13.10",
    :run_list => "role[base],role[base_debian]"
  },
  :oneiric => {
    :hostname => "oneiric-cookbook-test",
    :ipaddress => "172.16.13.11",
    :run_list => "role[base],role[base_debian]"
  }
}
Vagrant::Config.run do |global_config|
  cookbook_testers.each_pair do |name, options|
    global_config.vm.define name do |config|
      vm_name = "#{name}-cookbook-test"
      ipaddress = options[:ipaddress]
      config.vm.share_folder("v-root", "/vagrant", ".", :disabled => true)
      config.vm.box = name.to_s
      config.vm.boot_mode = :headless
      config.vm.host_name = vm_name
      config.vm.network :hostonly, ipaddress
      config.vm.provision :chef_client do |chef|
        chef.chef_server_url = Chef::Config[:chef_server_url]
        chef.validation_key_path = "#{current_dir}/.chef/#{Chef::Config[:validation_client_name]}.pem"
        chef.validation_client_name = Chef::Config[:validation_client_name]
        chef.node_name = vm_name
        chef.provisioning_path = "/etc/chef"
        chef.log_level = :info
        run_list = []
        run_list << ENV['CHEF_RUN_LIST'].split(",") if ENV.has_key?('CHEF_RUN_LIST')
        chef.run_list = [options[:run_list].split(","), run_list].flatten
      end
    end
  end
end
```
