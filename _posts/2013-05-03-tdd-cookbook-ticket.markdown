---
layout: post
title: "TDD Cookbook Ticket"
date: 2013-05-03 14:36
comments: true
categories: [chef, cookbooks, development]
---

This post will briefly describe how I did a TDD update to Opscode's
[runit](http://ckbk.it/runit) to resolve an
[issue reported last night](https://tickets.opscode.com/browse/COOK-2867).

First, the issue manifests itself only on Debian systems. The runit
cookbook's `runit_service` provider will write an
[LSB init.d script](http://tickets.opscode.com/browse/COOK-1576) on
Debian, rather than symlinking to `/usr/bin/sv`. The problem raised in
the new ticket is that the template will follow the link and write to
`/usr/bin/sv`. This is bad, as it will end up in a forkbomb as
runsvdir attempts to restart sv on
[all the things](http://drupal.org/files/x-all-the-things-template.png).
Oops! Sorry about that. Let's get it fixed, and practice some TDD.

The runit cookbook includes support for test-kitchen, though I did
need to
[update it](https://github.com/opscode-cookbooks/runit/commit/8d2e0fcb9d6becf99c0d30694164e57d59fb667b)
for this effort. Part of this change was adding a box for Debian in
the `.kitchen.yml`. I set about resolving this with TDD in mind.

First, the runit cookbook includes a couple
["test" cookbooks](https://github.com/opscode-cookbooks/runit/tree/master/test/cookbooks)
to facilitate setting up the system with the `runit_service` resource
so the outcome can be tested to ensure the behavior is correct. I
started by adding a "failing test" in the `runit_test::service`
recipe, meaning a link resource, and a `runit_service` resource that
would overwrite `/usr/bin/sv`.

```ruby
link "/etc/init.d/cook-2867" do
  to "/usr/bin/sv"
end

runit_service "cook-2867" do
  default_logger true
end
```

Then I ran `kitchen test` on the Debian box. As expected, the link was
created, and then the runit service was configured. The service's
provider will wait until the service is up. Since we've destroyed the
sv binary, that will never happen, so I destroyed it. I manually
confirmed the behavior too, to make sure I wasn't seeing something
weird. Due to its very nature, this is *really* hard to test for
automatically, but it will happen consistently.

Next, I had to write the code to implement the fix for this bug.
Essentially, this means checking if the `/etc/init.d/cook-2867` file
is a symbolink link, and removing it.

```ruby
initfile = ::File.join( '/etc', 'init.d', new_resource.service_name)
::File.unlink(initfile) if ::File.symlink?(initfile)
```

Simple enough. Next I tested again by destroying the existing
environment and rerunning it from scratch. This takes some time, but
it verifies that everything is working properly. Here's the output on
Debian:

```
INFO: Processing link[/etc/init.d/cook-2867] action create (runit_test::service line 147)
INFO: link[/etc/init.d/cook-2867] created
INFO: Processing service[cook-2867] action nothing (dynamically defined)
INFO: Processing runit_service[cook-2867] action enable (runit_test::service line 151)
INFO: Processing directory[/etc/sv/cook-2867] action create (dynamically defined)
INFO: Processing template[/etc/sv/cook-2867/run] action create (dynamically defined)
INFO: Processing directory[/etc/sv/cook-2867/log] action create (dynamically defined)
INFO: Processing directory[/etc/sv/cook-2867/log/main] action create (dynamically defined)
INFO: Processing directory[/var/log/cook-2867] action create (dynamically defined)
INFO: Processing file[/etc/sv/cook-2867/log/run] action create (dynamically defined)
INFO: Processing template[/etc/init.d/cook-2867] action create (dynamically defined)
INFO: template[/etc/init.d/cook-2867] updated content
INFO: template[/etc/init.d/cook-2867] owner changed to 0
INFO: template[/etc/init.d/cook-2867] group changed to 0
INFO: template[/etc/init.d/cook-2867] mode changed to 755
INFO: runit_service[cook-2867] configured
INFO: Chef Run complete in 7.267132764 seconds
INFO: Running report handlers
```

I didn't feel I needed a specific test for this in minitest-chef,
because it wouldn't have finished converging (earlier behavior I saw
in the "failing" test).

If you're contributing to cookbooks, and they have support for
test-kitchen, it's awesome if you can open a bug report with a failing
test. In this case, it was fairly easy to reproduce the bug.
