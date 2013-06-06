---
layout: post
title: "Using BGP to serve High-Availability DNS"
date: 2012-01-27 22:48
comments: true
author: Aaron Brady
categories: BGP DNS
---

Routing protocols, like BGP and OSPF, can be used for more than just
establishing connectivity. The best-known example of this is probably using BGP
to blackhole DoS origins, [Remotely Triggered Blackhole Routing][1].

Another use is to provide failover at layer 3, as an alternative to services
like [Heartbeat][2]. This post will explain how to use BGP to provide a highly
available recursive DNS service.

<!--more-->

#### Rationale

Why HA DNS? DNS is already able to handle outages if you specify more than one
name server. Unfortunately, while true, in practice this breaks down. There can
be a significant timeout before rolling over to the next resolver, a timeout
which in many cases is longer than other application-level ones. 

The default on Ubuntu is 5 seconds, the same value as Varnish's default 'first
byte' timeout- in the simplest case this means that all database connections,
reverse lookups or connections to web services could add 5 seconds *each* to
the rendering time of a page. Even if you avoid the request failing, it will be
significantly slower than if all of your resolvers were available.

#### Set up

For this example, there will be two machines serving DNS in an active/active
set up. In the event that one of the machines fails then both sets of IPs will
be handled by the single machine left. [iWeb][4] deploys Ubuntu almost
exclusively, so you'll need to mentally translate if you're using another
distribution.

    192.168.0.0/29:
      192.168.0.1 - Border Router
      192.168.0.2 - Machine A's Real IP
      192.168.0.3 - Machine B's Real IP
      192.168.0.4 - First Virtual IP
      192.168.0.5 - Second Virtual IP
    192.168.0.8/29:
      192.168.0.9 - Border Router (Area 0)
      192.168.0.10 - A Client

#### Network Bit

Configure the networking like below. Machine B is the same as Machine A except
for its real IP; replace 192.168.0.2 with 192.168.0.3.

{% codeblock "Machine A - /etc/network/interfaces" %}

auto lo
iface lo inet loopback

auto lo:0
iface lo:0 inet static
address 192.168.0.4
netmask 255.255.255.255

auto lo:1
iface lo:1 inet static
address 192.168.0.5
netmask 255.255.255.255

auto eth0
iface eth0 inet static
address 192.168.0.2
netmask 255.255.255.248
gateway 192.168.0.1

{% endcodeblock %}

As both of the machines' `eth0` interfaces are in the same LAN they will both
clash by sending out ARP responses for the same IPs. We can tell the kernel to
only answer ARP requests which match the interface the request came in from:

{% codeblock "/etc/sysctl.conf" lang:bash %}
net.ipv4.conf.eth0.arp_ignore = 1
net.ipv4.conf.eth0.arp_announce = 2
{% endcodeblock %}

Reload with `sysctl -p`.

#### DNS Bit

The basic DNS install is uneventful: install the daemons and *make sure* to
lock down the recursive resolver to your network. I'm assuming that you
want to serve all of the hypothetical 192.168.0.0/24 block.

    apt-get install bind9
    vi /etc/bind/named.conf.options
    # add the line
    #      allow-query-cache { 192.168.0.0/24; }; 
    # inside the options stanza
    /etc/init.d/bind9 restart

#### BGP Bit

We're going to use [exabgp][3] by [Thomas Mangin][5] on the DNS server side -
it's a pure-Python BGP implementation that doesn't depend on too much. On the
router we're going to use [Quagga][6], though the syntax should basically be
identical for IOS.

It's not packaged *for Lucid*, so we'll install using `pip`.

    apt-get install python-pip
    pip install exabgp

That puts it in `/usr/local/bin/exabgp`, we'll need to supply our own config
file and be responsible for launching it. Here's an example `upstart` job:

{% codeblock lang:bash %}
start on runlevel [2345]
stop on runlevel [!2345]
respawn
exec /usr/local/bin/exabgp /etc/bgp.conf
{% endcodeblock %}

*Update:* ExaBGP is, infact, packaged in Oneiric and Precise, though it's the
previous stable release.

<blockquote class="twitter-tweet"><p>Ubuntu users will find ExaBGP on Oneiric <a href="http://t.co/3pXamdn3" title="http://packages.ubuntu.com/hu/oneiric/net/exabgp">packages.ubuntu.com/hu/oneiric/net…</a></p>&mdash; ExaBGP (@exabgp) <a href="https://twitter.com/exabgp/status/148911146781523968" data-datetime="2011-12-19T23:42:38+00:00">December 19, 2011</a></blockquote>
<script src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Our upstream router advertises its connected routes into OSPF area 0. Because
it has 192.168.0.1/29 as an interface it will advertise this whole network, and we
won't need to redistribute BGP into OSPF. Something like the below Quagga
config will do. This is not a complete Quagga config, you should follow another
guide to get basic routing up and running.

{% codeblock %}
router bgp 64512
 bgp router-id 192.168.0.1
 neighbor 192.168.0.2 remote-as 64512
 neighbor 192.168.0.2 timers 5 10
 neighbor 192.168.0.3 remote-as 64512
 neighbor 192.168.0.3 timers 5 10 
!
router ospf
 ospf router-id 192.168.0.9
 redistribute connected metric 60
 network 192.168.0.8/29 area 0.0.0.0
!
{% endcodeblock %}

AS numbers above 64511 are reserved for private use, so you can use this freely.

This will advertise 192.168.0.0/29 into area 0 unconditionally. We have set up
both of the 'real' IPs as iBGP peers, and set very short values for the
keep-alive and hold-time timers. We will advertise ourselves every five seconds,
and if we don't hear from `exabgp` every 10 seconds we will consider that
path to be dead.

The defaults are 60 and 180 seconds respectively, but that would lead to a
noticable outage, though of course YMMV.

For the `exabgp` side, use a config like this for Machine A:

{% codeblock %}
neighbor 192.168.0.2 {
    router-id 192.168.0.2;
    local-address 192.168.0.2;
    local-as 64512;
    peer-as 64512;
    hold-time 5;
    static {
        route 192.168.0.4/32 next-hop 192.168.0.2 local-preference 200;
        route 192.168.0.5/32 next-hop 192.168.0.2 local-preference 150;
    }
}
{% endcodeblock %}

And this for Machine B:

{% codeblock %}
neighbor 192.168.0.3 {
    router-id 192.168.0.3;
    local-address 192.168.0.3;
    local-as 64512;
    peer-as 64512;
    hold-time 5;
    static {
        route 192.168.0.4/32 next-hop 192.168.0.3 local-preference 150;
        route 192.168.0.5/32 next-hop 192.168.0.3 local-preference 200;
    }
}
{% endcodeblock %}

We're not learning any routes, we're exclusively advertising routes. Machine A
will advertise both routes, with a higher priority for the first VIP, and
Machine B will do the same with a higher priority for the second VIP. Local
preference seems to be more effective than weight or metrics for bullying iBGP
into doing what you want.

On each machine, start the BGP process and watch tcpdump while you ping the
virtual IPs.

    start bgp
    tcpdump -ni any icmp
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on any, link-type LINUX_SLL (Linux cooked), capture size 96 bytes
    23:44:59.389054 IP 192.168.0.10 > 192.168.0.4: ICMP echo request, id 40574, seq 1, length 64
    23:44:59.389256 IP 192.168.0.4 > 192.168.0.10: ICMP echo reply, id 40574, seq 1, length 64

You should see the first VIP pings appear on Machine A, and the second VIP
pings on Machine B. Keep those pings going.

By stopping the BGP service on either box you should see the hand-over happen
with no missed pings. If you force a failure of one of the machines, it will be
within seconds:

    iptables -I INPUT -i eth0 -j DROP

then:

    PING 91.208.170.131 (91.208.170.131) 56(84) bytes of data.
    64 bytes from 91.208.170.131: icmp_seq=1 ttl=63 time=1.02 ms
    64 bytes from 91.208.170.131: icmp_seq=2 ttl=63 time=0.890 ms
    64 bytes from 91.208.170.131: icmp_seq=3 ttl=63 time=4.23 ms
    [ Four missed pings ]
    64 bytes from 91.208.170.131: icmp_seq=8 ttl=63 time=0.956 ms
    64 bytes from 91.208.170.131: icmp_seq=9 ttl=63 time=1.94 ms

Hopefully this has been instructive. If you have any comments, please direct
them to my Twitter account, [@insom][7].

[1]: http://packetlife.net/blog/2009/jul/6/remotely-triggered-black-hole-rtbh-routing/
[2]: http://www.linux-ha.org/wiki/Main_Page
[3]: http://code.google.com/p/exabgp/
[4]: http://www.iweb.co.uk/
[5]: http://thomas.mangin.com/
[6]: http://www.quagga.net/
[7]: http://twitter.com/insom
