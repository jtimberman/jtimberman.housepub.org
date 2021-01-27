---
layout: post
title: "Chef Audit Mode Introduction"
date: 2015-04-03 20:48:23 -0600
comments: true
categories: [chef]
---

I've started working with the [audit mode](https://docs.chef.io/analytics.html#audit-mode) feature introduced in Chef version 12.1.0. Audit mode allows users to write custom rules (controls) in Chef recipes using new DSL helpers. In his [ChefConf 2015 talk, "Compliance At Velocity,"](https://www.youtube.com/watch?v=g-_SW3adPwU) James Casey goes into more of the background and reasoning for this. For now, I wanted to share a few tips with users who may be experimenting with this feature on their own, too.

First, we need to update ChefDK to version 0.5.0, as that includes a version of test kitchen that [allows us to configure audit mode](https://github.com/test-kitchen/test-kitchen/pull/652) for chef-client.

```
curl -L https://chef.io/chef/install.sh | sudo bash -s -- -P chefdk
```

Next, create a new cookbook for the audit mode tests.

```
chef generate cookbook audit-test
cd audit-test
```

Then, modify the audit-test cookbook's `.kitchen.yml`:

```
---
driver:
  name: vagrant

provisioner:
  name: chef_zero
  client_rb:
    audit_mode: :audit_only

platforms:
  - name: ubuntu-12.04
  - name: centos-6.5

suites:
  - name: default
    run_list:
      - recipe[audit-test::default]
    attributes:
```

This is the generated `.kitchen.yml` with `client_rb` added to the provisioner config. Note that we must use the Ruby symbol syntax for the config value, `:audit_only`. The other valid values for `audit_mode` are `:enabled` and `:disabled`. This will be translated to an actual Ruby symbol in the generated config file (`/tmp/kitchen/client.rb`):

```ruby
audit_mode :audit_only
```

Next, let's write a control rule to test. Since we're using the default `.kitchen.yml`, which includes Ubuntu 12.04 and uses SSH to connect, we can assume that SSH is running, so port 22 is listening. The following control asserts this is true.

```ruby
control_group 'Blog Post Examples' do
  control 'SSH' do
    it 'should be listening on port 22' do
      expect(port(22)).to be_listening
    end
  end
end
```

Now run `kitchen converge ubuntu` to run Chef, but not tear down the VM aftward - we'll use it again for another example. Here's the audit phase output from the Chef run:

    % kitchen converge ubuntu
    Synchronizing Cookbooks:
      - audit-test
    Compiling Cookbooks...
    Starting audit phase

    Blog Post Examples
      SSH
        should be listening on port 22

    Finished in 0.10453 seconds (files took 0.37536 seconds to load)
    1 example, 0 failures
    Auditing complete

Cool! So we have asserted that the node complies with this control by default. But what does a failing control look like? Let's write one. Since we're working with SSH already, let's use the SSHd configuration. By default in the Vagrant base box we're using, root login is permitted, so this value is present:

```
PermitRootLogin yes
```

However, our security policy mandates that we set this to `no`, and we want to audit that.

```ruby
control_group 'Blog Post Examples' do
  control 'SSH' do
    it 'should be listening on port 22' do
      expect(port(22)).to be_listening
    end

    it 'disables root logins over ssh' do
      expect(file('/etc/ssh/sshd_config')).to contain('PermitRootLogin no')
    end
  end
end
```

Rerun `kitchen converge ubuntu` and we see the validation fails.

    Starting audit phase

    Blog Post Examples
      SSH
        should be listening on port 22
        disables root logins over ssh (FAILED - 1)

    Failures:

      1) Blog Post Examples SSH disables root logins over ssh
         Failure/Error: expect(file('/etc/ssh/sshd_config')).to contain('PermitRootLogin no')
    expected File "/etc/ssh/sshd_config" to contain "PermitRootLogin no"
         # /tmp/kitchen/cache/cookbooks/audit-test/recipes/default.rb:8:in `block (3 levels) in from_file'

    Finished in 0.13067 seconds (files took 0.32089 seconds to load)
    2 examples, 1 failure

    Failed examples:

    rspec  # Blog Post Examples SSH disables root logins over ssh
    [2015-04-04T03:29:41+00:00] ERROR: Audit phase failed with error message: Audit phase found failures - 1/2 controls failed

    Audit phase exception:
    Audit phase found failures - 1/2 controls failed

When we have a failure, we'll have contextual information about the failure, including the line number in the recipe where itwas found, and a stack trace (cut from the output here), in case more information is required for debugging. To fix the test, we can simply edit the config file to have the desired setting, or we can manage the file with Chef to set the value accordingly. Either way, after updating the file, the validation will pass, and all will be well.

We can put as many `control_group` and `control` blocks with the `it` validation rules as required to audit our policy. If we have many validations, it can be difficult to follow with all the output if there are failures. Chef's audit mode is based on [Serverspec](http://serverspec.org/), which is based on [RSpec](http://rspec.info/). We can use the `filter_tag` configuration feature of RSpec to only run the `control` blocks or `it` statements that we're interested in debugging. To do this, we need an `RSpec.configuration` block within the `control_group` - due to the way that audit mode is implemented, we can't do it outside of `control_group`.

For example, we could debug our root login configuration:

```ruby
control_group 'Blog Post Examples' do
  ::RSpec.configure do |c|
    c.filter_run focus: true
  end

  control 'SSH' do
    it 'should be listening on port 22' do
      expect(port(22)).to be_listening
    end

    it 'disables root logins over ssh', focus: true do
      expect(file('/etc/ssh/sshd_config')).to contain('PermitRootLogin no')
    end
  end
end
```

The key here is to pass the argument `focus: true` (or if you like hash rockets, `:focus => true`) on the `it` block. This could also be used on a `control` block:

```ruby
control 'SSH', focus: true do
  it 'does stuff...'
end
```

Then, when running `kitchen converge ubuntu`, we see only that validation:

    Starting audit phase

    Blog Post Examples
      SSH
        disables root logins over ssh (FAILED - 1)

    Failures:

      1) Blog Post Examples SSH disables root logins over ssh
         Failure/Error: expect(file('/etc/ssh/sshd_config')).to contain('PermitRootLogin no')

This example is simple enough that this isn't necessary, but if we were implementing audit mode checks for our entire security policy, that could be dozens or even hundreds of controls.

As of this writing, audit mode is still under development, and is considered an experimental feature. There will be further information, guides, and documentation about it coming to the Chef blog and docs site, and I'll have a post coming soon with something I'm working on, so stay tuned!
