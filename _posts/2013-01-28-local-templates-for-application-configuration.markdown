---
layout: post
title: "Local Templates for Application Configuration"
date: 2013-01-28 09:57
comments: true
categories: [chef, deployment, applications]
---

Today I joined the
[Food Fight Show](http://foodfightshow.org/2013/01/application-deployment.html)
for a conversation about Application Deployment. Along the way, the
question came up about where to store application specific
configuration files. Should they be stored in a Chef cookbook for
setting up the system for the application? Or shoud they be stored in
the application codebase itself?

The answer is either, as far as Chef is concerned. Chef's
[template resource](http://docs.opscode.com/resource_template.html)
can render a template from a local file on disk, or retrieve the
template from a cookbook. The latter is the most common pattern, so
let's examine the former, using a local file on disk.

For sake of discussion, let's use a Rails application that needs a
`database.yml` file rendered. Also, we'll assume that information
about the application (database user, password, server) we need is
stored in a Chef
[data bag](http://docs.opscode.com/essentials_data_bags_store.html).
Finally, we're going to assume that the application is already
deployed on the system somehow and we just want to render the
database.yml.

The application source tree looks something like this:

```
myapp/
-> config/
    -> database.yml.erb
```

Note that there should not be a database.yml (non-.erb) here, as it
will be rendered with Chef. The deployment of the app will end up
in `/srv`, so the full path of this template is, for example,
`/srv/myapp/current/config/database.yml.erb`. The content of the
template may look like this:

```yaml
<%= @rails_env %>:
  adapter: <%= @adapter %>
  host: <%= @host %>
  database: <%= @database %>
  username: <%= @username %>
  password: <%= @password %>
  encoding: 'utf8'
  reconnect: true
```

The Chef recipe looks like this. Note we'll use a search to find
the first node that should be the database master (there should only
be one). For the adapter, we may have set an attribute in the role
that selects the adapter to use.

```ruby
results = search(:node, "role:myapp_database_master AND environment:#{node.chef_environment}")
db_master = results[0]

template "/srv/myapp/shared/database.yml" do
  source "/srv/myapp/current/config/database.yml.erb"
  local true
  variables(
    :rails_env => node.chef_environment,
    :adapter => db_master['myapp']['db_adapter'],
    :host => db_master['fqdn'],
    :database => "myapp_#{node.chef_environment}",
    :username => "myapp",
    :password => "SUPERSECRET",
  )
end
```

The rendered template, `/srv/myapp/shared/database.yml`, will look
like this:

```yaml
production:
  adapter: mysql
  host: domU-12-31-39-14-F1-C3.compute-1.internal
  database: myapp_production
  username: myapp
  password: SUPERSECRET
  encoding: utf8
  reconnect: true
```

This post is only part of the puzzle, mainly to explain what I
mentioned on the Food Fight Show today. There are a number of
unanswered questions like,

* Should database.yml be .gitignore'd?
* How do developers run the app locally?
* How do I use this with Chef Solo?

As mentioned on the show, there's currently a
[thread](http://lists.opscode.com/sympa/arc/chef/2013-01/msg00392.html)
related to this topic on the Chef mailing list.
