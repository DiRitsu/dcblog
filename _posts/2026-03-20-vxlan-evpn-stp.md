---
layout: post
title: "vxlan-evpn-stp"
date: 2026-02-20
---


# VXLAN/EVPN fabrics and loop avoidance

Hello.

I want to start this blog as a reminder to myself of what problems and solutions were encountered during some of the worst searcing and testing sessions. And what can be better than his majesty broadcast storm.  

For the external switches EVPN/VXLAN fabric usually looks like a one big L3 switch. BGW can use L3 external ports, but standard core-to-fabric connection? There is a standard SVI way, redundant connections, but the problem:

Fabric will happily create a loop.

STP if not disabled already has different switch ID on different fabric switches. so no ports are blocked. BPDUs are not transferred via fabric so there is no way core switch knows that there is a loop.

This is not a new theme as you can see it in the useful [ipspace blog](https://blog.ipspace.net/2019/02/loop-avoidance-in-vxlan-networks/).

There are some solutions so lets dive in:

## Link aggregation

First and the most obvious is to aggregate links from external switch to fabric. There are 2 widely implemented variants:

### EVPN-MH

Uses EVPN route types 1 and 4 (I hope to write post about them later, as I previously didn't understand why we need two types for essentially the same purpose). Leaves coordinate which ones are connected to the same external L2 domain (ES, Ethernet Segment) and choose who is gonna send the BUM traffic. LACP ID is configured and external switch sees fabric swithes as one. Also you actually can span same ES to more than 2 Leaves (please read vendor docs as some do not support it).

There are some caveats:

#### 1\. Core isolation

In case of one Leaf losing connetion to all Spines, half of the traffic from external switch is blackholed so you should implement core isolation prevention.

Some reading on that: 

* [Cisco](https://www.ciscolive.com/c/dam/r/ciscolive/global-event/docs/2025/pdf/TACENT-2026.pdf)
* [Juniper](https://www.juniper.net/documentation/us/en/software/junos/evpn/topics/concept/evpn-vxlan-core-isolation-disabling.html)

#### 2\. Vendor support

Some vendors ignored EVPN-MH for a long time and some OS has no support for route types 1 and 4. Cisco for example.

#### 3\. DCI

In case of DCI swithes that also play the BGW role (not really uncommon in small DC) there is a problem in case of eBGP AS design:

-- If you use same AS on DCI leaves they will drop each other routes so EVPN-MH is impossible (without allow-as but this creates even more problems)

-- If you use different AS numbers there is a possibility of questionable pathing in some cases. Use route-filters or just don't use EVPN-MH on DCI.

Some reading on that: 
* [Cumulus](https://docs.nvidia.com/networking-ethernet-software/guides/Data-Center-Interconnect-Reference-Design-Guide/Extending-Fabric/)

### MLAG/MC-LAG

This method uses familiar vPC pairs. Nothing really new to say, if you are familiar with vPC.

VPC obligatory read: [vPC_design_guide](https://www.cisco.com/c/dam/en/us/td/docs/switches/datacenter/sw/design/vpc_design/vpc_best_practices_design_guide.pdf)  
EVPN addtional read: [ciscolive](https://www.ciscolive.com/c/dam/r/ciscolive/emea/docs/2025/pdf/BRKDCN-2912.pdf)

There are some caveats:

#### 1\. vPC implementation

All the standard design restrictions are there. Some vendors implementations are questionable (Cumulus uses peer-keepalive not as keepalive but as reserve so GL with 200g peer-link failure into 1g copper as reserve peer-link for those who don't expect this). Loop avoidance mechanism is still there so you have to account for that in VXLAN/EVPN environment.

#### 2\. Peer-link traffic

As vPC uses virtual IP as a VTEP IP in case where most the connections to the servers are non-aggregated (orphans). Route type 2 is advertised bt VIP, not PIP even in case of orphans. As example - some virtualisation solutions recommend using load-balancing without aggregation. So in this case half of the traffic is balanced to the wrong switch in pair, which in extreme case can overload peer-link.

## Loop avoidance

Since fabric is an L2 domain, it's good to have loop protection mechanisms anyway. Even with agregation there is a possibility that some switch will be randomly connected to 2 Leaves.

Cisco implemented beacons which are pretty much resembling BPDUs.
Read: [cisco](https://blogs.cisco.com/datacenter/detecting-and-mitigating-loops-in-vxlan-networks)

Arista has "super roor" which is pretty much all 0s ID and prio - thus fabric is always root and all the external swithes see Leaves like one switch.
Read: [arista](https://www.arista.com/assets/data/pdf/Whitepapers/EVPN-DC-Multihoming-for-Resiliency-WP.pdf)

And probably some more that I missed. Please write an email to me if it is the case.

## Conclusion

Always think through the L2 connections to the fabric in advance.   
EVPN-MH is more flexible but has to be accounted for in the design, especially BGP ASNs.  
Read vendor docs as they usually have preffered tested solution.