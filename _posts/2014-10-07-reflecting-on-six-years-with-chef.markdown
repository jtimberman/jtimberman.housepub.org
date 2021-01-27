---
layout: post
title: "Reflecting on Six Years with Chef"
date: 2014-10-07 11:36:43 -0600
comments: true
categories: [chef, personal]
---

It actually started a bit over seven years ago. I saw the writing on the wall at IBM; my job was soon to be outsourced. I found an open position with the SANS institute, accepted an offer there, and was due to start work in a couple of weeks. Around the same time, my friends Adam Jacob and Nathan Haneysmith had started HJK Solutions. They invited me to join them then, but it wasn't the right time for me. Adam told me that at SANS I should at least use the [automation](https://puppetlabs.com) [tools](http://capistranorb.com/) and general [infrastructure management model](http://infrastructures.org) they planned to use. It turned out this was sage advice, for a number of reasons.

Around April, 2008, Adam told me he was working on "Chef," a Ruby based configuration management and system integration framework. I was excited about its potential, and a few months later on July 2, 2008, I started with HJK Solutions as a Linux system administration consultant. I got familiar with HJK's puppet-based stack, and ancillary Ruby tools like [iClassify](https://github.com/adamhjk/iclassify/wiki) while working on their customer infrastructures over the coming months. After Opscode was founded and we released Chef 0.5, my primary focus was porting HJK's puppet modules to [chef cookbooks](https://github.com/opscode/cookbooks).

## opscode/cookbooks

Adam had started the repository to give new users a place to begin using Chef with full working examples. I continued their development, and had the opportunity to solve hard problems of integration web application stacks with them. There were three important reasons for the repository to exist:

1. We have a body of knowledge as a tribe, and that can be codified.
1. Infrastructure as code is real, and it *can* be reusable.
1. The best way to learn Chef is to use Chef, and I had a goal to know Chef well enough to teach it to new users and companies.

The development of general purpose cookbooks ends up being harder than any of us really imagined, I think. Every platform is different, so not only did I have to learn Chef, I had to learn how different platforms behave for common (and uncommon) pieces of software in web operations stacks. Over the years of managing these cookbooks, I learned a lot about how the community was developing workflows for using Chef, and how they differed from our opinions. I learned also learned how to manage and contribute to open source projects at a rather large scale, and how to have compassion and empathy for new or frustrated users.

## Training and Services

In my time at CHEF, nee Opscode, I've had several job role changes. After several months of working on cookbooks, I added package and release management ([RIP, apt.opscode.com](http://lists.opscode.com/sympa/arc/chef/2014-11/msg00015.html)) to my repertoire. I then switched to technical evangelism and training. With mentorship from [John Willis](http://twitter.com/botchagalupe), I drafted the initial version of [Chef Fundamentals](https://www.getchef.com/blog/2010/07/13/open-source-chef-training-open-training/), and delivered our inaugural training class in Seattle.

I worked with the team John built to deliver training, speak at conferences, and work directly with customers to help make them successful with Chef. Eventually, John left the company to [build an awesome team](http://www.enstratius.com/news-events/press-releases/john-willis-announcement) at [Enstratius](http://www.enstratius.com/home). I took on the role of Director of the team, but eventually I discovered that the [management track](http://fractio.nl/2014/09/19/not-a-promotion-a-career-change/) was not the future of my career.

## Open Source and Community

I came back to working on the cookbooks, which I had previously [split into separate repositories](https://github.com/opscode-cookbooks). I was also working more directly in the community, doing public training classes only (our consulting team did private/onsite classes), participating in our [IRC channels](https://botbot.me/freenode/chef) and mailing [lists](http://lists.opscode.com). We had some organization churn, and I was moved around between four different managers, eventually reporting to [the inimitable Nathen Harvey](https://twitter.com/nathenharvey).

During one of our 1-1 discussions, he said, "You know, Joshua. You write a lot of cookbooks to automate infrastructure. But you haven't actually worked on any infrastructure in years. You should do something about that."

Around that time, there was a "senior system administrator" job posting on our very own careers site. I talked to our VP of Operations, and after a brief transition period, moved completely over to the ops team. I was able to bring with me the great practices from the community for developing cookbooks: testing with [chefspec](http://sethvargo.github.io/chefspec/) and [serverspec](http://serverspec.org/), code consistency with [rubocop]() and [foodcritic](http://foodcritic.io), and wrapping it all up with [test kitchen](http://kitchen.ci).

## The Future

I've had the privilege to do work that I love, which is automating hard problems using Chef. I've also had the privilege of being part of the web operations, infrastructure as code, devops, and Chef communities over the past six years. I've been to all four Chef summits, and all three ChefConfs. A thing I've noticed over the years is that many conversations keep coming up at the summits and ChefConf. Fresh on my mind because the last summit was so recent is the topic of cookbook reusability. See, during the time that I managed opscode/cookbooks, I eventually saw the point people in the community were making about these being real software repositories that need to be managed like other complex software projects. We split up the repository into individual repositories per cookbook. We started adding test coverage, and conforming to consistency via syntax and style lint checking. That didn't make cookboks more reusable, but it lowered the barrier of contribution, which in turn made them more reusable as more use cases could be covered. I got to be a part of that evolution, and it's been awesome.

While using Chef is one of my favorite technical things to do, I have come to the conclusion that based on my experience the best thing I can do is be a facilitator of stronger technical discipline with regard to using Chef. Primarily, this means improving how CHEF uses Chef to build Chef for our community and customers. We're already really good at using Chef to build Chef (the product), and run Hosted Chef (the service). However, awesome tools from the community such as Test Kitchen, Berkshelf, ChefSpec, and Foodcritic did not exist when we started out. Between new, awesome tools, and growing our organization with new awesome people, we need to improve on getting our team members up to speed on the process and workflow that helps us deliver higher quality products.

That is why I'm moving into a new role at CHEF. The sixth year marks as good a time as any to make a change, and I'm no stranger to that. I'm joining a team of quality advocacy led by Joseph Smith, as part of Jez Humble's "Office of Continuous Improvement and Velocity." In this new role, I will focus on improving our overall technical excellence so we can deliver better products to our community and customers, and so we can have awesome use cases and examples for managing Chef Server and its add-ons at scale.

My first short term goal in this new role is a workstation automation cookbook that can be used and extended by our internal teams for ensuring everyone has a consistent set of tools to work on the product. This will be made an open source project that the community can use and extend as well. We'll have more information about this as it becomes "a thing."

Next, I want to improve how we work on incidents. We've had sporadic blog posts about issues in Hosted Chef and Supermarket, and I'd like to see this get better.

I'm also interested in managing Chef Server 12 clusters, including all the add-ons. Recently I worked on the [chef-server-cluster](https://github.com/opscode-cookbooks/chef-server-cluster) cookbook, which will become the way CHEF deploys and manages Hosted Chef using the version 12 packages. Part of the earliest days of opscode/cookbooks, I maintained cookbooks to setup the open source Chef Server. Long time users may remember the "chef solo bootstrap" stack. Since then, CHEF has continued to iterate on that idea, and the "ctl" management commands largely use chef-solo under the hood. The new cookbook combines and wraps up manual processes and the "ctl" commands to enable us, our community, and our customers to build scalable Chef Server clusters using the omnibus packages. The cookbook uses chef-provisioning to do much of the heavy lifting.

It should be easy for organizations to be successful with Chef. That includes CHEF! My goal in my new position is to fuel the love of Chef internally and externally, whip up awesome, and stir up more delight. I also look forward to seeing what our community and customers do with Chef in their organizations.

## Thank you

I'd like to thank the friends and mentors I've had along this journey. You're all important, and we've shared some good times and code, and sometimes hugs. It's been amazing to see so many people become successful with Chef.

Above all, I'd like to thank Adam Jacob: for the opportunity to join in this ride, for inspiration to be a better system administrator and operations professional, for mentorship along the way, and for writing Chef in the first place. Cheers, my friend.

Here's to many more years of whipping up awesome!
