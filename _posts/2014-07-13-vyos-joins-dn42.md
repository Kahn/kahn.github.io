---
layout: post
title: VyOS joins DN42
description: "VyOS joins DN42"
modified: 
tags: [VyOS, Routing, DN42]
image:
  feature:
  credit:
  creditlink:
comments: true
share: true
---

Its not quite the next blog post that I promised but what the heck. Today we are talking about my recent experiences in peering VyOS into DN42 via OpenVPN.

This should help you to form site-to-site tunnels using OpenVPN!

## Generating the keys

First we need to generate a PSK to exchange with our peer. Also note that on VyOS you need to store keys in /config/auth/ or suffer key loss during image updates.

```
vyos@ext02:~$ generate openvpn key /config/auth/peer.key
```

After sharing the pre-shared key with your peer we move onto config.

## Tunnel configuration

Your peer will likely be running OpenVPN and actually using its config file syntax. Take this file for example.

```
mode p2p
rport 1194
lport 1194
remote 192.0.2.2
proto udp
dev-type tun
dev peer.example
encryption bf128
hash sha1
comp-lzo
secret /etc/openvpn/peer.key
user nobody
group nobody
persist-key
persist-tun
ifconfig 192.168.0.1 192.168.1.1
float
script-security 2
status /var/log/openvpn_status_peer.log
log-append /var/log/openvpn_peer.log
keepalive 10 60
```

Now we can convert to VyOS configuration statements.

```
set interfaces openvpn vtun1 encryption 'bf128'
set interfaces openvpn vtun1 hash 'sha1'
set interfaces openvpn vtun1 local-address '192.168.0.1'
set interfaces openvpn vtun1 local-host '192.0.2.1'
set interfaces openvpn vtun1 local-port '1194'
set interfaces openvpn vtun1 mode 'site-to-site'
set interfaces openvpn vtun1 openvpn-option 'comp-lzo'
set interfaces openvpn vtun1 remote-address '192.168.1.1'
set interfaces openvpn vtun1 remote-host '192.0.2.2'
set interfaces openvpn vtun1 remote-port '1194'
set interfaces openvpn vtun1 shared-secret-key-file '/config/auth/peer.key'
```

VyOS seems to expose most of the more arcane OpenVPN configuration statements via the ```set interfaces openvpn vtun1 openvpn-option``` command, so you should be able to match your peers configuration. By default VyOS will run the OpenVPN processes with a ```--ping 10 --ping-restart 60``` so you can omit that.

Once configured you can run ```show openvpn site-to-site status``` to view your tunnels!

I have now provisioned a dedicated VPS with [Binarylane](https://www.binarylane.com.au/) in the [NEXTDC B1](http://www.nextdc.com/data-centres/data-centre-locations/b1-brisbane-data-centre) facility. From here we have a good place for DN42 transit until I am eventually connected to the NBN. Those that are interested, please feel free to ```traceroute dn42.cycloptivity.net```.

Good luck and happy peering!
