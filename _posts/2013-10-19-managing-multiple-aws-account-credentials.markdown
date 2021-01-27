---
layout: post
title: "Managing Multiple AWS Account Credentials"
date: 2013-10-19 17:55
comments: true
categories: [aws, amazon, security]
---

**UPDATE**: All non-default profiles must have their profile name
  start with "profile." Below, this is "profile nondefault." The ruby
  code is updated to reflect this.

In this post, I will describe my local setup for using the
[AWS CLI](http://aws.amazon.com/cli/), the
[AWS Ruby SDK](http://aws.amazon.com/sdkforruby/), and of course the
[Knife EC2 plugin](http://rubygems.org/gems/knife-ec2).

The general practice I've used is to set the appropriate shell
environment variables that are used by default by these tools (and the
"legacy" ec2-api-tools, the java-based CLI). Over time and between
tools, there have been several environment variables set:

```sh
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_DEFAULT_REGION
AWS_SSH_KEY
AMAZON_ACCESS_KEY_ID
AMAZON_SECRET_ACCESS_KEY
AWS_ACCESS_KEY
AWS_SECRET_KEY
```

There is now a config file (ini-flavored) that can be used to set
credentials, `~/.aws/config`. Each ini section in this file is a
different account's credentials. For example:

```
[default]
aws_access_key_id=MY_DEFAULT_KEY
aws_secret_access_key=MY_DEFAULT_SECRET
region=us-east-1
[profile nondefault]
aws_access_key_id=NOT_MY_DEFAULT_KEY
aws_secret_access_key=NOT_MY_DEFAULT_SECRET
region=us-east-1
```

I have two accounts listed here. Obviously, the actual keys are not
listed :). I source a shell script that sets the environment variables
with these values. Before, I maintained a separate script for each
account. Now, I install the `inifile`
[RubyGem](http://rubygems.org/gems/inifile) and use a one-liner for
each of the keys.

```sh
export AWS_ACCESS_KEY_ID=`ruby -rinifile -e "puts IniFile.load(File.join(File.expand_path('~'), '.aws', 'config'))['default']['aws_access_key_id']"`
export AWS_SECRET_ACCESS_KEY=`ruby -rinifile -e "puts IniFile.load(File.join(File.expand_path('~'), '.aws', 'config'))['default']['aws_secret_access_key']"`
export AWS_DEFAULT_REGION="us-east-1"
export AWS_SSH_KEY='jtimberman'
```

This will load the specified file, `~/.aws/config` with the
`IniFile.load` method, retrieving the `default` section's
`aws_access_key_id` value. Then repeat the same for the
`aws_secret_access_key`.

To use the nondefault profile:

```
export AWS_ACCESS_KEY_ID=`ruby -rinifile -e "puts IniFile.load(File.join(File.expand_path('~'), '.aws', 'config'))['profile nondefault']['aws_access_key_id']"`
export AWS_SECRET_ACCESS_KEY=`ruby -rinifile -e "puts IniFile.load(File.join(File.expand_path('~'), '.aws', 'config'))['profile nondefault']['aws_secret_access_key']"`
```

Note that this uses `['profile nondefault']`.

Since different tools historically have used slightly different
environment variables, I export those too:

```sh
export AMAZON_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
export AMAZON_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
export AWS_ACCESS_KEY=$AWS_ACCESS_KEY_ID
export AWS_SECRET_KEY=$AWS_SECRET_ACCESS_KEY
```

I create a separate config script for each account.

The AWS CLI tool will automatically use the `~/.aws/config`, and can
load different profiles with the `--profile` option. The `aws-sdk`
Ruby library will use the environment variables, however. So
authentication in a Ruby script is automatically set up.

```ruby
require 'aws-sdk'
iam = AWS::IAM.new
```

Without this, it would be:

```ruby
require 'aws-sdk'
iam = AWS::IAM.new(:access_key_id => 'YOUR_ACCESS_KEY_ID',
                   :secret_access_key => 'YOUR_SECRET_ACCESS_KEY')
```

Which is a little ornerous.

To use this with `knife-ec2`, I have the following in my
`.chef/knife.rb`:

```ruby
knife[:aws_access_key_id]      = ENV['AWS_ACCESS_KEY_ID']
knife[:aws_secret_access_key]  = ENV['AWS_SECRET_ACCESS_KEY']
```

Naturally, since `knife.rb` is Ruby, I could use `Inifile.load` there,
but I only started using that library recently, and I have my knife
configuration setup already.
