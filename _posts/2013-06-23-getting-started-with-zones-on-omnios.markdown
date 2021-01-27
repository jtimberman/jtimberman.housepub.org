---
layout: post
title: "Getting Started With Zones on OmniOS"
date: 2013-06-23 23:47
comments: true
categories: [omnios, illumos, zones, virtualization]
---

I've become enamored with
[IllumOS](http://wiki.illumos.org/display/illumos/illumos+Home)
recently. Years ago, I used Solaris (2.5.1 through 8) at IBM.
Unfortunately (for me), I stopped using it before Solaris 10 brought
all the cool toys to the yard - zones, zfs, dtrace, SMF. Thanks to
OmniTI's excellent IllumOS distribution,
[OmniOS](http://omnios.omniti.com), I'm getting acclimated with the
awesomeness. I plan to write more about my experiences here.

First up, I spent today playing with zones. Zones are a kernel-level
container technology similar to Linux containers/cgroups, or BSD
jails. They're fast and lightweight. At least two of the plans I have
for them:

1. Segregating the services on my home-server.
2. Adding support to various tools in Chef's ecosystem.

The following is basically a compilation of several different blog
posts and documentation collections I've been poring over. Like most
technical blog writers, I'm posting this so I can find it later :-).

## Hardware

I have a number of options for learning OmniOS. I have spare hardware,
or VMware, or
[OmniTI's Vagrant box](http://omnios.omniti.com/wiki.php/Installation#UsingVagrant).
I'm doing all three of these, but the main use will be on physical
hardware, as I'm planning to port the aforementioned server to OmniOS
(#1, above).

The details of the hardware are not important, except that I have a
hard disk device `c3t1d0`, and a physical NIC device `nge1` that are
devoted to zones. To adapt these instructions for your own
installation, change those device names where appropriate.

You can find the name of the disk device to use in your system with
the `format` command.

    root@menthe:~# format
    Searching for disks...done


    AVAILABLE DISK SELECTIONS:
           0. c3t0d0 <ATA-WDCWD1500AHFD-0-7QR5 cyl 18238 alt 2 hd 255 sec 63>
              /pci@0,0/pci1043,cb84@d/disk@0,0
           1. c3t1d0 <ATA-SAMSUNG HD501LJ-0-12-465.76GB>
              /pci@0,0/pci1043,cb84@d/disk@1,0
    Specify disk (enter its number): ^D

Here I wanted to use the Samsung disk.

Use `dladm` to find the network devices:

    root@menthe:~# dladm show-phys
    LINK         MEDIA                STATE      SPEED  DUPLEX    DEVICE
    nge0         Ethernet             up         1000   full      nge0
    nge1         Ethernet             up         1000   full      nge1

## Setup

The example zone here is named `base`. Replace `base` with any zone
name you wish, e.g. `webserver37` or `noodlebarn`. It's also worth
noting that I'm going to use DHCP, rather than static networking here.
There are plenty of guides out there for static networking, and I had
to hunt around for DHCP. Also worth noting is that this was all
performed right after installing the OS.

First, create a zpool to use for zones. This is a 500G disk, so I have
plenty of space.

    zpool create zones c3t1d0

Next, create a VNIC on the interface which is devoted to zones
(`nge1`). It can be named anything, but must end with a number.

    dladm create-vnic -l nge1 vnicbase0

Rather than use the `zonecfg` REPL, I used the following configuration
file, for repeatability.

    create -b
    set zonepath=/zones/base
    set ip-type=exclusive
    set autoboot=false
    add net
    set physical=vnicbase0
    end
    commit

Use this config file to configure the zone with `zonecfg`.

    zonecfg -z base -f base.conf

Now we're ready to install the OS in the new zone. This may take
awhile as all the packages need to be downloaded.

    zoneadm -z base install

The default `nsswitch.conf(4)` does not use DNS for hosts. This is
fairly standard for Solaris/IllumOS. Also, the `resolv.conf(4)` is not
configured automatically, which is a departure from automagic Linux
distributions (and a thing I agree with).

    cp /etc/nsswitch.dns /etc/resolv.conf /zones/base/root/etc

OmniOS does not use
[`sysidcfg`](http://lists.omniti.com/pipermail/omnios-discuss/2012-December/000323.html),
so the way to make the new zone boot up with an interface configured
for DHCP is to write out the `ipadm.conf` configuration for `ipadm`.
The following is `base.ipadm.conf` that I used, with the `vnicbase0`
VNIC created with `dladm` earlier.

    _ifname=vnicbase0;_family=2;
    _ifname=vnicbase0;_family=26;
    _ifname=vnicbase0;_aobjname=vnicbase0/v4;_dhcp=-1,no;

Copy this file to the zone.

    cp base.ipadm.conf /zones/base/root/etc/ipadm/ipadm.conf

Now, boot the zone.

    zoneadm -z base boot

Now you can log into the newly created zone and verify that things are
working, and do any further configuration required.

    zlogin -e ! base

I use `!` as the escape character because I'm logging into my global
zone over SSH. This means you disconnect with `!.` instead of `~.`.

Once complete, the zone can be cloned.

## Clone a Zone

I'm going to clone the `base` zone to `clonebase`. Again, rename this
to whatever you like.

First, a zone must be halted before it can be cloned.

    zoneadm -z base halt

Now, create a new VNIC for the zone.

    dladm create-vnic -l nge1 clonebase

Read the `base` zone's configuration, and replace `base` with
`clonebase`.

    zonecfg -z base export | sed 's/base/clonebase/g' | tee clonebase.conf

Then, create the new zone configuration, and clone the base zone.

    zonecfg -z clonebase -f clonebase.conf
    zoneadm -z clonebase clone base

Again, ensure that the network configuration to use DNS is available.

    cp /etc/nsswitch.dns /etc/resolv.conf /zones/clonebase/root/etc

Create the `ipadm.conf` config for the new zone. I named it `clonebase.ipadm.conf`

    sed 's/base/clonebase/g' base.ipadm.conf > clonebase.ipadm.conf

Now copy this to the zone.

    cp clonebase.ipadm.conf /zones/clonebase/root/etc/ipadm/ipadm.conf

Finally, boot the new zone.

    zoneadm -z clonebase boot

Login and verify the new zone.

    zlogin -e ! clonebase

## Cleaning Up

Use the following to clean up the zone when it's not needed anymore.

    zone=clonebase
    zoneadm -z $zone halt
    zoneadm -z $zone uninstall -F
    zonecfg -z $zone delete -F

## Sans Prose

[This gist](https://gist.github.com/jtimberman/5848129) contains all
the things I did above minus the prose.

## What's Next?

I have a few goals in mind for this system. First of all, I want to
manage the zones with Chef, of course. Some of the functions of the
zones may be:

* IPS package repository
* Omnibus build system for OmniOS
* Adding OmniOS support to cookbooks

I also want to facilitate plugins and the ecosystem around Chef for
IllumOS, including zone based knife, vagrant and test-kitchen plugins.

Finally, I plan to convert my Linux home-server to OmniOS. There are a
couple things I'm running that will require Linux (namely
[Plex](http://www.plexapp.com)), but fortunately,
[OmniOS has KVM](http://omnios.omniti.com/wiki.php/VirtualMachinesKVM)
thanks to [SmartOS](http://smartos.org).

## References

The following links were helpful in composing this post, and of course
for the reference material they contain.

* [http://zero-knowledge.org/post/74](http://zero-knowledge.org/post/74)
* [https://blogs.oracle.com/mandalika/entry/solaris_10_zone_creation_for](https://blogs.oracle.com/mandalika/entry/solaris_10_zone_creation_for)
* [http://www.oracle.com/technetwork/articles/servers-storage-admin/o11-092-s11-zones-intro-524494.html](http://www.oracle.com/technetwork/articles/servers-storage-admin/o11-092-s11-zones-intro-524494.html)
* [http://lists.omniti.com/pipermail/omnios-discuss/2012-December/000322.html](http://lists.omniti.com/pipermail/omnios-discuss/2012-December/000322.html)
* [http://lists.omniti.com/pipermail/omnios-discuss/2012-December/000323.html](http://lists.omniti.com/pipermail/omnios-discuss/2012-December/000323.html)
* [http://omnios.omniti.com/ticket.php/11](http://omnios.omniti.com/ticket.php/11) (related to above list post(s))
