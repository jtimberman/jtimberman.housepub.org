---
layout: post
title: "Quick Tip: ChefDK Provision"
date: 2015-05-15 16:18:03 -0600
comments: true
categories: [quicktips, chef]
---

Earlier today, [ChefDK 0.6.0 was released](https://www.chef.io/blog/2015/05/15/chefdk-0-6-0-released/). In this post, I will illustrate a fairly simple walkthrough using Amazon EC2, based on information in the [document](https://github.com/chef/chef-dk/blob/master/PROVISION_README.md#basic-example). This example will include Policyfile use, too. Let's get started.

First, install ChefDK 0.6.0. You can get it from the [Chef Downloads page](downloads.chef.io/chef-dk).

Next, setup [AWS credentials](https://jtimberman.housepub.org/blog/2013/10/19/managing-multiple-aws-account-credentials/). Ensure that they are exported to the current shell environment.

    AWS_ACCESS_KEY_ID=secrets
    AWS_SECRET_ACCESS_KEY=secrets
    AWS_DEFAULT_REGION=us-east-1
    AWS_SSH_KEY=your_ssh_key_name
    AWS_ACCESS_KEY=secrets
    AWS_SECRET_KEY=secrets

I'm going to use Hosted Chef as my Chef Server. I already have my user API key and configuration in `~/.chef`, and I'm going to rely on the automatic configuration detection in the `chef` command for that.

Generate a new repository using the `chef generate` command. Further commands run from this directory.

    chef generate repo chefdk-provision-demo
    cd chefdk-provision-demo

Generate a `provision` cookbook. This is the required name, and it must be in the current directory.

    chef generate cookbook provision

Edit the default recipe, `$EDITOR provision/recipes/default.rb`.

```ruby
context = ChefDK::ProvisioningData.context
with_driver 'aws::us-west-2'
options = {
  ssh_username: 'admin',
  use_private_ip_for_ssh: false,
  bootstrap_options: {
    key_name: 'jtimberman',
    image_id: 'ami-0d5b6c3d',
    instance_type: 'm3.medium',
  },
  convergence_options: context.convergence_options,
}
machine context.node_name do
  machine_options options
  action context.action
  converge true
end
```

To break this down, first we get the ChefDK provisioning context that will pass in options to `chef-provisioning`. Then we tell `chef-provisioning` to use the AWS driver, and in the `us-west-2` region. The options hash is used to setup the instance. We're using [Debian 8](https://wiki.debian.org/Cloud/AmazonEC2Image/Jessie), which uses the `admin` user to log in, an SSH key that exists in the AWS region, the actual AMI, and finally the instance type. Then, we're going to set the convergence options automatically from ChefDK. This is the important part that will ensure the node has the right run list.

Generate a `Policyfile.rb`.

    chef generate policyfile

And edit its content, `$EDITOR Policyfile.rb`.

```ruby
name            "chefdk-provision-demo"
default_source  :community
run_list        "recipe[libuuid-user]"
cookbook        "libuuid-user"
```

Here we're simply getting the [libuuid-user cookbook from Supermarket](https://supermarket.chef.io/cookbooks/libuuid) and applying the default recipe to the nodes that have this policy.

The next step is to install the Policyfile. This generates the Policyfile.lock.json, and downloads the cookbooks to the cache, `~/.chefdk/cache/cookbooks`. If this isn't run, `chef` will complain, with a reminder to run it.

    chef install

Finally, we can provision a testing system with this policy:

    chef provision testing --sync -n debian-libuuid

This will result in output similar to this:

    Uploading policy to policy group testing
    Uploaded libuuid-user 1.0.1 (c3220a49)
    Compiling Cookbooks...
    Recipe: provision::default
      * machine[debian-libuuid] action converge
        - Create debian-libuuid with AMI ami-0d5b6c3d in us-west-2
        - create node debian-libuuid at https://chef-api.example.com/organizations/testing
        -   add normal.tags = nil
        -   add normal.chef_provisioning = {"hash" => "of options"}
        - waiting for debian-libuuid (i-251f1cec on aws::us-west-2) to be connectable (transport up and running) ...
        - been waiting 60/120 -- sleeping 10 seconds for debian-libuuid (i-251f1cec on aws::us-west-2) to be connectable ...
        - debian-libuuid is now connectable        - generate private key (2048 bits)
        - create directory /etc/chef on debian-libuuid
        - write file /etc/chef/client.pem on debian-libuuid
        - create client debian-libuuid at clients
        -   add public_key = "-----BEGIN PUBLIC KEY-----\n..."
        - Add debian-libuuid to client read ACLs
        - Add debian-libuuid to client update ACLs
        - create directory /etc/chef/ohai/hints on debian-libuuid
        - write file /etc/chef/ohai/hints/ec2.json on debian-libuuid
        - write file /etc/chef/client.rb on debian-libuuid
        - write file /tmp/chef-install.sh on debian-libuuid
        - run 'bash -c ' bash /tmp/chef-install.sh'' on debian-libuuid
        [debian-libuuid] Starting Chef Client, version 12.3.0
                         [2015-05-15T22:42:43+00:00] WARN: Using experimental Policyfile feature
                         resolving cookbooks for run list: ["libuuid-user::default@1.0.1 (c3220a4)", "libuuid-user::verify@1.0.1 (c3220a4)"]
                         Synchronizing Cookbooks:
                           - libuuid-user
                         Compiling Cookbooks...
                         Converging 1 resources
                         Recipe: libuuid-user::default
                           * user[libuuid] action create
                           - create user libuuid

                         Running handlers:
                         Running handlers complete
                         Chef Client finished, 1/1 resources updated in 5.29666495 seconds
        - run 'chef-client -l auto' on debian-libuuid
