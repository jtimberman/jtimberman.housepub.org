---
layout: post
title: "Chef Repository Berkshelf Conversion"
date: 2012-11-19 22:10
comments: true
categories: [chef, personal, development]
---

I've been managing my personal systems with Chef since Chef was
created, though I didn't always use the same chef-repo for them. For
about two years though, I've used pretty much the same repository,
which has grown and accumulated cruft over time. Fortunately since
it's only me working on it, and I only have a few systems, it is
really easy to make drastic changes.

I have a number of to do items that I've put off, so this weekend I
decided to spend some time cleaning house, and convert the repository
to have cookbooks managed by [Berkshelf](http://berkshelf.com/).

# Rationale

There are other cookbook management tools, including the built in
"`knife cookbook site install`",
[librarian-chef](https://github.com/applicationsonline/librarian), and
[whisk](https://github.com/kisoku/whisk). I have used the knife
command as long as it has existed, and it worked well for awhile. The
buzz in the community since the
[Chef Summit](http://wiki.opscode.com/display/chef/Opscode+Community+Summit+2)
has been around "library" vs "application" cookbooks, especially in
conjunction with Berkshelf so I thought I'd give it a go.

# Before Berkshelf

Before I started on this migration, here are some numbers about
cookbooks in my chef-repo.

- 113 total cookbooks
- 33 "chef-vendor" branches (`knife cookbook site install` creates a
  branch for each cookbook)
- 50 "cookbook site" tags

Overall, I had about a half dozen cookbooks that I actually modified
from their "upstream" versions on the community site. Most of those
customizations were adding munin plugins, changing a couple minor
settings in a template, or long term workarounds that are actually
fixed in the current released versions.

# The Conversion

The conversion was fairly straight-forward. It required some preparation:

- Determine the cookbooks that would be managed by Berkshelf.
- Refactor customizations into "application" cookbooks or otherwise.
- Remove all those cookbooks, and the completely unused cookbooks.

## Cookbooks in Berkshelf

Determining the cookbooks that would be managed by Berkshelf was
simple. I started with all the cookbooks that had been installed via
`knife cookbook site install`. Since the command creates a branch for
each one, I had a nice list already. I did review that for cookbooks I
know I wasn't using anymore, or didn't plan to use for long, to
simplify matters.

```
git branch | grep 'chef-vendor' | awk -F- '{print $3}'
```

I also looked at the cookbooks that are applied to node's expanded run
lists. This `knife exec` one-liner will return such a list.

```
knife exec -E "nodes.find('recipes:*').map {|n| n[:recipes]}.flatten.map {|r| r.gsub(/::.*/, '')}.sort.uniq"
```

## Refactoring Customization

My repository has a fair amount of customization to the cookbooks from
the community site. Rather than go through all the changes, I'll
summarize with the more interesting parts.

First, I use Samba for filesharing from an Ubuntu server. I originally
changed the `samba::server` recipe so the services used upstart as the
provider and set a `start_command` on Ubuntu, which looked like this
(`s` is smbd or nmbd):

```ruby
service s do
  pattern "smbd|nmbd" if node["platform"] =~ /^arch$/
  provider Chef::Provider::Service::Upstart if platform?("ubuntu")
  start_command "/usr/bin/service #{s} start" if platform?("ubuntu")
  action [:enable, :start]
end
```

The upstream cookbook doesn't have this change, so I added an
"application" cookbook, `housepub-samba`, which has this as the
default recipe:

```ruby
["smbd", "nmbd"].each do |s|
  srv = resource("service[#{s}]")
  srv.provider Chef::Provider::Service::Upstart
  srv.start_command "/usr/bin/service #{s} start"
end if platform?("ubuntu")
```

For each of the Samba services, we look up the resource in the
resource collection, then change the provider to upstart, and set the
`start_command` to use upstart's service command.

Next, I use OpenVPN. I also want to modify the template used for the
`/etc/openvpn/server.conf` and `/etc/openvpn/server.up.sh` resources.
Again, I create an "application" cookbook, `housepub-openvpn`, and the
default recipe looks like this:

```ruby
resources("template[/etc/openvpn/server.conf]").cookbook "housepub-openvpn"
resources("template[/etc/openvpn/server.up.sh]").cookbook "housepub-openvpn"
```

This is a shorter form of what was done for Samba's services above.
The `#resources` method does the lookup and returns the resource, and
any of the resource parameter attributes can be used as a method, so I
send the `cookbook` method to both template resources, setting this
cookbook, `housepub-openvpn` as the cookbook that contains the
template to use. Then, I copy my customized templates into
`cookbooks/housepub-openvpn/templates/default`, and Chef will do the
right thing.

Other cookbook changes I made were:

- Change the data bag name used in `djbdns::internal_server`, which I
  changed back so I could use the upstream recipe.
- Add munin plugins to various cookbooks. As I'm planning to move
  things to Graphite, this is unnecessary and removed.
- A few of my OS X cookbooks have the plist file for use with
  `mac_os_x_plist` LWRP. These are simply moved to my
  [workstation data bag](https://github.com/jtimberman/workstation-chef-repo/blob/master/cookbooks/workstation/recipes/default.rb#L72-L76).

Finally, one special case is
[Fletcher Nichol's rbenv cookbook](http://fnichol.github.com/chef-rbenv/).
The `rbenv::user_install` recipe manages `/etc/profile.d/rbenv.sh`,
which requires root privileges. However, on my workstations where I
use this particular cookbook, I run Chef as my user, so I had to
comment this resource out. To allow for a non-privileged user running
Chef, the better approach is to determine whether to manage that file
by using an attribute, so I
[opened a pull request](https://github.com/fnichol/chef-rbenv/pull/20),
which is now merged. Now I just have the attribute set to `false` in
my workstation role, and can use the cookbook unmodified.

## Remove Unused and Berkshelf Cookbooks

Removing the unused cookbooks, and the cookbooks managed by Berkshelf
was simple. First, each cookbook gained an entry in the Berksfile. For
example, `apache2`.

```ruby
cookbook "apache2"
```

Next, the cookbook was deleted from the Chef Server. I did this,
purging all versions, because I planned to upload all the cookbooks as
resolved by Berkshelf.

```
knife cookbook delete -yap apache2
```

Finally, I removed the cookbook from the git repository.

```
git rm -r cookbooks/apache2
git add Berksfile
git commit -m 'apache2 is managed by Berkshelf'
```

The cookbooks that I didn't plan to use, I simply didn't add to
Berkshelf, and removed them all in one commit.

# After Berkshelf

The net effect of this change is a simpler, easier to manage
repository. I now have only 23 cookbooks in my `cookbooks` directory.
Some of those are candidates for refactoring and updating to the
upstream ones, I just didn't get to that yet. Most of them are
"internal" cookbooks that aren't published, since they're specific for
my internal network, such as my housepub-samba or housepub-openvpn
cookbooks.

On my Chef Server, I have 90 total cookbooks, which means 67 are
managed by Berkshelf. I have 62 entries in my Berksfile, and some of
those are dependencies of others, which means that can be refactored
some as well.

The workflow is simpler, and there's fewer moving parts to worry about
changing. I think this is a net positive for this since I do it in my
free time. However, there's a couple of issues, which should be
addressed in Berkshelf soon.

First,
[Berkshelf issue #190](https://github.com/RiotGames/berkshelf/issues/190),
which would have `berks update` take a single cookbook to update.
Currently, it has to update all the cookbooks, and this takes time for
impatient people.

Second,
[issue #191](https://github.com/RiotGames/berkshelf/issues/191), which
would allow `berks upload` to take a single cookbook to upload.
Normally, one could just use `knife cookbook upload`, but the
directory where Berkshelf stores cookbooks it is managing are not
located in the `cookbook_path`, and the knife command uses the
directory name a the cookbook name. Berkshelf creates directories like
`~/.berkshelf/cookbooks/apache2-1.3.0`, so the way to upload Berkshelf
managed cookbooks is all together with the `berks upload` command.
This isn't a huge deal for me as I already uploaded all the cookbooks
I've been using once.

All in all, I am happy with this workflow, though. It is simple and
hassle-free for me. Plus, I have more flexibility for maintaining my
additional non-Opscode cookbooks.
