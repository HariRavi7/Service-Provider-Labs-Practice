# Troubleshooting log

Things that went wrong while I built and tested this lab, and what fixed them.

## Static route rejected with "Inconsistent address and mask"

I typed `ip route vrf A 10.1.2.1 255.255.255.0 192.168.3.2` and the router refused it. I had used the CE's host address (10.1.2.1) instead of the network. With a /24 mask the last octet has to be 0, so the right line is `ip route vrf A 10.1.2.0 255.255.255.0 192.168.3.2`. My first attempt had no mask at all and gave "Incomplete command".

## BGP session was up but no routes came across

`show bgp vpnv4 unicast all summary` showed the neighbour established, but the prefix count was 0 and the BGP table version was still 1. Plain `show bgp` printed nothing either.

The cause was where I had put `redistribute connected` and `redistribute static`. They were under the global `address-family ipv4`, and I had no `address-family ipv4 vrf A` or `vrf B` at all. So the session between the PEs was fine, but nothing from the VRFs was ever handed to BGP. Once I added the redistribute lines under each VRF family on both PEs, the prefix count went to 4.

## Tests from the PE showed only stars

I ran `traceroute 10.1.2.1 source 1.1.1.1` on PE1 and got nothing but `* * *`, even though CE to CE traffic worked. I had left out the `vrf` keyword, so the router looked in the global routing table, which has no customer routes. From a PE the test has to name the VRF and use a source address from inside it, for example `ping vrf A 10.1.2.1 source FastEthernet0/1`. Testing from the CE works too.

## LDP broken on one link, but ping still worked

I removed `mpls ip` from P2's Fa0/1, the link to PE2. Ping and traceroute from PE1 to 2.2.2.2 kept working, because OSPF still forwards plain IP. The sign that something was wrong was in the forwarding table on P2, where the entry for 2.2.2.2 changed from `Pop Label` to `No Label`. It taught me not to trust ping alone for MPLS. Check the LDP and forwarding tables and a traceroute with labels.

## Route target mismatch

I changed the import RT in VRF A on PE1 to see what a mismatch looks like. My first attempt used the wrong values (100:x instead of 65000:x), so it only added an extra import and nothing broke. When I did it properly, the VRF A entry for 10.1.2.0/24 vanished from `show bgp vpnv4 unicast all 10.1.2.0/24` and only the VRF B entry was left.

I had expected the route to stay in the BGP table without being installed in the VRF. In fact PE1 throws away VPNv4 routes whose RT doesn't match any import list on the router, so the route was gone completely while the session stayed up. The quickest check is `show ip vrf detail` on both PEs, because the export RT on the sender has to appear in the import list on the receiver. Putting the original import back fixed it.

## How I debug it now

I go from the bottom up and stop at the first thing that fails: OSPF route to the PE loopback, then LDP neighbour, then bindings and the forwarding table, then the VRF and BGP side (routes, RTs), and last the labels in a traceroute. The full list is in `notes/mpls-l3vpn-debug-commands.md`.
