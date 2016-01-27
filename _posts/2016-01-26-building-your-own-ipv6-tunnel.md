---
layout: post
title: Building your own IPv6 Tunnelbroker
description: "Building your own IPv6 Tunnelbroker"
modified:
tags: [Routing, IPv6, OpenVPN]
image:
  feature:
  credit:
  creditlink:
comments: true
share: true
---
Ahoy!

A few weeks ago I decided that it was time to take the plunge and move over to NBNco's new FTTB offering. Sadly this meant I had to leave behind my native dual-stack with [Internode](http://www.internode.on.net/about/our_network/ipv6/) but the potential to take my 6/0.7Mbit ADSL2+ and turn it into 100/40Mbit was too appealing.

So here we are, as promised my link syncs are 107/44Mbit, Yeehah! On the downsides;

* iiNet hands out DHCP leases for VDSL, no more static IP's. (Yes, I locked myself out [many](https://github.com/Kahn/scripts/blob/master/ddns-firewall.sh) [times](https://github.com/Kahn/scripts/blob/master/ddns-firewall-centos.sh))
* Bridging PPPoE to my Edgerouter is not an option since now I need the iiNet CPE to manage the SIP ATA
* My native dual-stack is gone, no more IPv6 in the house. Good-bye sweet DHCPv6-PD prince!

### Building a tunnelbroker

Despite my 1337 [Hurricane Electric IPv6 certificate](https://ipv6.he.net/certification/scoresheet.php?pass_name=cycloptivity) I figured I have a bit of a way to go to truly grok IPv6. So while the idea of *just* configuring the HE.net tunnelbroker was tempting, I, like always, made life hard for myself instead! Lets dig in!

I use [Vultr](http://www.vultr.com/?ref=6813536) to host a KVM instances nice and close to me in Australia, since international transit tends to suck . Incidentally their IPv6 network has left beta and has been maturing over the last 12 months. Great! I'll just route some of that address space from the instance to my home router (a Ubiquiti [Edgerouter](https://www.ubnt.com/edgemax/edgerouter-lite/) lite) and I will be back on the IPv6 Internet.

#### The tunnel


                            .----------------------------.                        .----------------------------.
                            |           syd01            |                        |           erl01            |
                            |----------------------------|                        |----------------------------|
                            | eth0:   198.51.100.56/23   |     OpenVPN - vtun0    | eth1:   203.0.113.1/24     |
      .----------.          |         2001:DB8:5400::/64 |.----------------------.|         2001:DB8:2000::/67 |
      | Internet |----------|                            ||       SIT sit1       ||                            |
      '----------'          | tun0:   192.0.2.1          |'----------------------'| tun0:   192.0.2.2          |
                            |                            |                        |                            |
                            | sit1:   2001:DB8::1/66     |                        | sit1:   2001:DB8::2/66     |
                            '----------------------------'                        '----------------------------'


Let's talk about this diagram quickly. **syd01** is the Vultr instance and **erl01** is my home router. Not in the picture is the upstream hardware doing IPv4 NAT on eth0 for my FTTB connection.

In order to get IPv6 routes back into my home network I needed to use a 6in4 tunnel using **SIT** (rfc4213), however there is no mechanisms in the SIT protocol to provide privacy or authentication. So, we turn to our old friend OpenVPN and configure a layer 3 site-to-site link **tun0** then route the SIT traffic across tun0.

At this point it seems pretty simple right?

#### The details

Wrong. Or at least I was. There is a bunch of dirty hacks involved in this. Lets just jot them down quickly before we get into configurations.

1. Splitting a /64 is considering *bad practice*. This breaks SLAAC and requires the implementation of stateful routing.

      From my basic understanding the reasoning here is that IPv6 addresses are 128bit, with the leading most significant 64bits making up the network address and the least significant 64bits making up the host address. Therefore protocols like SLAAC can safely assume that the *smallest* IPv6 subnet can be a /64 and any address space further down *obviously* belongs to that network.

1. CentOS 6 and specifically the Linux kernel 2.6.32.* is way too old, just hurry up and deploy CentOS 7.
1. Why CentOS 7? The Vultr instance is getting its eth0 addresses via SLAAC. The issue related to kernel version is support for accepting Router Advertisements at the same time as IPv6 forwarding is enabled. (Broken prior to 2.6.37)
1. Continuing on about the *on-link* differences. We implement a real hack on *syd01* where we need to proxy Neighbour Discovery Protocol (think IPv4 ARP but via ICMPv6). This method has been panned for being "the new NAT", but in this case we need to split the /64 so there are no other options left. More on this later.

Ok, with that list of pain out of the way. Lets actually unpack what this solution looked like.

#### What did we actually deliver?

Not much really. The tunnels performance is pretty bad. I can only get about 10Mbit via IPv6, but as a proof of concept and a learning exercise I am considering that a first pass success. Hosts on my local home network have global IPv6 routing from 2001:DB8:2000::/67 as well as existing IPv4 connectivty.


##### SYD01 configuration

A few moving parts to capture here on CentOS 7.

**Kernel Configuration**

```
/etc/sysctl.conf
...
# Enable global forwarding
net.ipv6.conf.all.forwarding = 1
# Accept IPv6 RA, despite forwarding enabled
net.ipv6.conf.all.accept_ra = 2
# Proxy neighbour discovery protocol to downstream routers
net.ipv6.conf.all.proxy_ndp = 1
```

**Network Configuration**

```
/etc/sysconfig/network-scripts/ifcfg-eth0
TYPE="Ethernet"
BOOTPROTO="dhcp"
DEFROUTE="yes"
PEERDNS="yes"
PEERROUTES="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_PEERDNS="yes"
IPV6_PEERROUTES="yes"
IPV6_FAILURE_FATAL="no"
NAME="eth0"
DEVICE="eth0"
ONBOOT="yes"

/etc/sysconfig/network-scripts/ifcfg-sit1
DEVICE=sit1
BOOTPROTO=none
ONBOOT=yes
IPV6INIT=yes
IPV6_TUNNELNAME=sit1
IPV6TUNNELIPV4=192.0.2.2
IPV6TUNNELIPV4LOCAL=192.0.2.1
IPV6ADDR=2001:DB8::1/66
IPV6_MTU=1280
TYPE=sit
```

**OpenVPN**

Nothing to complicated here. A site-to-site UDP tunnel using a static key (no PFS for you).

```
mode p2p
rport 1194
lport 1194
remote home.example.org
proto udp
dev-type tun
dev vtun0
secret /etc/openvpn/home.key
persist-key
persist-tun
ifconfig 192.0.2.1 192.0.2.2
float
script-security 2
status /var/log/openvpn_status_home.log
log-append /var/log/openvpn_home.log
keepalive 10 60
cipher AES-128-CBC
auth SHA256
user nobody
group nobody
```

**FirewallD**

There is a bit of hackery here because FirewallD doesn't really support complex (?) use cases like routing, good thing its still just *iptables*.

```
firewall-cmd --direct --permanent --add-rule ipv6 filter FORWARD 0 -i sit1 -j ACCEPT
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 -i tun0 -p 41 -j ACCEPT
firewall-cmd --complete-reload
```

This does still need to be cleaned up a little bit but should give you the right direction.

**ndppd**

Here is the real *hacks* / *magic* depending on your perspective.

Lead down the right path by Sean Groarke and his write up at [https://www.ipsidixit.net/2010/03/24/239/](https://www.ipsidixit.net/2010/03/24/239/) I got to the point where all my routing was working. I could ping interface-to-interface but when I tried to get from my home LAN out to the internet? Dropped by the remote end of sit1. What was going on?

      ip -6 neigh add proxy 2001:DB8::3 dev eth0

And suddenly my ping is working? Thanks Sean!

As it turns out the root cause here is that the SLAAC addressing being used between the Vultr upstream and the actual VM instance, so subsequently when the upstream router sends a neighbour discovery packet seeking *2001:DB8::3* there is no way to find the address on the L2 link on eth0. So invoke the Linux NDP proxy for *2001:DB8::3* and suddenly for each incoming discovery of *2001:DB8::3* on eth0 we will actually now forward the packet.

This solution works ok, but how do I deal with all the other clients in the house?

There are a couple of daemons that will manage this for you and is more or less the part where "we re-invented NAT". Now there is a spinning daemon which watches for NS messages and adds the appropriate NDP proxy entires. I do feel like this is a valid use case but it is also easy to see how **no**, actually a /64 is really the smallest IPv6 subnet.

* [ndppd](https://github.com/DanielAdolfsson/ndppd)
* [ndp6](https://github.com/npd6/npd6)

Somehow I ended up building **ndppd**, I have opened some work I want to do (get it into EPEL and support SystemD) but for now I have just started the daemon manually.

```
/etc/ndppd.conf
route-ttl 30000
proxy eth0 {
   router yes
   timeout 500
   ttl 30000
   rule 2001:DB8::/64 {
      static
   }
}

$ ndppd -d
```

Great! If you run a ping now from your home network out to an Internet address like *ipv6.google.com* chances are it should be working. If not *tcpdump -i sit1 icmp6* is your friend!

##### ERL01 configuration

On the Ubiquiti side things are pretty standard. These boxes rock!

**Inteface Configuration**

Even though we are using dhcpv6-server the Edgerouter must still run the router advertisement function. The difference to watch for here is setting the *managed-flag*, *other-config-flag* and *autonomous-flag* to ensure clients make use of dhcpv6-server's direction.

Is now a good time to bring up that your Android device won't work? - See http://www.techrepublic.com/article/androids-lack-of-dhcpv6-support-poses-security-and-ipv6-deployment-issues/

```
ethernet eth1 {
    address 192.168.1.1/24
    address 2001:DB8:2000:1/67
    duplex auto
    ipv6 {
        router-advert {
            managed-flag true
            other-config-flag true
            prefix 2001:DB8:2000:1/67 {
                autonomous-flag false
            }
            send-advert true
        }
    }
    speed auto
}
openvpn vtun0 {
    description syd1.example.org
    encryption aes128
    hash sha256
    local-address 192.0.2.2 {
    }
    local-port 1194
    mode site-to-site
    protocol udp
    remote-address 192.0.2.1
    remote-host syd1.example.org
    remote-port 1194
    shared-secret-key-file /config/auth/home.key
}
tunnel tun0 {
    address 2001:DB8::2/66
    description "IPv6 Tunnel"
    encapsulation sit
    local-ip 192.0.2.2
    mtu 1280
    remote-ip 192.0.2.1
}
```

**Service Configuration**

Since we have no SLAAC here the stateful dhcpv6-server needs to hand out name-servers and manage IP assignment within the allocated space 2001:DB8:2000::2 -> 2001:DB8:2000::1999.

```
dhcpv6-server {
    shared-network-name lan {
        subnet 2001:DB8:2000::/67 {
            address-range {
                prefix 2001:DB8:2000::/67 {
                }
                start 2001:DB8:2000::2 {
                    stop 2001:DB8:2000::1999
                }
            }
            name-server 2001:4860:4860::8888
            name-server 2001:4860:4860::4444
        }
    }
}
```

### Closing thoughts

* If your thinking hey this would be sweet too I don't want to use SixXS either! Ping Vultr. I raised support case **ZXQ-36YYP** asking for an additional /64 so I could just run *radvd*. If there is enough intrested hopefully we can see this become more of a reality.
* I really miss a decent Australian ISP. This excercise has really made me appreciate the elegance of a DHCPv6-PD tail assigning me a /56.
* Network operators need to give up on the idea that IP addresses are the "Golden Goose". Drip feeding out little /120 subnets to networks is incredibly frustrating and PLEASE instead consider monetizing *transit* proper by enabling DHCPv6-PD rather than thinking your $1 per IPv4 strategy makes sense for IPv6. (Props to Linode who anecdotally are onboard with /48's if you ask for it)
* Continuing on network operators, if you haven't seen it yet go watch Owen DeLong talk to [IPv6 enterprise addressing](https://www.youtube.com/watch?v=Q65QuB1CXis), then please come back and defend handing out really **really** tiny subnets.
