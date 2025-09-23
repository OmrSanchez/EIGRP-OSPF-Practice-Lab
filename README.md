# EIGRP ↔ OSPF Redistribution Lab – Full Documentation

**Author:** Omar D. Sanchez  
**Focus:** Route summarization, ABR behavior, bidirectional redistribution with loop prevention, verification & troubleshooting.  
**Tools:** CML with CSR100V and L3 switches for access Routing and Switching with VLANs on L3 switches.

---

## 1) Lab Goal & Outcomes
**Goal:** Build two routing domains—EIGRP (top) and OSPF (bottom) and redistribute between them at a single border router (**R0-0**). Use summarization to keep each domain lean and implement route‑map tagging to prevent loops. Verify end‑to‑end reachability between user LANs without leaking transit links.

**What was achieved**
- Designed OSPF with **Area 0** and **Area 1**; **R4 is the ABR**.  
- Implemented **inter‑area summarization** on the ABR (R4) with `area 1 range` to advertise `/24` summaries **192.168.30.0/24** and **192.168.40.0/24** from Area 1 into Area 0.  
- Implemented **EIGRP edge summaries** on R1/R2 toward SW1/SW2 so the EIGRP core only carries `/24` routes **192.168.10.0/24** and **192.168.20.0/24**.  
- Performed **bidirectional redistribution** on R0‑0: EIGRP→OSPF and OSPF→EIGRP using route‑maps, prefix‑lists, metrics, and **tags (100/200)** for loop prevention.  
- Verified that **hosts in 10/20** can reach **30/40** and vice‑versa while **transit /30 links remain hidden** from the opposite domain.  

---

## 2) Topology (High Level)

<img width="1114" height="736" alt="image" src="https://github.com/user-attachments/assets/690e80dc-ef9e-4d3a-9785-7ed0eb95f0ad" />


- **EIGRP domain:** R0, R1, R2, R3 (+ SW1/SW2 access VLANs).
- **OSPF domain:** R4 (ABR), R5, R6, R7 (+ SW3/SW4 access VLANs). R0‑0 connects EIGRP ↔ OSPF Area 0.
- **No redistribution on R4** (ABR does only inter‑area functions). All redistribution is centralized on **R0‑0**.

> **DR/BDR note:** DR/BDR is relevant only on multi‑access segments. All routed links here are p2p /30s; access VLANs are L2—so DR/BDR election is typically not in play on the routed core.

---

## 3) Addressing Plan

### 3.1 Inter‑Router Links (routed /30s)
| Link | Subnet | Side A | Side B | Notes |
|---|---|---|---|---|
| R0 ↔ R1 | 10.0.1.0/30 | R0 G1 | R1 G1 | EIGRP core |
| R0 ↔ R2 | 10.0.2.0/30 | R0 G2 | R2 G1 | EIGRP core |
| R0 ↔ R3 | 10.0.3.0/30 | R0 G3 | R3 G3 | EIGRP core |
| R2 ↔ R3 | 10.0.4.0/30 | R2 G2 | R3 G2 | EIGRP core |
| R1 ↔ R3 | 10.0.5.0/30 | R1 G3 | R3 G1 | EIGRP core |
| R0‑0 ↔ R3 | 10.2.1.0/30 | R0‑0 Gi1 | R3 G4 | Border to EIGRP |
| R0‑0 ↔ R4 | 10.1.1.0/30 | R0‑0 Gi2 | R4 G1 | OSPF **Area 0** |
| R4 ↔ R5 | 10.0.6.0/30 | R4 G2 | R5 G0/1 | OSPF **Area 1** |
| R4 ↔ R7 | 10.0.7.0/30 | R4 G3 | R7 G0/0 | OSPF **Area 1** |
| R4 ↔ R6 | 10.0.8.0/30 | R4 G4 | R6 G0/2 | OSPF **Area 1** |
| R5 ↔ R6 | 10.0.9.0/30 | R5 G0/2 | R6 G0/1 | OSPF **Area 1** |
| R6 ↔ R7 | 10.0.10.0/30 | R6 G0/0 | R7 G0/1 | OSPF **Area 1** |

> **Note:** Interface letters match the diagram labels. If your device slot/port numbers differ, keep subnets the same and adjust interface IDs in configs.

### 3.2 User VLANs / Access Networks
| Site | VLAN/Interface | Subnet | Device |
|---|---|---|---|
| SW1 LAN (behind R1) | G0/0 (or subifs) | 192.168.10.0/26, /64, /128, /192 | R1 ↔ SW1 |
| VLAN 10 | 192.168.10.0/26 |
| VLAN 20 | 192.168.10.64/26 | 
| VLAN 30 | 192.168.10.128/26 | 
| VLAN 40 | 192.168.10.192/26 |
| **Summary** | — | **192.168.10.0/24** | Summarized at R1 toward core |
| SW2 LAN (behind R2) | G0/0 (or subifs) | 192.168.20.0/26, /64, /128, /192 | R2 ↔ SW2 |
| VLAN 10 | 192.168.20.0/26 |
| VLAN 20 | 192.168.20.64/26 | 
| VLAN 30 | 192.168.20.128/26 | 
| VLAN 40 | 192.168.20.192/26 |
| **Summary** | — | **192.168.20.0/24** | Summarized at R2 toward core |
| SW3 LAN (behind R5) | subifs on R5 G0/1 | 192.168.40.0/26, /64, /128, /192 | R5 ↔ SW3 |
| VLAN 10 | 192.168.30.0/26 |
| VLAN 20 | 192.168.30.64/26 | 
| VLAN 30 | 192.168.30.128/26 | 
| VLAN 40 | 192.168.30.192/26 |
| **Summary** | — | **192.168.40.0/24** | Summarized by R4 (area range) into Area 0 |
| SW4 LAN (behind R7) | subifs on R7 G0/2.{10,20,30,40} | 192.168.30.0/26, /64, /128, /192 | R7 ↔ SW4 |
| VLAN 10 | 192.168.40.0/26 |
| VLAN 20 | 192.168.40.64/26 | 
| VLAN 30 | 192.168.40.128/26 | 
| VLAN 40 | 192.168.40.192/26 |
| **Summary** | — | **192.168.30.0/24** | Summarized by R4 (area range) into Area 0 |

### 3.3 Router IDs (OSPF/EIGRP)
- **R0‑0:** 10.255.0.0 (seen in outputs)   
- Others use `10.255.x.1` loopbacks; exact values are flexible—ensure uniqueness.

---

## 4) Protocol Design

### 4.1 EIGRP (AS 100) – Top Domain
- Routers: R0, R1, R2, R3, plus R0‑0’s Gi1 link into EIGRP.
- **K‑values:** default (K1=1 bandwidth, K3=1 delay; others 0).
- **Edge summarization:**
  - On R1 (toward SW1): `ip summary-address eigrp 100 192.168.10.0 255.255.255.0`  
  - On R2 (toward SW2): `ip summary-address eigrp 100 192.168.20.0 255.255.255.0`
- Expected in R0‑0’s RIB (pre‑redistribution): `D 192.168.10.0/24` and `D 192.168.20.0/24` only.

### 4.2 OSPF – Bottom Domain
- **Area 0:** R0‑0 ↔ R4 p2p.
- **Area 1:** R4 (ABR), R5, R6, R7, plus SW3/SW4 LANs.
- **Area summarization (ABR R4):**
  ```
  router ospf 1
   area 1 range 192.168.30.0 255.255.255.0
   area 1 range 192.168.40.0 255.255.255.0
  ```
- Expected in R0‑0’s RIB (pre‑redistribution): `O IA 192.168.30.0/24` and `O IA 192.168.40.0/24` only.

> `summary-address` under `router ospf` is for **ASBR externals**. Inter‑area summaries must be configured on the **ABR** using `area <id> range` (as done on R4).

### 4.3 Redistribution Point – R0‑0
- All redistribution centralized on R0‑0.  
- **Loop prevention:** Administrative **tags** used in both directions (100 for EIGRP→OSPF; 200 for OSPF→EIGRP).  
- **Filtering:** Only user LAN **/24** summaries are exchanged; **transit /30s** are intentionally **not** leaked.

**EIGRP → OSPF**
```cisco
ip prefix-list E2O_ALLOWED seq 5  permit 192.168.10.0/24
ip prefix-list E2O_ALLOWED seq 10 permit 192.168.20.0/24
!
route-map EIGRP-TO-OSPF deny 5
 match tag 200                    
!
route-map EIGRP-TO-OSPF permit 10
 match ip address prefix-list E2O_ALLOWED
 set tag 100
 set metric 20
 set metric-type type-2           
!
router ospf 1
 redistribute eigrp 100 route-map EIGRP-TO-OSPF
```

**OSPF → EIGRP**
```cisco
ip prefix-list O2E_ALLOWED seq 5  permit 192.168.30.0/24
ip prefix-list O2E_ALLOWED seq 10 permit 192.168.40.0/24
!
route-map OSPF-TO-EIGRP deny 5
 match tag 100                    
!
route-map OSPF-TO-EIGRP permit 10
 match ip address prefix-list O2E_ALLOWED
 set tag 200
 set metric 100000 100 255 1 1500 ! EIGRP seed metric (BW, delay, rel, load, MTU)
!
router eigrp MAIN
 address-family ipv4 unicast autonomous-system 100
  redistribute ospf 1 route-map OSPF-TO-EIGRP
```

---

## 5) Skills Demonstrated
- **IGP Design & Hierarchy:** Built OSPF backbone/area architecture; identified and used an **ABR** correctly.  
- **Route Summarization:** Implemented **EIGRP interface summaries** and **OSPF inter‑area summaries** to reduce LSDB size and routing churn.  
- **Controlled Redistribution:** Centralized at a single border (**R0‑0**), filtered to business‑relevant **/24s**, and applied **administrative tags** to prevent loops—mirrors enterprise change‑control standards.  
- **Metric & Path Control:** Used OSPF E2 metrics and EIGRP seed metrics; ready to demonstrate E1 behavior for multi‑ASBR scenarios.  
- **Operational Verification:** Wrote and executed a concise test plan (neighbors, DB, RIB, CEF, end‑to‑end pings/traces) and captured artifacts.  
- **Security Hygiene:** Planned authentication (OSPF/EIGRP MD5), iACLs, and passive interfaces; reduced attack surface by not leaking transit links.  
- **Documentation & Reproducibility:** Produced repo‑ready configs, diagrams, and a troubleshooting playbook suitable for peer review or handoff.

---

## 7) Credits
Topology and implementation by **Omar D. Sanchez**. Screenshots/diagrams to be added in `/diagrams`. All configs and `show` outputs captured from the live lab.

