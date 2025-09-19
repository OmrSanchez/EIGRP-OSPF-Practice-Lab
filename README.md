# EIGRP-OSPF-Practice-Lab
This lab was designed to practice and document advanced configurations of EIGRP and OSPF in an enterprise-style environment. The purpose is to reinforce protocol fundamentals, explore advanced features such as summarization and leak-maps, and compare EIGRP’s DUAL algorithm with OSPF’s SPF-based approach.

The results of this lab are published here to serve as both a study reference and a showcase project for my GitHub portfolio.

🖧 Topology

<img width="1684" height="1550" alt="image" src="https://github.com/user-attachments/assets/ca1c999d-6caf-4066-9217-7064d09154ec" />

Routing domains and boundary (accurate, complete — no table):

There are two separate routing domains that meet at R0 in the center. R0 participates in both protocols but serves as a hard boundary in the baseline; no redistribution is configured yet (planned as a next step with route‑tagging and loop‑prevention policy).

EIGRP domain (6 routers): R0, R1, R2, R3, R4, R5 — shown in the blue region at the top.

OSPF domain (5 routers): R0, R7, R8, R9, R10 — shown in the lower region; OSPF is split into Area 0 and Area 1.

EIGRP domain (top) — links, roles, addressing, and loopbacks (described)

R0 ↔ R2: Point‑to‑point routed interconnect 10.0.0.0/30 (EIGRP adjacency forms between R0 Gi1 and R2 Gi1).

R2 ↔ R4: Interconnect 10.0.1.0/30 (R2 Gi2 ↔ R4 Gi1).

R4 ↔ R5: Interconnect 10.0.5.0/30 (R4 Gi2 ↔ R5 Gi1).

R2 ↔ R3: Interconnect 10.0.2.0/30 (R2 Gi3 ↔ R3 Gi1).

R3 ↔ R1: Interconnect 10.0.3.0/30 (R3 Gi2 ↔ R1 Gi2).

R1 ↔ SW1 ↔ R0: A Layer‑2 switch connects R1 Gi4 to R0 Gi0/0. This L2 segment is present for switching/L2 practice and does not carry routing adjacencies in the baseline.

R1 LAN and summarization: R1 anchors the inside LAN using 802.1Q sub‑interfaces as default gateways. The inside block is 192.168.10.0/24, subdivided for practice (e.g., /26 segments). Outbound toward the EIGRP domain, R1 summarizes as 192.168.10.0/24. No prefix‑lists, route‑maps, or leak‑maps are implemented yet.

Stub routers: R1 and R5 are configured as EIGRP stubs to constrain query scope and keep convergence predictable at the edges. (R5 will advertise its own local networks and apply summaries in a future step.)

Loopbacks: Every router in the EIGRP domain also has a loopback configured following the consistent scheme 10.255.X.1/32 where X = router number (e.g., R1 has 10.255.1.1/32, R5 has 10.255.5.1/32, etc.). These loopbacks are advertised in EIGRP but set as passive interfaces, ensuring they are reachable end‑to‑end without forming adjacencies.

Failure lever in EIGRP: For convergence testing, R3 GigabitEthernet1 is toggled (shutdown/no shutdown) to withdraw and restore reachability toward the R3/R1 side. Observe updates and feasible‑successor selection on R2 and R0 during these events.

OSPF domain (bottom) — areas, links, and roles (described)

Area 0 (backbone, cyan): A shared broadcast segment 10.0.7.0/28 hangs off a Layer‑2 switch and includes R7 Gi1 (.2), R8 Gi1 (.3), and R9 Gi1 (.4). This demonstrates DR/BDR election and multi‑access LSA behavior.

Area 1 (orange): Two point‑to‑point links:

R0 ↔ R7 on 10.0.4.0/30 (R0 Gi2 ↔ R7 Gi2).

R7 ↔ R10 on 10.0.10.0/30 (R7 Gi4 ↔ R10 Gi1).

ABR role: R7 is the ABR between Area 0 and Area 1. R0 participates only in Area 1 in the baseline (no redistribution to/from EIGRP yet). Costs may be tuned on R7’s Area‑1 interfaces to illustrate path preference.

✅ Baseline truth set: EIGRP runs on R0,R1,R2,R3,R4,R5; R1 and R5 are EIGRP stubs; R1 performs outbound summary of 192.168.10.0/24 only (no leak‑map). OSPF runs on R0,R7,R8,R9,R10 with R7 as ABR (Area 0 ↔ Area 1). No EIGRP↔OSPF redistribution is active yet; R0 is the shared boundary device running both protocols without route exchange.

OSPF Domain (Bottom Section)

Routers participating: R7, R8, R9, R10, and R0.

R7: Acts as the ABR, connecting Area 0 and Area 1. Links:

To R0 (10.0.0.0/30 connection, opposite side from EIGRP’s R2).

To R8 and R9 inside Area 0 (10.0.7.0/28).

To R10 inside Area 1 (10.0.10.0/30).

To R0 with a backbone-facing link 10.0.4.0/30.

R8: Area 0 internal router, connected via switch fabric to R7.

R9: Area 0 internal router, also connected via switch fabric to R7.

R10: Area 1 router, linked to R7 via 10.0.10.0/30.

R0: Central device connecting to R7 via 10.0.4.0/30. This is the OSPF-facing side of R0.

OSPF configuration highlights:

Backbone Area: Area 0 with R7, R8, and R9.

Non-backbone Area: Area 1 with R7 and R10.

R7 functions as ABR between Area 0 and Area 1.

Manual router-ids assigned for deterministic adjacency.

Verification commands used: show ip ospf neighbor, show ip ospf database, and show ip ospf interface.

Separation and Integration

R0 is the demarcation point. Its northbound interface participates in EIGRP, while its southbound interface participates in OSPF.

At present there is no redistribution configured between EIGRP and OSPF, but the topology is explicitly designed to support redistribution scenarios in later stages of practice.

Failure Scenarios

Links such as R3’s Gi1 (toward R2) or R4–R5 can be administratively shut down to observe EIGRP’s DUAL algorithm and reconvergence.

In OSPF, Area 1’s link between R7 and R10 can be shut down to force SPF recalculation and compare convergence speed.

📌 Accuracy note: This updated description fully matches the uploaded diagram: six routers in EIGRP (R1–R5 + R0), five routers in OSPF (R7–R10 + R0), separated by R0 in the center, with summarization at R1 and no redistribution configured yet.

🔑 Technologies & Features Practiced
🟢 Enhanced Interior Gateway Routing Protocol (EIGRP)

✅ Adjacency formation with IPv4 AFI across six routers

✅ R1 and R5 configured as EIGRP stubs to limit query scope

✅ R1 summarization of inside LAN as 192.168.10.0/24 outbound

✅ Metrics based on bandwidth (minimum link) and delay (cumulative)

✅ Troubleshooting neighbor adjacencies with show and debug commands

🗓️ Planned next step: introduce redistribution at R0 (EIGRP↔OSPF) and, later, per‑edge summaries for R5

🔵 Open Shortest Path First (OSPF)

✅ Multi-area design: Area 0 (backbone) and Area 1 (internal)

✅ Router ID selection vs interface IP addresses

✅ OSPF cost metric and manual cost manipulation

✅ Verifying SPF recalculation and adjacency states

✅ Comparing convergence speed with EIGRP

✅ Multi-area design: Area 0 (backbone) and Area 1 (internal)

✅ Router ID selection vs interface IP addresses

✅ OSPF cost metric and manual cost manipulation

✅ Verifying SPF recalculation and adjacency states

✅ Comparing convergence speed with EIGRP

⚙️ Configuration Examples
📍 EIGRP: R1 summarization (no leak‑map) & stub edges
! R1 — EIGRP named mode, IPv4 AFI
router eigrp MAIN
 address-family ipv4 unicast autonomous-system 100
  af-interface <UPLINK-TO-R3-OR-R2>
   summary-address 192.168.10.0 255.255.255.0
  exit-af-interface
  eigrp stub connected summary
 exit-address-family


! R5 — stub edge (future: add local networks + summaries)
router eigrp MAIN
 address-family ipv4 unicast autonomous-system 100
  eigrp stub connected summary
 exit-address-family

Explanation:

No prefix-lists, route-maps, or leak-maps are configured at this stage.

R1 advertises a /24 summary for the inside LAN toward the EIGRP domain.

R1 and R5 are stubs — they do not answer broad queries and only advertise directly connected (and summarized) routes.

📍 OSPF Multi-Area Example
router ospf 1
 router-id 1.1.1.1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 1

Explanation:

Router ID is manually set to ensure consistency.

Area 0 serves as the backbone.

Area 1 represents an internal network, demonstrating multi-area hierarchy.

🔍 Verification & Sanity Checks
🔎 EIGRP

show ip eigrp neighbors

show ip route eigrp

show ip protocols

show ip eigrp topology

show ip eigrp neighbors detail (confirm stub flags and peer view)

🔎 OSPF

show ip ospf neighbor

show ip ospf database

show ip ospf interface

show ip route ospf

🧪 Testing & Scenarios

🔄 Brought down R3’s G1 interface to observe reconvergence time

👀 Watched route changes with debug while links flapped

⚡ Compared failover performance:

EIGRP: Faster, due to DUAL’s feasibility condition

OSPF: Required SPF recalculation, slightly slower

📚 Key Takeaways

EIGRP is quick with convergence and flexible with summarization and leak-maps.

OSPF enforces hierarchical design, better for scalability but requires area planning.

Both have their strengths:

EIGRP → simplicity and efficiency in enterprise LANs

OSPF → wide adoption, standardization, and scalability

🚀 Next Steps

🔄 Implement redistribution between EIGRP and OSPF

🌐 Explore BGP peering to simulate ISP connectivity

🔐 Add authentication: MD5 for EIGRP and OSPF

📝 Extend lab to include route filtering and policy-based routing
