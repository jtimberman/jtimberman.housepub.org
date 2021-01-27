---
layout: post
title: "Missing Transitive Dependencies"
date: 2015-03-25 12:54:08 -0600
comments: true
categories: [chef]
---

One of my home projects while I'm on vacation this week is rebuilding my server with Fedora 21 ([Server](https://getfedora.org/en/server/)). In order to do this, I needed to [add Fedora support](https://github.com/hw-cookbooks/runit/commit/c3d46bcf2c330e90aff8f1bb1e1701d57078cca8) to the [runit](https://supermarket.chef.io/cookbooks/runit) cookbook, since I use runit for a number of services on my system. That's really neither here nor there, as the topic of this blog post isn't specific to Fedora, nor runit.

The topic is actually about an issue with transitive dependencies and how Chef dependency resolution works.

Here's the scenario:

1. I run chef on my node
2. The runit cookbook is synchronized
3. An older version of the runit cookbook is downloaded
4. The new changes I expected were not made
5. WTFs ensue

So what happened?

The runit cookbook itself is updated to use Ian Meyer's Package Cloud repository - at least, the version I want to use is, which is on GitHub. When I submitted the PR for adding this repository for RHEL platforms, Ian had not yet added Fedora packages. That's okay because Fedora is not listed as supported in the cookbook. However, I wanted to use it, and figured folks in the community using Fedora Server might benefit too.

I digress. Ian pushed a Fedora package earlier today, so I added "fedora" to the various platform family conditionals, in the cookbook and opened the PR linked earlier. All was well in test kitchen. So I change my local repository's Berksfile:

```ruby
cookbook 'runit', github: 'hw-cookbooks/runit', ref: 'jtimberman/fedora-21'
```

Then a quick `berks update runit` and `berks upload runit`, and I was in business.

Or so I thought. Enter scenario listed above. The problem is, that when I did the `berks upload`, it only uploaded `runit`. However in the latest `runit` cookbook from that branch, it also adds a dependency on Computology's [packagecloud](https://supermarket.chef.io/cookbooks/packagecloud) cookbook, since it uses that to add the repository. When the `runit` cookbook is specified on the `berks upload` command, it doesn't upload the transitive dependencies when the location is a git URI. This appears to be by design.

What happens in Chef is that the server solves the graph, seeing that the node needs the runit cookbook. But the latest version of the runit cookbook depends on packagecloud, which hasn't yet been uploaded. So the dependency solver looks for the latest version of the runit cookbook that meets the constraint (none), and doesn't have the packagecloud cookbook. Thus, I end up with runit version 1.5.18 on my node, but it fails to converge because it doesn't have the changes required for Fedora, which are in 1.5.20.

The simple solution here is to upload the packagecloud cookbook. This can be done with `berks upload packagecloud`, as it does exist in the Berksfile.lock and has been cached in the berkshelf. Alternatively, `berks upload` will also upload the cookbook, as that operates on all cookbooks in  the Berksfile.lock.

I hope this helps anyone who's faced this issue with transitive dependencies when working on a cookbook "in development."
