---
layout: post
title: "Chef Reporting API and Resource Updates"
date: 2014-10-07 06:18:34 -0600
comments: true
categories: [chef, knife, reporting, plugins]
---

Have you ever wanted to find a list of nodes that updated a specific resource in a period of time? Such as "show me all the nodes in production that had an application service restart in the last hour"? Or, "which nodes have updated their apt cache recently?" For example,

```
% knife report resource 'role:supermarket-app AND chef_environment:supermarket-prod' execute 'apt-get update'
execute[apt-get update] changed in the following runs:
app-supermarket1.example.com 2230cf30-6d95-4e43-be18-211137eaf802 @ 2014-10-07T14:07:03Z
app-supermarket2.example.com c5e4d7bf-95a6-4385-9d8e-c6f5617ed79b @ 2014-10-07T14:14:04Z
app-supermarket3.example.com c4c4b4bb-91b6-4f73-9876-b24b093c7f1e @ 2014-10-07T14:09:54Z
app-supermarket4.example.com 3eb09034-7539-4a3c-af6d-5b01d35bc63f @ 2014-10-07T13:31:56Z
app-supermarket5.example.com aa48c1d3-da91-4031-a43d-582a577cbf2d @ 2014-10-07T13:35:15Z
Use `knife runs show` with the run UUID to get more info
```

I have released a new knife plugin to do that, but first some background.

At CHEF, we run the community's cookbook site, [Supermarket](https://supermarket.getchef.com). We monitor the systems that run the site with [Sensu](http://sensuapp.org). The current infrastructure runs instances on Amazon Web Services EC2, with an Elastic Load Balancer (ELB) in front of them. As a corrective action for a [Supermarket outage](https://www.getchef.com/blog/2014/07/10/supermarket-intermittent-unresponsiveness-postmortem/), CHEF's operations team added a new check for elevated HTTP 500 responses from the application servers behind the ELB. One thing we found was that when Supermarket was deployed, and the `unicorn` server restarted, we would see elevated 500's, but the site often wouldn't actually be impacted.

The Sensu check is run from a "relay" node. That is, it isn't run on the application servers or the Sensu server - it's run out of band since it's for the ELB. One might imagine we could have similar checks for other services that aren't run on "managed nodes," but that's neither here nor there. The issue is that we get an alert message that looks like this:

```
Sensu Alerts	 ALERT - [i-d1dfd5d9/check-elb-backend-500] - CheckELBMetrics CRITICAL: supermarket-elb; Sum of HTTPCode_Backend_5XX is 2538.0. (expected lower than 30.0); (HTTPCode_Backend_5XX '2538.0' within 300 seconds between 2014-08-19 13:33:36 +0000 to 2014-08-19 13:38:36 +0000) [Playbook].
```

The first part, `[i-d1dfd5d9/check-elb-backend-500]` is the node name and the check that alerted. The node name here is the monitoring relay that runs the check, not the actual node or nodes where Supermarket was deployed and restarted. This is where Chef Reporting comes into play. In Chef Reporting, we can view information about recent Chef client runs, which gives us a graph like this.

![reporting graph](/assets/images/reporting-graph.png)

If we go look at the reports in the Chef Manage console, we can drill down to something like this.

![reporting-resources](/assets/images/reporting-resources.png)

This shows that unicorn was restarted in this run. That's great, but if I'm getting this alert at a time when I'm not particularly coherent (e.g, 2AM), I want a command in a playbook that I can run to get more information quickly without having to log into the webui and click around imprecisely. CHEF publishes a `knife-reporting` gem that has a couple handy sub-commands to retrieve this run data. For example, we can list runs.

```
% knife runs list
node_name:  i-3022aa3b
run_id:     9eccd8f6-876b-4a57-87ac-0b3e7b7ef1e7
start_time: 2014-08-21T17:03:56Z
status:     started

node_name:  i-a09424a8
run_id:     f2b7871a-149b-4fd3-abdc-d74a838d719a
start_time: 2014-08-21T17:00:23Z
status:     success
```

Or, we can display a specific run.

```
% knife runs show eecb04fb-11df-438a-8e81-dd610eb66616
run_detail:
  data:
  end_time:          2014-08-20T17:50:12Z
  node_name:         i-9f22aa94
  run_id:            eecb04fb-11df-438a-8e81-dd610eb66616
  run_list:          ["role[base]","role[supermarket-app]"]
  start_time:        2014-08-20T17:45:37Z
  status:            success
  total_res_count:   261
  updated_res_count: 17
run_resources:
  cookbook_name:    supermarket
  cookbook_version: 2.7.2
  duration:         209
  final_state:
    enabled: false
    running: true
  id:               unicorn
  initial_state:
    enabled: false
    running: true
  name:             unicorn
  result:           restart
  type:             service
  uri:              https://api.opscode.com/organizations/supermarket/reports/org/runs/eecb04fb-11df-438a-8e81-dd610eb66616/15
```

This is handy, but a little limited. What if I want to display only the runs containing the `service[unicorn]` resource?

That's where my `knife-report-resource` plugin helps. At first, it was very much specific to finding unicorn restarts on Supermarket app servers. However, I wanted to make it more general purpose as I think people would want to be able to find when arbitrary resources were updated. This is how it works:

1. Query the Chef Server for a particular set of nodes. For example, `'role:supermarket-app AND chef_environment:supermarket-prod'`.
2. Get all the Chef client runs for a specified time period up until the current time. By default, it starts from one hour ago, but we can pass an ISO8601 timestamp.
3. Iterate over all the runs looking for runs by the nodes that were returned by the search query, gathering the specified resource type and name.
4. Display some nice output with the node's FQDN, the run's UUID, and a timestamp.

From the earlier example:

```
% knife report resource 'role:supermarket-app AND chef_environment:supermarket-prod' execute 'apt-get update'
execute[apt-get update] changed in the following runs:
app-supermarket1.example.com 2230cf30-6d95-4e43-be18-211137eaf802 @ 2014-10-07T14:07:03Z
app-supermarket2.example.com c5e4d7bf-95a6-4385-9d8e-c6f5617ed79b @ 2014-10-07T14:14:04Z
app-supermarket3.example.com c4c4b4bb-91b6-4f73-9876-b24b093c7f1e @ 2014-10-07T14:09:54Z
app-supermarket4.example.com 3eb09034-7539-4a3c-af6d-5b01d35bc63f @ 2014-10-07T13:31:56Z
app-supermarket5.example.com aa48c1d3-da91-4031-a43d-582a577cbf2d @ 2014-10-07T13:35:15Z
Use `knife runs show` with the run UUID to get more info
```

Then, we can drill down further into one of these runs with the `knife-reporting` plugin.

```
% knife runs show 2230cf30-6d95-4e43-be18-211137eaf802
run_detail:
  data:
  end_time:          2014-10-07T14:07:03Z
  node_name:         i-d7fed0df
  run_id:            2230cf30-6d95-4e43-be18-211137eaf802
  run_list:          ["role[base]","role[supermarket-app]"]
  start_time:        2014-10-07T14:03:59Z
  status:            success
  total_res_count:   271
  updated_res_count: 12
run_resources:
  cookbook_name:    chef-client
  cookbook_version: 3.6.0
  duration:         99
  final_state:
    enabled: true
    running: false
  id:               chef-client
  initial_state:
    enabled: true
    running: true
  name:             chef-client
  result:           enable
  type:             runit_service
  uri:              https://api.opscode.com/organizations/supermarket/reports/org/runs/2230cf30-6d95-4e43-be18-211137eaf802/0
...
  cookbook_name:    supermarket
  cookbook_version: 2.11.0
  duration:         8506
  final_state:
  id:               apt-get update
  initial_state:
  name:             apt-get update
  result:           run
  type:             execute
  uri:              https://api.opscode.com/organizations/supermarket/reports/org/runs/2230cf30-6d95-4e43-be18-211137eaf802/5
```

Hopefully you find this plugin useful! It is a RubyGem, and is available on RubyGems.org, and the source is available on GitHub.

* [Supermarket tool page](https://supermarket.getchef.com/tools/knife-report-resource)
* [RubyGem page](https://rubygems.org/gems/knife-report-resource)
* [GitHub repository](https://github.com/jtimberman/knife-report-resource)
