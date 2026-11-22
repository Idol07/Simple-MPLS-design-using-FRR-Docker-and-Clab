# Simple-MPLS-design-using-FRR-Docker-and-Clab
A  simulation of a simple MPLS architecture using  FRR and dockers 

A compact service-provider MPLS lab. Two PE routers, two P routers and four customer routers run in Docker containers using FRRouting (FRR). The lab shows an MPLS core with LDP, redundant paths, and two isolated customer VPNs (CustA and CustB) that use overlapping IP ranges.

## Topology

!Screenshot 2026-11-23 at 10.07.31 am.png

| Role | Nodes |
| --- | --- |
| Provider edge (PE) | PE1, PE2 |
| Provider core (P) | P1 (primary path), P2 (backup path) |
| Customer edge (CE) | CE1A, CE2A (CustA); CE1B, CE2B (CustB) |

## Design

- **Underlay:** OSPF area 0 with /32 loopbacks. The P1 path has cost 10 and the P2 path has cost 100, so P1 is preferred and P2 is the backup.
- **Labels:** LDP distributes transport labels across the core.
- **Overlay:** PE1 and PE2 run an iBGP VPNv4 session between their loopbacks (no route reflector). P1 and P2 are BGP-free.
- **Customers:** Each customer has its own VRF with its own RD and RT. PE-CE routing uses eBGP.
- **Failure detection:** BFD (300 ms x 3) on core links.

## Addressing

| Device | Loopback / LAN | AS |
| --- | --- | --- |
| PE1 | 2.2.2.2/32 | 65000 |
| P1 | 1.1.1.1/32 | n/a |
| PE2 | 3.3.3.3/32 | 65000 |
| P2 | 4.4.4.4/32 | n/a |
| CE1A | 192.168.1.1/24 | 65010 |
| CE2A | 192.168.2.1/24 | 65005 |
| CE1B | 192.168.1.1/24 | 65015 |
| CE2B | 192.168.2.1/24 | 65020 |

| Link | Subnet | OSPF cost |
| --- | --- | --- |
| P1:eth1 :: PE1:eth1 | 10.0.0.4/30 | 10 |
| P1:eth2  :: PE2:eth1 | 10.0.0.0/30 | 10 |
| P1:eth3 :: P2:eth3 | 10.0.1.0/30 | 10 |
| P2:eth1 :: PE1:eth4 | 10.0.0.24/30 | 100 |
| P2:eth2 :: PE2:eth4 | 10.0.0.28/30 | 100 |
| PE1:eth2 :: CE1A:eth1 | 10.0.0.8/30 | eBGP |
| PE1:eth3 - CE1B:eth1 | 10.0.0.12/30 | eBGP |
| PE2:eth2 - CE2A:eth1 | 10.0.0.16/30 | eBGP |
| PE2:eth3 - CE2B:eth1 | 10.0.0.20/30 | eBGP |

| VRF | RD | RT | Kernel table | vrf id  |
| --- | --- | --- | --- | --- |
| CustA | 65000:10 | 65000:10 | 10 | 3 |
| CustB | 65000:20 | 65000:20 | 20 | 4 |

CustA and CustB both use 192.168.1.0/24 and 192.168.2.0/24. The RD keeps the routes unique and the RT keeps the customers separate.

## Requirements

- Linux host or VM (for example Ubuntu in VMware Fusion or UTM on a your windows or Mac). Docker Desktop's VM may not include the MPLS kernel modules.
- Docker and containerlab
- Kernel modules: `mpls_router`, `mpls_iptunnel`, `mpls_gso`, `vrf`

## Repository layout

```
text.├── mpls-core.clab.yml     # containerlab topology
├── daemons-p              # FRR daemons for P routers (no bgpd)
├── daemons-pe             # FRR daemons for PE routers (bgpd + bfdd)
├── daemons-ce             # FRR daemons for CE routers
├── pe1.conf  pe1-vrf.conf
├── pe2.conf  pe2-vrf.conf
├── p1.conf   p2.conf
├── ce1a.conf ce1b.conf ce2a.conf ce2b.conf
├── bfd-on.sh              # attaches BFD to OSPF on core links
└── docs/                  # screenshots and captures
```

## Quick start

```
bashsudo modprobe mpls_router mpls_iptunnel mpls_gso vrf
sudo containerlab deploy -t mpls.clab.yml
```

Open a router CLI:

```
dockerexec -it clab-mpls-pe1 vtysh
```

To tear down:

```powershell
sudo containerlab destroy -t mpls-core.clab.yml

sudo containerlab destroy -t mpls-core.clab.yml --cleanup
# to clean up your virtual env and  permanently deletes all generated lab directories and runtime data.
```

## Verification

On a PE (inside `vtysh`):

| Command | Expected result |
| --- | --- |
| `show ip ospf neighbor` | Full adjacencies with P1 and P2 |
| `show mpls ldp neighbor` | LDP sessions operational |
| `show mpls ldp binding` | Labels for the loopbacks |
| `show bgp ipv4 vpn summary` | PE-to-PE session Established |
| `show bgp ipv4 vpn` | VPN routes with RDs and labels |
| `show ip route vrf CustA` | CustA routes only |
| `show bfd peers` | Core BFD sessions up |

On P1 and P2, `show bgp summary` should show that BGP is not running.

End-to-end pings (source set to the customer LAN):

```bash
docker exec clab-mpls_core-ce1aping -c3 -I 192.168.1.1 192.168.2.1
docker exec clab-mpls_core-ce1bping -c3 -I 192.168.1.1 192.168.2.1
 
 # or directly check by going to bash 
```

Label capture while a ping runs:

```powershell
dockerexec clab-mpls_core-p1 tcpdump -nv -i eth1 mpls
# PE1 -> P1: transport + VPN labeldockerexec clab-mpls_core-p1 tcpdump -nv -i eth2 mpls# P1 -> PE2: VPN label only (after PHP)
```

## Failure tests

| Test | Command | What to observe |
| --- | --- | --- |
| Link down | `docker exec clab-mpls_core-pe1 ip link set eth1 down` | Traffic moves to P2; the transport label changes, the VPN label stays the same |
| Silent loss | `tc qdisc add dev eth1 root netem loss 100%` on both ends of the link | Slow recovery without BFD, fast with BFD |
| Restore | `ip link set eth1 up` / `tc qdisc del dev eth1 root` | Traffic returns to P1 (lower cos |

## Key findings

- The core carries two labels: an outer LDP transport label and an inner VPN label.
- After penultimate hop popping, the last P-to-PE link carries only the VPN label.
- P1 and P2 never see customer routes, so they need no BGP.
- During failover only the transport label changes, because the PE-to-PE VPN session stays up.
- Overlapping customer prefixes stay separate because of the RD and RT.
- Silent failures need BFD to converge quickly.

## Limitations

- Lab-grade timers and a single OSPF area.
- Two PEs only, so iBGP is a direct session with no route reflector.
- No traffic engineering or segment routing.

## Possible next steps

- Add a route reflector and a third PE.
- Add a script that checks OSPF, LDP, BFD, BGP and pings automatically.
- Try Segment Routing (SR-MPLS) in place of LDP.

## License

MIT
