---
layout: post
title: Network shaping with ERL
description: "Network shaping with ERL"
modified: 
tags: [Ubiquiti, Wireless, Routing]
image:
  feature:
  credit:
  creditlink:
comments: true
share: true
---

I was recently asked to help one of our elderly neighbours get access to Skype via iPad to keep in touch with a very spread out family.

Unfortunately, for the time being our ADSL2+ bandwidth is very constrained with about 5Mbit down and 1Mbit up. Naturally you can see how an existing network of consumers might make sharing this link difficult.

Enter my recent WRT54GL replacement, the Ubiquiti [Edge Router Lite](http://www.ubnt.com/edgemax). I am the first to admit that the CLI at first was very off putting coming from the familiar environment of OpenWRT. However, the lower power consumption, price point and possibility of getting high speed inter-vlan routing made it an attractive choice.

## The network

I really enjoy simplicity of the fantastic [asciiflow.com](http://asciiflow.com/). So here you can enjoy a network diagram in ascii!

~~~
+-----------------+         XXXX  Wireless                             
| Guest Client    |                                                    
| 10.0.150.xxx/24 |         +--+  Wired                                
|                 |                                                    
+-----------------+                                                    
        X                                                              
        X                                EdgeRouter Lite               
        X                 +-------------------------------------------+
        X                 |                                           |
+-------------+---+       | +---------------+       +---------------+ |
| Asus WL500GP-V2 |       | | Guest Network |       | PPPOE Bridge  | |
| L2 Bridge       +---------+ 10.0.150.1/24 |       | 10.0.109.1/29 | |
| OpenWRT         |       | | bond0.150     |       | eth1          | |
+-----------------+       | +---------------+       +---------------+ |
                          |                                           |
                          +-------------------------------------------+
~~~

The network diagram does not show the other networks the ERL is managing but you get the picture. Since I am ultimately greedy, I have NOT configured any shaping on the other networks. The goal with this exercise is to ensure that the neighbors cant give me any surprises by hammering the available bandwidth.

We set out with the goal of:

* Downstream speeds shaped to 1Mbit
* Upstream speeds shaped to 256Kbit
* Guests must use a different external IP for NAT
* No transfer limits or other restrictions (Only a WPA2 PSK)

## The configuration

First we need to establish our traffic policies.

~~~
# show traffic-policy
 rate-control guestnet-ratecontrol {
     bandwidth 1mbit
     burst 15k
     latency 50ms
 }
 rate-control guestnet-out-ratecontrol {
     bandwidth 256kbit
     burst 15k
     latency 50ms
 }
~~~

Next, we need to implement a workaround to apply traffic policy to an outbound interface. We achieve this using the IFB interfaces to redirect our traffic a second time (At these speeds I have not seen a particular overhead for this double handling of traffic).

~~~
# show interfaces input ifb2
 traffic-policy {
     out guestnet-out-ratecontrol
 }
~~~

Finally, we assign the inbound policy to the guest interface and apply our traffic redirection.

~~~
# show interfaces bonding bond0 vif 150
 address 10.0.150.1/24
 description OPENWRT
 firewall {
     in {
         name OPENWRT_IN
     }
     local {
         name OPENWRT_LOCAL
     }
 }
 redirect ifb2
 traffic-policy {
     out guestnet-ratecontrol
 }
~~~

There is vastly more detail available on the Ubiquiti wiki if your heading down the QOS or rate limiting paths. For me, the above works like a charm!

* [Quality of Service](http://wiki.ubnt.com/Quality_of_Service_(QoS))
* [Traffic Policies](http://wiki.ubnt.com/Examples_of_Traffic_Policies)

## Results

The outcome as measured by Speedtest.net is right on the spot.

*Unrestricted*

![](http://www.speedtest.net/iphone/845446893.png)

*Guest*

![](http://www.speedtest.net/iphone/845446045.png)
