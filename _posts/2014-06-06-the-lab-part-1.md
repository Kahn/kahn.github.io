---
layout: post
title: The lab - part 1
description: "Home network lab part 1"
modified: 
tags: [Ubiquiti, Wireless, Routing, ERL, DN42]
image:
  feature:
  credit:
  creditlink:
comments: true
share: true
---

I have recently stumbled across [DN42](http://dn42.net/) and have been fairly excited to transition my networking know how from layer 3 meshing via OLSR (back in the NTFreenet days) to now understanding more of complex enterprise networks.

What better way to learn about such things than to:

1. Install a complex enterprise network in your own home
1. Make sure you have outage sensitive customers on the network. Any Wife, kids or friends will usually do

And so with this in place I introduce my humble home lab.

![](/images/20140604-homelab.jpg)

## Details

The network is so far comprised of:

* 2x Ubiquiti ERL
* 2x Ubiquiti Rocket M2
* 1x Cisco SG200-26
* 1x Linksys SLM2008
* 1x Fritzbox 7270 + Fritzfon DECT handset
* 1x Asus WL500GPV2
* 1x Netcomm NB6PLUS4
* 1x Dovado GO + Seirra Wireless 320U

There are also some servers

* 1x SmartOS host - 2x gbit nics plus 1x IPMI
* 1x QNAP TS-410 - 2x gbit nics

And finally some 11 clients between wired clients, mobile clients and media devices.

## Layer 1 and 2

Things a pretty boring here. You can see most of the detail already in the picture at the start of this post. None the less here is the documentation for my benifit.

I have not really found a cleaner way for this short of tables so you will have to suffer through or suggest a better alternative!

### Bedroom

**NB6PLUS4**

| Interface | Endpoint |
|---|---|
|ADSL | Filter |
|Port 1 | RocketM2-eth0 |

### Office

**ERL-01**

| Interface | Endpoint | VLANS |
|---|---|---|
| eth0 | SG200-26-ge21 | lag1 |
| eth1 | SG200-26-ge22 | lag1 |
| eth2 | ERL-02-eth2 | NA |

**ERL-02**

| Interface | Endpoint | VLANS |
|---|---|---|
| eth0 | SG200-26-ge23 | lag1 |
| eth1 | SG200-26-ge24 | lag1 |
| eth2 | ERL-01-eth2 | NA |

**SG200-26**

| Interface | Endpoint | VLANS |
|---|---|---|
| ge1 | Desktop-eth0 | 110UP |
| ge2 | Desktop-eth0 | 110UP |
| ge3 | NA | 110UP |
| ge4 | NA | 1UP |
| ge5 | NA | 1UP |
| ge6 | NA | 1UP |
| ge7 | NA | 1UP |
| ge8 | NA | 1UP |
| ge9 | NA | 1UP |
| ge10 | SLM2008-ge1 | 1UP,130T,140T |
| ge11 | Fritz-lan1 | 140UP |
| ge12 | WL500GPv2-eth0 | 150UP |
| ge13 | NA | 1UP |
| ge14 | NA | 1UP |
| ge15 | NA | 1UP |
| ge16 | NA | 1UP |
| ge17 | NA | 1UP |
| ge18 | Server-ipmi0 | 130UP |
| ge19 | Server-eth0 | 120UP |
| ge20 | Server-eth1 | 130UP |
| ge21 | ERL-01-eth0 | lag1 |
| ge22 | ERL-01-eth1 | lag1 |
| ge23 | ERL-02-eth0 | lag2 |
| ge24 | ERL-02-eth1 | lag2 |
| ge25 | RocketM2-eth0 | 191UP |
| ge26 | Dovado-lan0 | 192UP |
| lag1 | ERL-01-bond0 | 1UP,120T,150T,161T,191T,192T |
| lag2 | ERL-02-bond0 | 1UP,110T,130T,140T,162T,191T,192T |

### Lounge

**SLM2008**

| Interface | Endpoint | VLANS |
|---|---|---|
| ge1 | SG200-26-ge10 | 1UP,130T,140T |
| ge2 | Wired Client | 140UP |
| ge3 | Wired Client | 140UP |
| ge4 | Wired Client | 140UP |
| ge5 | Wired Client | 140UP |
| ge6 | NA | 140UP |
| ge7 | TS410-eth0 | lag1 |
| ge8 | TS410-eth1 | lag1 |
| lag1 | TS410-bond0 | 140UP |

In the next installment we will describe the layer 3 networks!
