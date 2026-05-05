---
layout: post
title: "one-loop-story"
date: 2026-05-05
---

# The story of one ttl expired problem

For the last month I've been diving into kubernetes networking and it was fun and challenging. 
Installing nginx ingress into learning that it is not recommended anymore, talos problems with VMXNET (well it is actually flannel problem but talos uses flannel by default, and it is in instruction but was overlooked cause I rushed it) and learning about where exactly ip addresses are. All this stuff reminded me of one loop story.

The network in that story was pretty simple, we had 2 racks, 2 mlag ToR switches in each, vxlan/evpn fabric. Just your average small dc network nowadays, everyone wants vxlan/evpn even if there is no real reason to have one. So the server guys were eager to isolate themselves from the switches thus NSX was used. 

I understood nothing about NSX at that time but bgp is bgp so I thought it was pretty straightforward. Four nsx edge VMs, each one has bgp with each leaf. It is eBGP so half of the neighbours are down. Ss it hops to rack 2 switch from rack one switch and then has to go to nsx so ttl 1 doesn't work. I think nsx to switch worked as no "Forwarding" is used. As there is no routing, IRB does not gets in a way so TTL is not decreased as per RFC 2003/2473. But the traffic from switch 1 is processed by switch 2 (Should recheck in lab if IRB-originated traffic TTL is decreased. I remember that I had to do multihop-ttl on Cumulus switches, but nothing really about ttl on nsx. Hmm.) so ttl is decreased. It is described by architect as "highly available". If nsx edge vm relocates to another rack it establishes a neighbour connection to the local pair of ToRs. 

Well it works, but half of the connections are down. And it does not look on the monitoring. So the architect's idea is to allow ebgp multihop. It may (and of course will) lead to some pathing issues (traffic goes first to the pair in the other rack, then to the rack host actually is, so there is an amount of bandwidth used for no reason), but bandwidth is plenty at that moment and monitoring guys just straight up say they'll not allow "down" status bgp. Design is bad, there is suboptimal pathing but it should at least deliver traffic so it is acceptable “for now”. So it was agreed to go for this design for some time to please the monitoring team and then fix it with some correct one later. 

Counterproposal was to just pin VMs to racks, but it was not allowed at that time by server department as they didn't want to restrict VMs (and probably do anything whatsoever with this problem as they had their hands full with replication troubleshooting at that time) and at the same time they want the same amount of edges active even if some host goes down. Consistency. Predictability. Surely nothing would go wrong.

So ebgp multihop is enabled. And some networks behind nsx became unavailable. Troubleshooting shows ttl expired so goes the fun part about L2 and L3 and tunneling:

Fabric ofc uses symmetric IRB, so routing has to be done on the "out" leaf. Well for some reason (I think the timers, as after the classic OMNI tiebreaker order is timer then Router ID then Neighbor IP) at the time two leaf switches each had the best route through the nsx edge from the other rack. ECMP was not used for some reason and I think it is for the best or the debug would be probably harder.

So leaf 1 has the route through the edge in rack 2. Leaf 2 has a route through the edge in rack 1.

Traffic from outside goes through border leaf to leaf 1 as it advertises the routes that it received via bgp from the nsx. Leaf 1 knows that the address is behind edge 2 so well it has to forward traffic to edge 2. But edge 2 is not locally connected so traffic goes to leaf 2, which advertises via EVPN that it has the edge 2 MAC connected. Now for the best part. VXLAN outer header destination is leaf 2, everything is suboptimal (edge 1 is connected locally to leaf 1 and can forward traffic to the destination) but ok.

Now for the internal header. It has to have a real destination ip (some address behind nsx), what about a mac address? Fabric uses symmetric IRB, so routing does not happen on leaf 1 and it uses symmetric rules for l3 traffic - destination mac of the internal header is set as leaf 2 mac address. So leaf 2 gets the packet, strips vxlan and then sees that packet is his to route. Route as L3, not forward at L2. I guess you get the picture. Leaf 2 has a route through edge 1 which is connected to leaf 1. And we get ttl expired as leaf 1 and leaf 2 play beautiful symmetric ping pong with each other. 

Asymmetric would not have this problem but the design with multihop was just wrong on multiple levels. So vm pinning was used, useless neighbourships removed, some more edge VMs added and everything went well after. The end.
