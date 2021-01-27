---
layout: post
title: "Chef::Node.debug_value"
date: 2014-09-02 14:26:21 -0600
comments: true
categories: [chef]
---

Update: As mentioned by [Dan DeLeo](https://twitter.com/kallistec/status/506953082404483072), he discussed this feature on [Chef 11 In-Depth: Attributes Changes](http://www.getchef.com/blog/2013/02/05/chef-11-in-depth-attributes-changes/) last year when Chef 11 was released. I somehow never got a chance to use it, and thought this post would be a helpful example.

Earlier today I was reminded by [Steven Danna](https://github.com/stevendanna) about a newer feature of Chef called `debug_value`. This is a method on the `node` object (`Chef::Node`) which will show where in Chef's attribute hierarchy a particular attribute or sub-attribute was set on the node.

Fire up a chef shell in client mode on the node you want to see:

```
chef-shell -z
```

For example, I'll use my minecraft server, using the excellent [minecraft cookbook](https://supermarket.getchef.com/cookbooks/minecraft).

```
chef > node.run_list
 => role[minecraft-server]
```

The cookbook itself sets a `node['minecraft']` attribute hash.

```
chef > node['minecraft']
 => {"user"=>"mcserver", "group"=>"mcserver", "install_dir"=>"/srv/minecraft", "install_type"=>"vanilla" ... omg a huge hash of attributes}
```

Of note are the [server properties](http://minecraft.gamepedia.com/Server.properties) attributes, which I customize in the role. Here is the `node['minecraft']['properties']` attributes hash on my node:

```
chef > node['minecraft']['properties']
 => {"allow-flight"=>false, "allow-nether"=>true, "difficulty"=>"1", "enable-query"=>false, "enable-rcon"=>false, "enable-command-block"=>false, "force-gamemode"=>true, "gamemode"=>"0", "generate-structures"=>true, "hardcore"=>false, "level-name"=>"creative-survival", "level-seed"=>"", "level-type"=>"DEFAULT", "max-build-height"=>"256", "max-players"=>"20", "motd"=>"It's the will to survive", "online-mode"=>true, "op-permission-level"=>4, "player-idle-timeout"=>0, "pvp"=>"false", "query.port"=>"25565", "rcon.password"=>"", "rcon.port"=>"25575", "server-ip"=>"", "server-name"=>"Housepub", "server-port"=>"25565", "snooper-enabled"=>"false", "spawn-animals"=>true, "spawn-monsters"=>true, "spawn-npcs"=>true, "spawn-protection"=>1, "texture-pack"=>"", "view-distance"=>10, "white-list"=>false}
```

And I can see where these were set using the `#debug_value` method. Each sub-attribute should be passed as an argument.

```
chef > pp node.debug_value('minecraft', 'properties')
[["set_unless_enabled?", false],
 ["default",
  {"allow-flight"=>false,
   "allow-nether"=>true,
   "difficulty"=>1,
   "enable-query"=>false,
   "enable-rcon"=>false,
   "enable-command-block"=>false,
   "force-gamemode"=>false,
   "gamemode"=>0,
   "generate-structures"=>true,
   "hardcore"=>false,
   "level-name"=>"world",
   "level-seed"=>"",
   "level-type"=>"DEFAULT",
   "max-build-height"=>"256",
   "max-players"=>"20",
   "motd"=>"A Minecraft Server",
   "online-mode"=>true,
   "op-permission-level"=>4,
   "player-idle-timeout"=>0,
   "pvp"=>true,
   "query.port"=>"25565",
   "rcon.password"=>"",
   "rcon.port"=>"25575",
   "server-ip"=>"",
   "server-name"=>"Unknown Server",
   "server-port"=>"25565",
   "snooper-enabled"=>true,
   "spawn-animals"=>true,
   "spawn-monsters"=>true,
   "spawn-npcs"=>true,
   "spawn-protection"=>16,
   "texture-pack"=>"",
   "view-distance"=>10,
   "white-list"=>false}],
 ["env_default", :not_present],
 ["role_default",
  {"difficulty"=>"1",
   "gamemode"=>"0",
   "force-gamemode"=>true,
   "motd"=>"It's the will to survive",
   "pvp"=>"false",
   "server-name"=>"Housepub",
   "level-name"=>"creative-survival",
   "spawn-protection"=>1,
   "snooper-enabled"=>"false"}],
 ["force_default", :not_present],
 ["normal", :not_present],
 ["override", :not_present],
 ["role_override", :not_present],
 ["env_override", :not_present],
 ["force_override", :not_present],
 ["automatic", :not_present]]
```

From the role, we can see some properties attributes are set:

```ruby
 ["role_default",
  {"difficulty"=>"1",
   "gamemode"=>"0",
   "force-gamemode"=>true,
   "motd"=>"It's the will to survive",
   "pvp"=>"false",
   "server-name"=>"Housepub",
   "level-name"=>"creative-survival",
   "spawn-protection"=>1,
   "snooper-enabled"=>"false"}],
```

Note that even though these are also set by default, we get them in the output here too.
