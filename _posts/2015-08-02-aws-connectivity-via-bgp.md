---
layout: post
title: AWS outbound connectivity via BGP
description: "A write up on my spike to test using multiple BGP routers for a better outbound connectivity solution in AWS"
modified: 
tags: [Routing, AWS, BGP, NAT, VPC, Private]
image:
  feature:
  credit:
  creditlink:
comments: true
share: true
---
Ahoy!

Today I spent some time hacking on AWS after having been faced with the issues of VPC private connectivity in my [day job](https://careers.smartrecruiters.com/Atlassian/).

Its only a quick post today but the *tldr* is that, yes using either Amazon Direct Connect's or AWS VPN service will enable you to shift the outbound connectivity away from AWS itself and back to your corporate routers. The code to have a play with this approach yourself is at https://github.com/Kahn/aws-outbound-bgp

When your considering this there is a few quick pros and cons.

**Pros:**
* Failover is incredibly quick compared to solutions based on auto-scaling groups. BGP will is configured with 30 second timers by default by Amazon and a dead route will be removed even quicker.
* You can work around the single route only options presented to you via the AWS console.
* You can scale the solution out past N+1 if your so inclined
* Your VPC stays more secure by sending your internet bound traffic back to your existing corporate environment ISP's (Your watching *those* links right?) rather than the nearest IGW

**Cons:**
* You loose literally all of the network elasticity that using AWS affords you by congesting your existing links with AWS originated traffic
* Your now going to have to get some more of that PI space to address these hosts as your not going to be able to have Amazon EIP's easily either.

In short I think this solution will **is** appealing to bigger enterprises who can easily handle the sort of transit bandwidth they require. Being able to simplfy routing across their AWS footprint (think multiple accounts per organization) does make the job of a security team and network team much easier.

Meanwhile, I am sure a small startup just wanting to avoid getting their office 4G link DOS'ed by server traffic will really not care at all about these options and just stick with NAT or HTTP Proxies.

That is all I have time for now but I hope this can help next time your considering how to get traffic out of your private VPCs. If you have any questions please feel free to use the comments or [hit me up](http://www.cycloptivity.net/about/#Contact) online.
