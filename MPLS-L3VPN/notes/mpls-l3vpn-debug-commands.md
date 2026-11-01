---
tags: [mpls, l3vpn, troubleshooting, cisco, ios-xe, cheatsheet]
date: 2026-10-31
status: reference
related: [[MPLS Study - Blocks 1-3 Summary]], [[VRF]], [[BGP]], [[OSPF]]
---

# MPLS L3VPN: Debug Commands Cheat Sheet

Lab: c7200, IOS/IOS-XE syntax. PE1 1.1.1.1, P1 3.3.3.3, P2 4.4.4.4, PE2 2.2.2.2, AS 65000, VRF A (RD/RT 65000:1) and VRF B (65000:2). Customer LANs overlap (10.1.1.0/24 and 10.1.2.0/24).

**Method:** follow the route from the sending CE to the receiving CE. Stop at the first rung that fails. Rungs 1 to 3 are the core, 4 to 8 are the VPN, 9 is the proof.

## The ladder

### 1. IGP: can PEs reach each other's loopbacks?
```
show ip route 2.2.2.2
show ip ospf neighbor
ping 2.2.2.2 source 1.1.1.1
```
Good: /32 route via OSPF, neighbours `FULL`, ping 5/5. No route here breaks everything above it.

### 2. LDP: are labels handed out for those loopbacks?
```
show mpls interfaces
show mpls ldp discovery
show mpls ldp neighbor
show mpls ldp bindings 2.2.2.2 32
show mpls forwarding-table 2.2.2.2 32
```
- `discovery` lists a peer but `neighbor` does not = TCP 646 problem (ACL, no route to peer router-id).
- `Pop Label` toward the egress PE = healthy PHP. `No Label` = next hop sent no binding = fault.

### 3. Transport path with labels
```
traceroute 2.2.2.2 source 1.1.1.1
```
Good: `[MPLS: Label N]` on the core hops, no label on the last hop (PHP). Ping alone proves nothing: plain IP still works with a broken label path.

### 4. VRF and PE-CE link
```
show vrf
show ip vrf detail A
show ip vrf interfaces
show ip interface FastEthernet0/1
show run | section vrf
```
Check: right VRF on the interface (name is case-sensitive), IP and mask present after `vrf forwarding`, interface `up/up`, RD and both RT lists as intended.

### 5. Does the PE learn the CE route?
```
show ip route vrf A
ping vrf A 192.168.1.2 source FastEthernet0/1
```
Good: `C` for the PE-CE link, `S` (or protocol route) for the customer LAN. A static route is **not installed** if its next hop is unreachable.

### 6. Did the route enter BGP?
```
show bgp vpnv4 unicast all
show bgp vpnv4 unicast rd 65000:1
show bgp vpnv4 unicast vrf A
```
Good: the prefix appears under `Route Distinguisher: 65000:1 (default for vrf A)`. Missing = no `redistribute` under `address-family ipv4 vrf A` (the "door").

### 7. PE-to-PE VPNv4 session
```
show bgp vpnv4 unicast all summary
show bgp vpnv4 unicast all neighbors 2.2.2.2
show bgp vpnv4 unicast all neighbors 2.2.2.2 advertised-routes
show bgp vpnv4 unicast all neighbors 2.2.2.2 routes
show run | section bgp
```
Good: neighbour `Established` with a prefix count. In `neighbors` output, `Local host:` should be the loopback. Not established? Check loopback reachability, `update-source Loopback0` on **both** sides, `neighbor ... activate` under `address-family vpnv4`, `send-community extended`, same AS for iBGP.

### 8. Route targets: sent, received, imported
```
show bgp vpnv4 unicast all 10.1.2.0/24
show ip vrf detail A
show ip route vrf A
```
- The detail view shows `Extended Community: RT:...` on the route.
- Match that RT against the **import** list of the receiving VRF.
- Classic mismatch on a normal PE: the route **disappears from `show bgp vpnv4 unicast all` entirely**, because IOS discards VPNv4 routes whose RT matches no local import list (default route-target filter). The entry for that RD/prefix is simply gone (healthy entries say `table A` in the header). On a route reflector, or with the filter disabled, you instead see the route in the BGP table but not in the VRF.
- Prove it by comparing `show ip vrf detail` on both PEs: the sender's **export** RT must appear in the receiver's **import** list.
- After a change: `clear ip bgp vpnv4 unicast * soft` (use `soft in` / `soft out` for one direction). Avoid the hard `clear ip bgp *`.

### 9. Labels and the data plane
```
show ip cef vrf A 10.1.2.0 255.255.255.0 detail
show mpls forwarding-table vrf A
show bgp vpnv4 unicast all labels
traceroute vrf A 192.168.3.2 source FastEthernet0/1
ping 10.1.2.1 source 10.1.1.1          ! from the CE
traceroute 10.1.2.1 source 10.1.1.1    ! from the CE
```
- `show ip cef ... detail` shows the **push**: `recursive via <PE loopback> label <VPN>` then `nexthop ... label <transport>`.
- `show mpls forwarding-table vrf A` on a PE shows the VPN labels **that PE assigned**. The label it pushes for the remote site is in CEF, not here.
- The VPN label PE1 pushes must equal the Local Label on the remote PE for that prefix.
- `No Label` toward a CE is **normal** (CEs do not run MPLS). It is only a fault on core links.
- Traceroute from a CE: no label on hop 1 (arrives as plain IP), `transport/VPN` on the core hops, `VPN` only after PHP at the egress PE, no label on the last hop.
- `source` must be an address the router owns. A PE cannot source from a CE's loopback. Use `vrf` plus a PE-side interface.
- **On a PE, always include the VRF** (`ping vrf A ...`, `traceroute vrf A ...`). Without it the global table is searched, which holds no customer routes, so you get `* * *` even when the VPN is fine.

## Symptom to cause

| Symptom | Likely cause | Check |
|---|---|---|
| BGP stuck `Active` / `Idle` | No route to peer loopback, `update-source` on one side only, wrong AS, TCP 179 blocked | rung 1, rung 7 |
| Session up, no VPN routes at all | VPNv4 neighbour not activated, `send-community extended` missing | rung 7 |
| Session up, route missing on remote PE | No `redistribute` under `address-family ipv4 vrf X` on the originating PE | rung 6 |
| Route missing on the receiving PE (gone from the BGP table and the VRF), session up, sender still advertises it | RT mismatch: default RT filter discarded it | rung 8, compare `show ip vrf detail` on both PEs |
| Test from a PE shows `* * *` or fails, but CE-to-CE works | Test run without `vrf` (searches the global table) or source not in the VRF | use `ping vrf A ... source <PE-CE interface>` |
| Static route not in the VRF table | Next hop unreachable, interface down, wrong VRF on interface | rung 5 |
| `% Inconsistent address and mask` | Host address used with a /24 mask (use the network address) | config |
| CE cannot reach the other site | CE default route missing, or return route missing on the other side | CE `show ip route` |
| Ping works, VPN traffic fails | Label path broken (LDP down, `No Label` in core) | rungs 2 and 3 |
| One customer sees the other's routes | Shared RT between different customers | rung 8 |
| Same prefix lost or replaced | Same RD on different customers' VRFs | `show ip vrf detail` |

## Quick config reminders
- RD sits **above** the address family. RTs sit **inside** `address-family ipv4`.
- `vrf forwarding X` first, then `ip address` (it wipes the address).
- `update-source Loopback0` goes under `router bgp`, per neighbour, not inside the vpnv4 family.
- Two address families, two jobs: `ipv4 vrf X` puts VRF routes into BGP, `vpnv4` carries them between PEs.

## Notes
- Debugs (`debug ...`) are rarely needed, the show commands above find almost everything. If you use one in the lab, run `undebug all` afterwards.
- Labels change after reloads. Compare label numbers between routers in the same session, not between sessions.

## References
- RFC 4364: BGP/MPLS IP Virtual Private Networks (VPNs). https://www.rfc-editor.org/rfc/rfc4364
- RFC 4760: Multiprotocol Extensions for BGP-4. https://www.rfc-editor.org/rfc/rfc4760
- RFC 5036: LDP Specification. https://www.rfc-editor.org/rfc/rfc5036
- RFC 4950: ICMP Extensions for MPLS. https://www.rfc-editor.org/rfc/rfc4950
- Cisco, MPLS Label Distribution Protocol Configuration Guide, IOS XE Release 3S. https://www.cisco.com/en/US/docs/ios-xml/ios/mp_ldp/configuration/xe-3s/mp-ldp-xe-3s-book.html
