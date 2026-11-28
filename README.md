
1 edit
Service-Provider MPLS L3VPN Lab: FRRouting + Docker + containerlab
A telco-style MPLS network built entirely from open-source parts. FRRouting (FRR) routers run in Docker containers and are wired together with containerlab. The lab was built in five stages: an MPLS core, customer VPNs, a redundant core with failure testing, BGP route reflectors, and automated health checks.

What this project demonstrates
An MPLS core with OSPF and LDP, and a BGP-free P layer

Layer 3 VPNs with VRFs, RD/RT and MP-BGP VPNv4

Two customers using overlapping IP ranges, kept isolated

A redundant core path and failure tests (link down and silent packet loss)

BFD for fast failure detection

Two route reflectors with the same cluster ID

A Python script that checks the whole lab and prints a pass/fail report

Topology
text
                  RR1 (1.1.1.1)        RR2 (2.2.2.2)     <- control plane only
                    |       \          /       |
                    |        \        /        |
                 +--+--------- P1 ---------+---+
                 |            (10.0.0.2)       |
 CE1A --\                                           /-- CE2A
         PE1 (10.0.0.1)                 PE2 (10.0.0.3)
 CE1B --/                                           \-- CE2B
                 |            (10.0.0.4)       |
                 +------------ P2 ------------+
Role	Nodes
Provider edge (PE)	PE1, PE2
Provider core (P)	P1 (primary path), P2 (backup path)
Route reflectors	RR1, RR2 (control plane only, not in the data path)
Customer edge (CE)	CE1A, CE2A (CustA); CE1B, CE2B (CustB)
Design summary
Layer	Choice
Underlay	OSPF area 0, /32 loopbacks. P1 links cost 10 (primary), P2 links cost 100 (backup)
Labels	LDP transport labels on PE and P core links
Overlay	MP-BGP VPNv4 (AS 65000) from each PE to both route reflectors
Route reflection	RR1 and RR2 share cluster ID 1.1.1.1. Each PE peers with both
Customers	One VRF each, own RD and RT. eBGP between PE and CE
Fast detection	BFD (300 ms x 3) on core links
P1 and P2 run no BGP. They only switch labels, so they never see customer routes.

Addressing
Device	Loopback	Notes
PE1	10.0.0.1/32	AS 65000
P1	10.0.0.2/32	no BGP
PE2	10.0.0.3/32	AS 65000
P2	10.0.0.4/32	no BGP
RR1	1.1.1.1/32	AS 65000, cluster-id 1.1.1.1
RR2	2.2.2.2/32	AS 65000, cluster-id 1.1.1.1
Link	Subnet	OSPF cost
PE1 eth1 - P1 eth1	10.1.12.0/24	10
P1 eth2 - PE2 eth1	10.1.23.0/24	10
PE1 eth4 - P2 eth1	10.1.14.0/24	100
P2 eth2 - PE2 eth4	10.1.34.0/24	100
RR1 eth1 - P1 eth3	10.1.100.0/24	10
RR1 eth2 - P2 eth3	10.1.101.0/24	100
RR2 eth1 - P1 eth5	10.1.102.0/24	100
RR2 eth2 - P2 eth5	10.1.103.0/24	10
Customer router	AS	PE-CE link	LAN (on lo)	VRF
CE1A	65101	10.10.1.0/30 (PE1 eth2)	192.168.1.1/24	CustA
CE2A	65102	10.10.2.0/30 (PE2 eth2)	192.168.2.1/24	CustA
CE1B	65201	10.20.1.0/30 (PE1 eth3)	192.168.1.1/24	CustB
CE2B	65202	10.20.2.0/30 (PE2 eth3)	192.168.2.1/24	CustB
VRF	RD	RT	Kernel table
CustA	65000:1	65000:1	10
CustB	65000:2	65000:2	20
CustA and CustB both use 192.168.1.0/24 and 192.168.2.0/24. The RD keeps the VPNv4 routes unique and the RT keeps the customers apart.

Requirements
Linux host or VM, for example Ubuntu in VMware Fusion or UTM on a Mac. Docker Desktop's VM may not include the MPLS kernel modules.

Docker and containerlab

Host kernel modules: mpls_router, mpls_iptunnel, mpls_gso, vrf

Repository layout
text
.
├── mpls-core.clab.yml          # containerlab topology
├── daemons-p                   # P routers: zebra, ospfd, ldpd, bfdd (no bgpd)
├── daemons-pe                  # PE routers: adds bgpd
├── daemons-ce                  # CE routers: zebra, bgpd
├── daemons-rr                  # route reflectors: zebra, ospfd, bgpd
├── pe1.conf  pe1-vrf.conf
├── pe2.conf  pe2-vrf.conf
├── p1.conf   p2.conf
├── rr1.conf  rr2.conf
├── ce1a.conf ce1b.conf ce2a.conf ce2b.conf
├── bfd-on.sh                   # attaches BFD to OSPF on core links
├── check_lab.py                # automated health check
└── docs/                       # screenshots and packet captures
Quick start
bash
sudo modprobe mpls_router mpls_iptunnel mpls_gso vrf
sudo containerlab deploy -t mpls-core.clab.yml
./bfd-on.sh
python3 check_lab.py
Open a router CLI:

bash
docker exec -it clab-mpls_core-pe1 vtysh
Tear down:

bash
sudo containerlab destroy -t mpls-core.clab.yml
How it was built
Stage 1: MPLS core
OSPF provides loopback reachability and LDP assigns labels. Each router needs net.mpls.platform_labels set and net.mpls.conf.<interface>.input=1 on its MPLS interfaces, because containers share the host kernel.

Stage 2: Customer VPNs
Each PE creates a kernel VRF per customer (ip link add CustA type vrf table 10) and moves the customer-facing interface into it. FRR then applies the RD, RT and label vpn export auto, and exports/imports routes between the VRF and the VPN table. Packets cross the core with two labels: an outer LDP transport label and an inner VPN label.

Stage 3: Redundancy and failure testing
P2 provides a backup path with a higher OSPF cost. BFD (300 ms interval, multiplier 3) is attached to core links by bfd-on.sh so OSPF learns about silent failures within about a second.

Stage 4 and 5: Route reflectors
Both PEs peer over their loopbacks with RR1 and RR2 only, instead of with each other. The RRs share the cluster ID 1.1.1.1 and do not peer with each other.

Verification
On a PE (inside vtysh):

Command	Expected result
show ip ospf neighbor	Full adjacencies with P1 and P2
show mpls ldp neighbor	LDP sessions operational
show mpls ldp binding	Labels for the loopbacks
show bfd peers	Core BFD sessions up
show bgp ipv4 vpn summary	Sessions to 1.1.1.1 and 2.2.2.2 Established
show bgp ipv4 vpn	VPN routes with RDs, labels, originator and cluster list
show ip route vrf CustA	CustA routes only
On P1 and P2, show bgp summary should show that BGP is not running.

End-to-end pings, sourced from the customer LAN:

bash
docker exec clab-mpls_core-ce1a ping -c3 -I 192.168.1.1 192.168.2.1
docker exec clab-mpls_core-ce1b ping -c3 -I 192.168.1.1 192.168.2.1
Label capture while a ping runs:

bash
docker exec clab-mpls_core-p1 tcpdump -nv -i eth1 mpls   # PE1 -> P1: transport + VPN label
docker exec clab-mpls_core-p1 tcpdump -nv -i eth2 mpls   # P1 -> PE2: VPN label only (after PHP)
Failure tests
Test	Command	What to observe
Link down	docker exec clab-mpls_core-pe1 ip link set eth1 down	Traffic moves to P2. The transport label changes and the VPN label stays the same
Silent loss	tc qdisc add dev eth1 root netem loss 100% on both ends	Slow recovery without BFD, fast with BFD
RR failure	docker exec clab-mpls_core-rr1 ip link set eth1 down (and eth2)	Existing traffic unaffected. New routes still propagate through RR2
Restore	ip link set eth1 up / tc qdisc del dev eth1 root	Traffic returns to the P1 path
Results (fill in from your own runs):

Test	BFD	Lost pings	Approx. outage
Link down	Off		
Silent loss	Off		
Silent loss	On		
RR1 down	n/a		
Automated health check
check_lab.py runs vtysh commands inside each container with docker exec and parses the output. It checks OSPF, LDP, BFD and VPNv4 session counts, confirms P1 and P2 run no BGP, checks the VRF routes, and runs end-to-end and isolation pings. It prints PASS or FAIL for each check and exits with code 1 if anything fails.

Key findings
The core carries two labels: an outer LDP transport label and an inner VPN label.

After penultimate hop popping, the last P-to-PE link carries only the VPN label.

During failover only the transport label changes, because the PE-to-PE VPN labels are unaffected.

P routers stay BGP-free and never see customer routes.

Overlapping customer prefixes stay separate because of the RD and RT.

Link-down failures converge quickly. Silent failures need BFD.

Route reflectors sit outside the data path, so an RR failure doesn't break existing traffic.

Limitations
Lab-grade timers and a single OSPF area.

Two RRs sharing a cluster ID can fail to exchange routes if one PE loses RR1 while another loses RR2 at the same moment.

No traffic engineering or segment routing yet.

Configs were written for FRR in containers. Command output can vary between FRR versions.

Possible next steps
Add a third PE.

Replace LDP with Segment Routing (SR-MPLS).

Add traffic engineering.

Run check_lab.py in CI.

License
MIT

12
Describe the change...

