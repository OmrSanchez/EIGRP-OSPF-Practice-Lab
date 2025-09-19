# EIGRP & OSPF Practice Lab

## Project Overview
This lab was designed to practice and document advanced configurations of **EIGRP** and **OSPF**. The purpose is to reinforce protocol fundamentals, explore summarization, stub configurations, and compare EIGRP’s DUAL algorithm with OSPF’s SPF-based approach.

The results of this lab are published here to serve as both a study reference and a showcase project for my GitHub portfolio.

---

## Topology

<img width="1684" height="1550" alt="image" src="https://github.com/user-attachments/assets/66b8b8f3-87d2-4136-8b35-a051804d8098" />

There are two separate routing domains that meet at **R0** in the center. R0 participates in both protocols but serves as a boundary in the baseline. No redistribution is configured yet (this will be a future step).

- **EIGRP domain (6 routers):** R0, R1, R2, R3, R4, R5  
- **OSPF domain (5 routers):** R0, R7, R8, R9, R10  
- **Boundary:** R0 runs both protocols but does not redistribute routes.

### EIGRP domain (top)
- **R0 ↔ R2**: 10.0.0.0/30 (R0 Gi1 ↔ R2 Gi1)  
- **R2 ↔ R4**: 10.0.1.0/30 (R2 Gi2 ↔ R4 Gi1)  
- **R4 ↔ R5**: 10.0.5.0/30 (R4 Gi2 ↔ R5 Gi1)  
- **R2 ↔ R3**: 10.0.2.0/30 (R2 Gi3 ↔ R3 Gi1)  
- **R3 ↔ R1**: 10.0.3.0/30 (R3 Gi2 ↔ R1 Gi2)  
- **R1 ↔ SW1 ↔ R0**: Layer 2 switch connects R1 Gi4 to R0 Gi0/0 (no adjacency formed)  

**LAN and summarization:** 
R1 anchors the inside LAN using 802.1Q sub-interfaces as default gateways. The inside block is 192.168.10.0/24, subdivided into /26 segments. Outbound toward the EIGRP domain, R1 summarizes as 192.168.10.0/24. No prefix-lists, route-maps, or leak-maps are implemented yet.

**Stub routers:** 
R1 and R5 are configured as EIGRP stubs to constrain query scope and keep convergence predictable at the edges. R5 will advertise its own local networks and summaries in the future.

**Loopbacks:** 
Every router in the EIGRP domain also has a loopback configured following the scheme `10.255.X.1/32` where X = router number (e.g., R1 = 10.255.1.1/32). These loopbacks are passive interfaces.

**Failure testing:** 
R3 Gi1 is toggled (shutdown/no shutdown) to test convergence. Observe updates and feasible-successor selection on R2 and R0.

### OSPF domain (bottom)
- **Area 0 (backbone):** Shared broadcast segment 10.0.7.0/28 on a Layer 2 switch. Includes R7 Gi1 (.2), R8 Gi1 (.3), and R9 Gi1 (.4). This demonstrates DR/BDR election and multi-access LSA behavior.  
- **Area 1:**  
  - R0 ↔ R7: 10.0.4.0/30 (R0 Gi2 ↔ R7 Gi2)  
  - R7 ↔ R10: 10.0.10.0/30 (R7 Gi4 ↔ R10 Gi1)  
- **ABR role:** R7 is the ABR between Area 0 and Area 1. R0 participates only in Area 1.

---

## Technologies & Features Practiced

### EIGRP
- Adjacency formation with IPv4 AFI across six routers  
- R1 and R5 configured as EIGRP stubs  
- R1 summarization of inside LAN as 192.168.10.0/24 outbound  
- Metrics based on bandwidth (minimum link) and delay (cumulative)  
- Troubleshooting neighbor adjacencies with show and debug commands  
- Planned next step: introduce redistribution at R0 (EIGRP ↔ OSPF) and configure R5 summaries  

### OSPF
- Multi-area design: Area 0 (backbone) and Area 1 (internal)  
- Router ID selection vs interface IP addresses  
- OSPF cost metric and manual cost manipulation  
- Verifying SPF recalculation and adjacency states  
- Comparing convergence speed with EIGRP  

---

## Verification & Sanity Checks

### EIGRP
- `show ip eigrp neighbors`
- `show ip route eigrp`
- `show ip protocols`
- `show ip eigrp topology`
- `show ip eigrp neighbors detail` (confirm stub status)

### OSPF
- `show ip ospf neighbor`
- `show ip ospf database`
- `show ip ospf interface`
- `show ip route ospf`

---

## Testing & Scenarios
- Shut/no shut **R3 Gi1** to observe EIGRP reconvergence  
- Observe OSPF adjacency states in multi-access and point-to-point links  
- Compare failover performance:  
  - **EIGRP:** faster, due to DUAL feasibility condition  
  - **OSPF:** requires SPF recalculation  

---

## Key Takeaways
- EIGRP converges quickly and benefits from stub configurations and summarization  
- OSPF enforces hierarchical design, better for scalability but requires careful area planning  
- Both protocols have strengths:  
  - *EIGRP* → efficiency and stub-friendly  
  - *OSPF* → standardization and scalability  

---

## Next Steps
- Implement redistribution between **EIGRP and OSPF** at R0  
- Configure **R5** with its own networks and summaries  
- Explore **BGP peering** to simulate ISP connectivity  
- Add **authentication (MD5)** for both EIGRP and OSPF  
- Extend lab with **policy-based routing** and filtering  


