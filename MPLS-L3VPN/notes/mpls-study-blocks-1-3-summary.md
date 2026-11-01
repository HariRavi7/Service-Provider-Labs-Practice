---
tags: [mpls, ldp, l3vpn, cisco, ios-xe, study]
date: 2026-10-29
status: in-progress
related: [[OSPF]], [[BGP]], [[VRF]]
---

# MPLS Study Notes: Blocks 1 to 3 (so far)

Lab platform: GNS3, c7200, FastEthernet only, IOS/IOS-XE syntax. Interview-level depth.

## 1. Core idea
- MPLS forwards on a short **label** instead of a destination-IP lookup at every hop.
- **P routers** forward on labels only, so they hold **no customer routes**. This is what makes it scale.
- Roles: **LSR** = any label-switching router (P). **LER** = edge router that pushes/pops labels (PE).
- MPLS is the forwarding method. **LDP is only one way to distribute labels.** Others: static, RSVP-TE, BGP, Segment Routing.

## 2. Control plane layers (the order matters)
1. **IGP (OSPF area 0)** carries the PE/P loopbacks (/32). No IGP route, no label.
2. **LDP** attaches labels to those IGP prefixes.
3. **MP-BGP VPNv4** (later, for L3VPN) carries customer routes + VPN labels between PE loopbacks.

## 3. LDP essentials
- **Discovery:** Hellos over **UDP 646**, multicast 224.0.0.2, out of every `mpls ip` interface.
- **Session:** **TCP 646** between the two routers' **LDP router-ids** (use Loopback0). Hellos can work while the TCP session fails.
- Router-id only needs to be unique **inside your own MPLS network**.
- **Binding** = (prefix, label). Labels are **locally significant**: each router picks its own.
- **LIB** = every binding heard from every neighbour. **LFIB** = what is actually used: the label from the **next hop chosen by the routing table**. The LIB offering several labels cannot cause a loop.
- **LDP-IGP sync** (`mpls ldp sync`): do not use a link for traffic until LDP is up on it.

## 4. Label operations
- **Push** (ingress PE), **Swap** (P), **Pop** (end of path).
- **PHP (penultimate hop popping):** the egress PE advertises **implicit-null** (label 3, shown as `Pop Label`) for its own loopback, so the router before it removes the label and the egress PE does one lookup instead of two.
- **`Pop Label`** = next hop sent implicit-null, pop on purpose. **`No Label`** = next hop sent **no binding at all**: a fault.
- Packet walk in the lab (labels will differ after reloads):
  `PE1 --push 17--> P1 --swap 17>17--> P2 --pop--> PE2 (plain IP)`

## 5. L3VPN preview
- Packet carries **two labels**: `[transport][VPN][customer IP]`. P routers read only the top one.
- **Transport label** (LDP) gets the packet to the egress PE. **VPN label** is chosen by the **egress PE** and tells it which VRF to look in.
- PE2 tells PE1 the VPN label over **iBGP VPNv4 between loopbacks**. P routers do not run BGP.
- The BGP session needs only **IP reachability** (TCP 179), not LDP. So routes can exchange while the label path is broken.
- **RD** makes overlapping customer prefixes unique (`100:1:10.1.1.0/24` vs `100:2:10.1.1.0/24`). It does **not** decide which VRFs import a route. That is the **RT**'s job (next tidbit).

## 6. Commands
```
! config
mpls label protocol ldp
mpls ldp router-id Loopback0 force
interface FastEthernet0/0
 mpls ip

! verify
show mpls interfaces
show mpls ldp discovery
show mpls ldp neighbor
show mpls ldp bindings <prefix> <mask>
show mpls forwarding-table <prefix> <mask>
traceroute <loopback> source <loopback>
```

## 7. Debug ladder (stop at the first failing rung)
1. IGP: `show ip route <PE loopback>`
2. Interface: `show mpls interfaces`
3. Hellos: `show mpls ldp discovery`
4. Session: `show mpls ldp neighbor`
5. Bindings: `show mpls ldp bindings`
6. Forwarding: `show mpls forwarding-table`
7. Data plane: `traceroute` (shows `[MPLS: Label N]` per hop)

Shortcut from a `No Label` symptom: go straight to rungs 4 and 5.

## 8. Gotchas worth remembering
- **Ping can succeed with a broken label path.** OSPF still forwards plain IP. Prove MPLS with the tables and traceroute, not ping.
- `discovery` listing a peer but `neighbor` not = TCP 646 problem (ACL, or no route to the peer's router-id).
- Fault drill done in lab: `no mpls ip` on P2 Fa0/1 turned `Pop Label` into `No Label` while ping kept working.

## 9. Lab reference
| Router | Loopback0 | Core links |
|---|---|---|
| PE1 | 1.1.1.1/32 | Fa0/0 10.0.0.1/30 to P1 |
| P1 | 3.3.3.3/32 | Fa0/0 10.0.0.2/30 (PE1), Fa0/1 10.0.0.5/30 (P2) |
| P2 | 4.4.4.4/32 | Fa0/0 10.0.0.6/30 (P1), Fa0/1 10.0.0.9/30 (PE2) |
| PE2 | 2.2.2.2/32 | Fa0/0 10.0.0.10/30 to P2 |

CE routers (CE-A1/A2, CE-B1/B2) and `PA-2FE-TX` in slot 1 of PE1/PE2 are added in the L3VPN block.

## 10. Still to cover
- RT (import/export) vs RD, VRF config on the PEs, PE-CE routing
- End-to-end L3VPN packet walk and troubleshooting
- L2VPN (AToM/VPLS) basics
- Mock interview + Anki deck

## References
- RFC 3031: Multiprotocol Label Switching Architecture. https://www.rfc-editor.org/rfc/rfc3031
- RFC 3032: MPLS Label Stack Encoding. https://www.rfc-editor.org/rfc/rfc3032
- RFC 5036: LDP Specification. https://www.rfc-editor.org/rfc/rfc5036
- RFC 5443: LDP IGP Synchronization. https://www.rfc-editor.org/rfc/rfc5443
- RFC 4364: BGP/MPLS IP Virtual Private Networks (VPNs). https://www.rfc-editor.org/rfc/rfc4364
- RFC 4760: Multiprotocol Extensions for BGP-4. https://www.rfc-editor.org/rfc/rfc4760
- RFC 4950: ICMP Extensions for MPLS (why traceroute shows labels). https://www.rfc-editor.org/rfc/rfc4950
- Cisco, MPLS Label Distribution Protocol Configuration Guide, IOS XE Release 3S. https://www.cisco.com/en/US/docs/ios-xml/ios/mp_ldp/configuration/xe-3s/mp-ldp-xe-3s-book.html
