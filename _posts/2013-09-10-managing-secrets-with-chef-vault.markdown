---
layout: post
title: "Managing Secrets With Chef Vault"
date: 2013-09-10 09:30
comments: true
categories: [chef, security]
---

Two years ago, I wrote a post about
[using Chef encrypted data bags](https://jtimberman.housepub.org/blog/2011/08/06/encrypted-data-bag-for-postfix-sasl-authentication/)
for SASL authentication with Postfix. At the time, my ISP didn't allow
non-authenticated SMTP, so I had to find a solution so I could get
cronspam and other vital email from my servers at home. I've since
switched ISPs to one that doesn't care so much about this, so I'm not
using any of that code anymore.

However, that doesn't mean I don't have secrets to manage! I actually
don't for my personal systems due to what I'm managing with Chef now,
but we certainly do for Opscode's hosted Enterprise Chef environment.
The usual suspects for any web application are required: database
passwords, SSL certificates, service API tokens, etc.

We're evaluating chef-vault as a possible solution. This blog post
will serve as notes for me so I can remember what I did when my
terminal history is gone, and hopefully information for you to be able
to use in your own environment.

# Chef Vault

Chef Vault is an
[open source project](https://github.com/Nordstrom/chef-vault)
published by [Nordstrom](http://nordstrom.com). It is distributed as a
RubyGem. You'll need it installed on your local workstation so you can
encrypt sensitive secrets, and on any systems that need to decrypt
said secrets. Since the workstation is where we're going to start,
install the gem. I'll talk about using this in a recipe later.

```
% gem install chef-vault
```

# Use Cases

Now, for the use cases, I'm going to take two fairly simple examples,
and explain how chef-vault works along the way.

1. A username/password combination. The `vaultuser` will be created on
the system with Chef's built-in `user` resource.
2. A file with sensitive content. In this case, I'm going to use a
junk RSA private key for `vaultuser`.

Secrets are generally one of these things. Either a value passed into
a command-line program (like `useradd`) or a file that should live on
disk (like an SSL certificate or RSA key).

# Command-line Structure

Chef Vault includes knife plugins to allow you to manage the secrets
from your workstation, uploading them to the Chef Server just like
normal data bags. The secrets themselves live in Data Bags on the Chef
Server. The "bag" is called the "vault" for chef-vault.

After installation, the `encrypt` and `decrypt` sub-commands will be
available for knife.

```
knife encrypt create [VAULT] [ITEM] [VALUES] --mode MODE --search SEARCH --admins ADMINS --json FILE
knife encrypt delete [VAULT] [ITEM] --mode MODE
knife encrypt remove [VAULT] [ITEM] [VALUES] --mode MODE --search SEARCH --admins ADMINS
knife rotate secret [VAULT] [ITEM] --mode MODE
knife encrypt update [VAULT] [ITEM] [VALUES] --mode MODE --search SEARCH --admins ADMINS --json FILE
knife decrypt [VAULT] [ITEM] [VALUES] --mode MODE
```

The
[README](https://github.com/Nordstrom/chef-vault/blob/master/README.md)
and
[Examples](https://github.com/Nordstrom/chef-vault/blob/master/KNIFE_EXAMPLES.md)
document these quite well.

## Mode: Solo vs Client

I'm using Chef with a Chef Server (Enterprise Chef), so I'll specify
`--mode client` for the knife commands.

It is important to note the `MODE` in the chef-vault knife plugin
commands affects where the encrypted data bags will be saved. Chef
supports data bags with both Solo and Client/Server use. When using
chef-solo, you'll need to configure `data_bag_path` in your
`knife.rb`. That is, even if you're using Solo, since these are knife
plugins, the configuration is for knife, not chef-solo. I'm using a
Chef Server though, so I'm going to use `--mode client`.

# Create a User with a Password

The user I'm going to create is the arbitrarily named `vaultuser`,
with the super secret password, `chef-vault`. I'm going to use this on
a Linux system with SHA512 hashing, so first I generate a password
using mkpasswd:

```
% mkpasswd -m sha-512
Password: chef-vault
$6$VqEIDjsp$7NtPMhA9cnxvSMTE9l7DMmydJJEymi9b4t1Vhk475vrWlfxMgVb3bDLhpk/RZt0J3X7l5H8WnqFgvq3dIa9Kt/
```

**Note**: This is the `mkpasswd(1)` command from the Ubuntu 10.04
  [mkpasswd package](http://packages.ubuntu.com/lucid/mkpasswd).

## Create the Item

The command I'm going to use is `knife encrypt create` since this is a
new secret. I'll show two examples. First, I'll pass in the raw JSON
data as "values". You would do this if you're not going to store the
unencrypted secret on disk or in a repository. Second, I'll pass a
JSON file. You would do this if you want to store the unencrypted
secret on disk or in a repository.

```
% knife encrypt create secrets vaultuser \
  '{"vaultuser":"$6$VqEIDjsp$7NtPMhA9cnxvSMTE9l7DMmydJJEymi9b4t1Vhk475vrWlfxMgVb3bDLhpk/RZt0J3X7l5H8WnqFgvq3dIa9Kt/"}' \
  --search 'role:base' \
  --admins jtimberman --mode client
```

The `[VALUES]` in this command is raw JSON that will be created in the
data bag item by `chef-vault`. The `--search` option tells chef-vault
to use the **public** keys of the nodes matching the SOLR query for
encrypting the value. Then during the Chef run, chef-vault uses those
node's **private** keys to decrypt the value. The `--admins` option tells chef-vault
the list of users on the Chef Server who are also allowed to decrypt
the secret. This is specified as a comma separated string for
multiple admins. Finally, as I mentioned, I'm using a Chef Server so I
need to specify `--mode client`, since "solo" is the default.

Here's the equivalent, using a JSON file named `secrets_vaultuser.json`. It has the content:

```json
{"vaultuser":"$6$VqEIDjsp$7NtPMhA9cnxvSMTE9l7DMmydJJEymi9b4t1Vhk475vrWlfxMgVb3bDLhpk/RZt0J3X7l5H8WnqFgvq3dIa9Kt/"}
```

The command is:

```text
% knife encrypt create secrets vaultuser \
  --json secrets_vaultuser.json
  --search 'role:base' \
  --admins jtimberman --mode client
```

Now, let's see what has been created on the Chef Server. I'll use the
core Chef knife plugin, `data bag item show` for this.

```text
% knife data bag show secrets
vaultuser
vaultuser_keys
```

I now have a "secrets" data bag, with two items. The first,
`vaultuser` is the one that contains the actual secret. Let's see:

```text
% knife data bag show secrets vaultuser
id:        vaultuser
vaultuser:
  cipher:         aes-256-cbc
  encrypted_data: j+/fFM7ist6I7K360GNfzSgu6ix63HGyXN2ZAd99R6H4TAJ4pQKuFNpJXYnC
  SXA5n68xn9frxHAJNcLuDXCkEv+F/MnW9vMlTaiuwW/jO++vS5mIxWU170mR
  EgeB7gvPH7lfUdJFURNGQzdiTSSFua9E06kAu9dcrT83PpoQQzk=
  iv:             cu2Ugw+RpTDVRu1QaaAfug==
  version:        1
```

As you can see, I have encrypted data. I also told chef-vault that my
user can decrypt this. I need to use the knife plugin to do so:

```text
% knife decrypt secrets vaultuser 'vaultuser' --mode client
secrets/vaultuser
	vaultuser: $6$VqEIDjsp$7NtPMhA9cnxvSMTE9l7DMmydJJEymi9b4t1Vhk475vrWlfxMgVb3bDLhpk/RZt0J3X7l5H8WnqFgvq3dIa9Kt/
```

The `'vaultuser'` in quotes is the key from the hash of JSON data that
I specified earlier. As you can see, the password is that which was
generated from the mkpasswd command earlier.

But what nodes have access to decrypt this password? That's what
chef-vault stored in the `vaultuser_keys` item. Let's look:

```text
% knife data bag show secrets vaultuser_keys
admins:              jtimberman
clients:
  os-945926465950316
  os-2790002246935003
id:                  vaultuser_keys
jtimberman:          0Q2bhw/kJl2aIVEwqY6wYhrrfdz9fdsf8tCiIrBih2ZORvV7EEIpzzKQggRX
4P4vnVQjMjfkRwIXndTzctCJONQYF50OSZi5ByXWqbich9iCWvVIbnhcLWSp
z5mQoSTNXyZz/JQZGnubkckh4wGLBFDrLJ6WKl6UNXH1dRwqDNo5sEK7/3Wn
b4ztVSRxzB01wVli0wLvFSZzGsKYJYINBcidnbIgLh/xGYGtBJVlgG2z/7TV
uN0b/qvGj8VlhbS6zPlwh39O3mexDdkLwry/+gbO1nj8qKNkKDKaix5zypwE
XdmdfMKNYGaM6kzG8cwuKZXLAgGAgblVUB1HP8+8kQ==

os-2790002246935003: kGQLsxsFmBe9uPuWxZpKiNBnqJq55hQZJLgaKdjG2Vvivv98RrFGz1y8Xbwe
uzeSgPgAURCZmxpNxpHrwvvKcvL77sBOL6TTKiNzs8n5B3ZOawy17dsuG24v
41R0cRMnYLgbLcjln9dpVe4Esr4goPxko+1XqBPik1SBapthQq/pLUJ1BIKh
Fxu1QVGj1w4HPUftLaUzeS33jKbtfvgZyZsYZBdVCVEVidOxC90WRf4wtkd6
Ueyj+0gd1QKv84Q387O1R5LtRMS6u+17PJinrcRIkVNZ6P1z6oT2Dasfvrex
rK3s5vD7v6jpkUW12Wj74Lz3Z6x3sKuIDzCtvEUnWw==

os-945926465950316:  XzTJrJ3TZZZ1u9L9p6DZledf3bo2ToH2yrLGZQKPV6/ANzElHXGcYrEdtP0q
14Nz1NzsqEftzviAebUUnc6ke91ltD8s6hNQQrPJRqkUoDlM7lNEwiUiz/dD
+sFI6CSzQptO3zPrUbAlUI1Zog5h7k/CCtiYtmFRD6wbAWnxmCqvLhO1jwqL
VNJ1vfjlFsG77BDm2HFw7jgleuxRGYEgBfCCuBuW70FAdUTvNHIAwKQVkfU/
Am75UYm7N4N0E+W76ZwojLoYtXXTV/iOGG1cw3C75SVAmCsBOuxUK/otub67
zsNDsKToKa+laxzXGylrmkTricYXIqVpIQO8OL5nnw==
```

As we can see, I have two nodes that are API clients with access to
decrypt the data bag items. These values are all generated by
chef-vault, and I'll talk about how to update the list and rotate
secrets later in this post.

## Manage a User Password

Let's manage a user resource with a password set to the value from our
encrypted data bag using Chef Vault.

First, I created a cookbook named `vault`, and added it to the base
role. It contains the following recipe:

```ruby
chef_gem "chef-vault"
require "chef-vault"

vault = ChefVault::Item.load("secrets", "vaultuser")

user "vaultuser" do
  password vault['vaultuser']
  home "/home/vaultuser"
  supports :manage_home => true
  shell "/bin/bash"
  comment "Chef Vault User"
end
```

Let me break this down.

```ruby
chef_gem "chef-vault"
require "chef-vault"
```

`chef-vault` is distributed as a RubyGem, and I want to use it in my
recipe(s), so here I use the
[`chef_gem` resource](http://docs.opscode.com/resource_chef_gem.html).
Then, I require it like any other Ruby library.

```ruby
vault = ChefVault::Item.load("secrets", "vaultuser")
```

This is where the decryption happens. If I do this under a
`chef-shell`, I can see:

```text
chef:recipe > vault = ChefVault::Item.load("secrets", "vaultuser")
 => data_bag_item["secrets", "vaultuser", {"id"=>"vaultuser", "vaultuser"=>"$6$VqEIDjsp$7NtPMhA9cnxvSMTE9l7DMmydJJEymi9b4t1Vhk475vrWlfxMgVb3bDLhpk/RZt0J3X7l5H8WnqFgvq3dIa9Kt/"}]
```

`ChefVault::Item.load` takes two arguments, the "vault" or data bag,
in this case `secrets`, and the "item", in this case `vaultuser`. It
returns a data bag item. Then in the
[`user` resource](http://docs.opscode.com/resource_user.html), I use
the password:

```ruby
user "vaultuser" do
  password vault['vaultuser']
  home "/home/vaultuser"
  supports :manage_home => true
  shell "/bin/bash"
  comment "Chef Vault User"
end
```

The important resource attribute here is `password`, where I'm using
the local variable, `vault` and the `vaultuser` key from the item as
decrypted by `ChefVault::Item.load`. When Chef runs, it will look like
this:

```text
Recipe: vault::default
  * chef_gem[chef-vault] action install
    - install version 2.0.1 of package chef-vault
  * chef_gem[chef-vault] action install (up to date)
  * user[vaultuser] action create
    - create user user[vaultuser]
```

Now, I can su to `vaultuser` using the password I created:

```text
ubuntu@-2790002246935003:~$ su - vaultuser
Password: chef-vault
vaultuser@os-2790002246935003:~$ id
uid=1001(vaultuser) gid=1001(vaultuser) groups=1001(vaultuser)
vaultuser@os-2790002246935003:~$ pwd
/home/vaultuser
```

Yay! To show that the user was created with the right password,
here's the DEBUG log output:

```text
INFO: Processing user[vaultuser] action create ((irb#1) line 12)
DEBUG: user[vaultuser] user does not exist
DEBUG: user[vaultuser] setting comment to Chef Vault User
DEBUG: user[vaultuser] setting password to $6$VqEIDjsp$7NtPMhA9cnxvSMTE9l7DMmydJJEymi9b4t1Vhk475vrWlfxMgVb3bDLhpk/RZt0J3X7l5H8WnqFgvq3dIa9Kt/
DEBUG: user[vaultuser] setting shell to /bin/bash
INFO: user[vaultuser] created
```

Next, I'll create a secret that is a file rendered on the system.

# Create a Private SSH Key

Suppose this `vaultuser` is to be used for deploying code by cloning a
repository. It will need a private SSH key to authenticate, so I'll
create one, with an empty passphrase in this case.

```text
% ssh-keygen -b 4096 -t rsa -f vaultuser-ssh
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in vaultuser-ssh.
Your public key has been saved in vaultuser-ssh.pub.
```

Get the SHA256 checksum of the private key. I use SHA256 because
that's what Chef uses for file content. We'll use this to verify
content later.

```text
% sha256sum vaultuser-ssh
a83221c243c9d39d20761e87db6c781ed0729b8ff4c3b330214ebca26e2ea89d  vaultuser-ssh
```

Assume that I also
[created the SSH key on GitHub](https://help.github.com/articles/generating-ssh-keys)
for this user.

In order to have a file's contents be a JSON value for the data bag
item, I'll remove the newlines (`\n`), and generate the JSON:

```text
ruby -rjson -e 'puts JSON.generate({"vaultuser-ssh-private" => File.read("vaultuser-ssh")})' \
  > secrets_vaultuser-ssh-private.json
```

Now, create the secret on the Chef Server:

```text
knife encrypt create secrets vaultuser-ssh-private \
  --search 'role:base' \
  --json secrets_vaultuser-ssh-private.json \
  --admins jtimberman \
  --mode client
```

Let's verify the server has what we need:

```text
% knife data bag show secrets vaultuser-ssh-private
id:                    vaultuser-ssh-private
vaultuser-ssh-private:
  cipher:         aes-256-cbc
  encrypted_data: mRRToM2N/0F+OyJxkYlHo/cUtHSIuy69ROAKuGoHIhX9Fr5vFTCM4RyWQSTN
  trimmed for brevity even though scrollbars
% knife decrypt secrets vaultuser-ssh-private 'vaultuser-ssh-private' --mode client
secrets/vaultuser-ssh-private
	vaultuser-ssh-private: -----BEGIN RSA PRIVATE KEY-----
trimmed for brevity even though scrollbars
```

## Manage the Key File

Now, I'll manage the private key file with the vault cookbook.

```ruby
vault_ssh = ChefVault::Item.load("secrets", "vaultuser-ssh-private")

directory "/home/vaultuser/.ssh" do
  owner "vaultuser"
  group "vaultuser"
  mode 0700
end

file "/home/vaultuser/.ssh/id_rsa" do
  content vault_ssh["vaultuser-ssh-private"]
  owner "vaultuser"
  group "vaultuser"
  mode 0600
end
```

Again, let's break this up a bit. First, load the item from the
encrypted data bag like we did before.

```ruby
vault_ssh = ChefVault::Item.load("secrets", "vaultuser-ssh-private")
```

Next, make sure that the vaultuser has an `.ssh` directory with the
correct permissions.

```ruby
directory "/home/vaultuser/.ssh" do
  owner "vaultuser"
  group "vaultuser"
  mode 0700
end
```

Finally, manage the content of the private key file with a `file`
resource and the `content` resource attribute. The value of
`vault_ssh["vaultuser-ssh-private"]` will be a string, with `\n`'s
embedded, but when it's rendered on disk, it will display properly.

```ruby
file "/home/vaultuser/.ssh/id_rsa" do
  content vault_ssh["vaultuser-ssh-private"]
  owner "vaultuser"
  group "vaultuser"
  mode 0600
end
```

And now run chef on a target node:

```text
Recipe: vault::default
  * chef_gem[chef-vault] action install (up to date)
  * user[vaultuser] action create (up to date)
  * directory[/home/vaultuser/.ssh] action create
    - create new directory /home/vaultuser/.ssh
    - change mode from '' to '0700'
    - change owner from '' to 'vaultuser'
    - change group from '' to 'vaultuser'

  * file[/home/vaultuser/.ssh/id_rsa] action create
    - create new file /home/vaultuser/.ssh/id_rsa with content checksum a83221
        --- /tmp/chef-tempfile20130909-1918-1v5hezo	2013-09-09 22:41:21.887239999 +0000
        +++ /tmp/chef-diff20130909-1918-xwbmsn	2013-09-09 22:41:21.883240065 +0000
        @@ -0,0 +1,51 @@
        +-----BEGIN RSA PRIVATE KEY-----
        +MIIJJwIBAAKCAgEAtZmwFTlVOBbr2ZfG+cDtUGx04xCcgaa0p0ISmeyMEoGYH/CP
        output trimmed because its long even though scrollbars again
```

Note the content checksum, `a83221`. This will match the checksum of
the source file from earlier (scroll up!), and the one rendered:

```text
ubuntu@os-2790002246935003:~$ sudo sha256sum /home/vaultuser/.ssh/id_rsa
a83221c243c9d39d20761e87db6c781ed0729b8ff4c3b330214ebca26e2ea89d  /home/vaultuser/.ssh/id_rsa
```

Yay! Now, we can SSH to GitHub (note, this is fake GitHub for example
purposes).

```text
ubuntu@os-2790002246935003:~$ su - vaultuser
Password: chef-vault
vaultuser@os-2790002246935003:~$ ssh -i .ssh/id_rsa github@172.31.7.15
$ hostname
os-945926465950316
$ id
uid=1002(github) gid=1002(github) groups=1002(github)
```

# Updating a Secret

What happens if we need to update a secret? For example, if an
administrator leaves the organization, we will want to change the
`vaultuser` password (and SSH private key).

```text
% mkpasswd -m sha-512
Password: gone-user
$6$zM5STNtXdmsrOSm$svJr0tauijqqxTjnMIGJGJPv5V3ovMFCQo.ZDBleiL.yOxcngRqh9yAjpMAsMBA7RlKPv5DKFd1aPZm/wUoKs.
```

The `encrypt create` command will return an error if the target
already exists:

```text
% knife encrypt create secrets vaultuser --search 'role:base' --json secrets_vaultuser.json --admins jtimberman --mode client
ERROR: ChefVault::Exceptions::ItemAlreadyExists: secrets/vaultuser already exists, use 'knife encrypt remove' and 'knife encrypt update' to make changes.
```

So, I need to use `encrypt update`. **Note** make sure that the
contents of the JSON file are valid JSON.

```text
% knife encrypt update secrets vaultuser --search 'role:base' --json secrets_vaultuser.json --admins jtimberman --mode client
```

`encrypt update` only updates the things that change, so I can also
shorten this:

```text
% knife encrypt update secrets vaultuser --json secrets_vaultuser.json --mode client
```

Since the search and the admins didn't change.

Verify it:

```text
% knife decrypt secrets vaultuser 'vaultuser' --mode client
secrets/vaultuser
	vaultuser: $6$zM5STNtXdmsrOSm$svJr0tauijqqxTjnMIGJGJPv5V3ovMFCQo.ZDBleiL.yOxcngRqh9yAjpMAsMBA7RlKPv5DKFd1aPZm/wUoKs.
```

Now, just run Chef on any nodes affected.

```text
Recipe: vault::default
  * chef_gem[chef-vault] action install (up to date)
  * user[vaultuser] action create
    - alter user user[vaultuser]

  * directory[/home/vaultuser/.ssh] action create (up to date)
  * file[/home/vaultuser/.ssh/id_rsa] action create (up to date)
Chef Client finished, 1 resources updated
```

And su to the vault user with the `gone-user` password:

```text
ubuntu@os-2790002246935003:~$ su - vaultuser
Password: gone-user
vaultuser@os-2790002246935003:~$
```

# Managing Access to Items

There are three common scenarios which require managing the access to an item
in the vault.

1. A system needs to be taken offline, or otherwise prevented from
accessing the item(s).
2. A new system comes online that needs access.
3. An admin user has left the organization.
4. A new admin user has joined the organization.

Suppose we have a system that we need to take offline for some reason,
so we want to disable its access to a secret. Or, perhaps we have a
user who has left the organization that was an admin. We can do that in a
few ways.

## Update the Vault Item

The most straightforward way to manage access to an item is to use the
`update` or `remove` sub-commands.

### Remove a System

Suppose I want to remove node `DEADNODE`, I can qualify the search to
exclude the node named `DEADNODE`:

```text
% knife encrypt update secrets vaultuser \
  --search 'role:base NOT name:DEADNODE' \
  --json secrets_vaultuser.json \
  --admins jtimberman --mode client
```

Note, as before, admins didn't change so I don't need to pass that
argument.

### Add a New System

If the node has run Chef and is indexed on the Chef Server already,
simply rerun the update command with the search:

```text
% knife encrypt update secrets vaultuser \
  --search 'role:base' \
  --json secrets_vaultuser.json \
  --admins jtimberman --mode client
```

There's a bit of a "Chicken and Egg" problem here, in that a new node
might not be indexed for search if it tried to load the secret during
a bootstrap beforehand. For example, if I create an OpenStack instance
with the base role in its run list, the node doesn't exist for the
search yet. A solution here is to create the node with an empty run
list, allowing it to register with the Chef Server, and then use
`knife bootstrap` to rerun Chef with the proper run list. This is
annoying, but no one claimed that chef-vault would solve *all*
problems with shared secret management :-).

### Remove an Admin

The admins argument takes a list. Earlier, I only had my userid as an
admin. Suppose I created the item with "bofh" as an admin too:

```text
% knife encrypt create secrets vaultuser \
  --search 'role:base' \
  --json secrets_vaultuser.json \
  --admins "jtimberman,bofh" --mode client
```

To remove the bofh user, use the `encrypt remove` subcommand. In this
case, the `--admins` argument is the list of admins to remove, rather
than add.

```text
% knife encrypt remove secrets vaultuser --admins bofh --mode client
```

### Add a New Admin

I want to add "mandi" as an administrator because she's awesome and
will help manage our secrets. As above, I just pass a comma-separated
string, `"jtimberman,mandi"` to the `--admins` argument.

```text
% knife encrypt update secrets vaultuser \
  --search 'role:base' \
  --json secrets_vaultuser.json \
  --admins "jtimberman,mandi" --mode client
```

## Regenerate the Client

The heavyhanded way to remove access is to regenerate the API client
on the Chef Server. For example, of my nodes, say I want to remove
`os-945926465950316`:

```text
% knife client reregister os-945926465950316
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAybzwv53tDLIzW+GHRJwLthZmiGTfZVyqQX6m6RGuZjemEIdy
trim trim
```

If you're familiar with Chef Server's authentication cycle, you'll
know that until that private key is copied to the node, it will
completely fail to authenticate. However, once the
`/etc/chef/client.pem` file is updated with the content from the knife
command, we'll see that the node fails to read the Chef Vault item:

```text
================================================================================
Recipe Compile Error in /var/chef/cache/cookbooks/vault/recipes/default.rb
================================================================================


OpenSSL::PKey::RSAError
-----------------------
padding check failed


Cookbook Trace:
---------------
  /var/chef/cache/cookbooks/vault/recipes/default.rb:4:in `from_file'


Relevant File Content:
----------------------
/var/chef/cache/cookbooks/vault/recipes/default.rb:

  1:  chef_gem "chef-vault"
  2:  require "chef-vault"
  3:
  4>> vault = ChefVault::Item.load("secrets", "vaultuser")
  5:
  6:  user "vaultuser" do
  7:    password vault["vaultuser"]
  8:    home "/home/vaultuser"
  9:    supports :manage_home => true
 10:    shell "/bin/bash"
 11:    comment "Chef Vault User"
 12:  end
 13:
```

**Note** I say this is heavy-handed because if you make a mistake, you
  need to re-upload every single secret that this node needs access to.

## Removing Users

We can also remove user access from Enterprise Chef simply by
disassociating that user from the organization on the Chef Server. I
won't show an example of that here, since I'm using Opscode's hosted
Enterprise Chef server and I'm the only admin, however :-).

# Backing Up Secrets

To back up the secrets, as encrypted data from the Chef Server, use
`knife-essentials` (comes with Chef 11+, available as a RubyGem for
Chef 10).

```text
% knife download data_bags/secrets/
Created data_bags/secrets/vaultuser_keys.json
Created data_bags/secrets/vaultuser.json
Created data_bags/secrets/vaultuser-ssh-private_keys.json
Created data_bags/secrets/vaultuser-ssh-private.json
```

For example, the vaultuser.json file looks like this:

```json
{
  "id": "vaultuser",
  "vaultuser": {
    "encrypted_data": "3yREwInxdyKpf8nuTIivXAeuEzHt7o4vF4FsOwmVLHmMWol5nCBoMWF0YdaW\n3P3NpEAAAxYEYeJYdVkrdLqjjB2kTJdx0+ceh/RBHBWqmSeHOWFH9pCRGjV8\nfS5XaTueShb320b/+Ia8iqUJJWg6utnbJCDx+VMcGNggPXgPKC8=\n",
    "iv": "EI+y74Uj2uwq7EVaP+0K6Q==\n",
    "version": 1,
    "cipher": "aes-256-cbc"
  }
}
```

Since these are encrypted using a strong cipher (AES 256), they should
be safe to store in repository. Unless you think the NSA has access to
that repository ;-).

# Conclusion

Secrets management is hard! Especially when you need to store secrets
that are used by multiple systems, services, and people. Chef's
encrypted data bag feature isn't a panacea, but it certainly helps.
Hopefully, this blog post was informative. While I don't always
respond, I do read all comments posted here via Disqus, so let me know
if something is out of whack, or needs an update.
