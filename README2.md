# EIGRP ↔ OSPF Redistribution Lab – Full Documentation

**Author:** Omar D. Sanchez  
**Focus:** Route summarization, ABR behavior, DR/BDR concepts, bidirectional redistribution with loop prevention, verification & troubleshooting.  
**Tools:** GNS3/IOSv (or CML) with L2 switches for access VLANs.

---

## 1) Lab Goal & Outcomes
**Goal:** Build two routing domains—EIGRP (top) and OSPF (bottom)—and redistribute between them at a single border router (**R0-0**). Use summarization to keep each domain lean and implement route‑map tagging to prevent loops. Verify end‑to‑end reachability between user LANs without leaking transit links.

**What was achieved**
- Designed OSPF with **Area 0** and **Area 1**; **R4 is the ABR**.  
- Implemented **inter‑area summarization** on the ABR (R4) with `area 1 range` to advertise `/24` summaries **192.168.30.0/24** and **192.168.40.0/24** into Area 0.  
- Implemented **EIGRP edge summaries** on R1/R2 toward SW1/SW2 so the EIGRP core only carries `/24` routes **192.168.10.0/24** and **192.168.20.0/24**.  
- Performed **bidirectional redistribution** on R0‑0: EIGRP→OSPF and OSPF→EIGRP using route‑maps, prefix‑lists, metrics, and **tags (100/200)** for loop prevention.  
- Verified that **hosts in 10/20** can reach **30/40** and vice‑versa while **transit /30 links remain hidden** from the opposite domain.  

**Why this matters to recruiters**  
This lab demonstrates practical, production‑grade skills: scalable IGP design, controlled redistribution, failure‑resilient summaries, and clear verification/runbook discipline—exactly what mid‑level network engineers do in enterprise/defense environments.

---

## 2) Topology (High Level)
```
[EIGRP domain]
SW1 — R1 — R0 — R2 — SW2
          |  
          R3
          |
         R0-0  <— Border / ASBR between domains
          |
[OSPF Area 0]
          R4 (ABR)
        /  |   \
   (A1)   (A1)  (A1)
      R5 — R6 — R7 — SW4
      |
     SW3
```
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
| R0‑0 ↔ R3 | 10.2.1.0/30 | R0‑0 Gi1 | R3 G4 | Border to EIGRP |
| R0‑0 ↔ R4 | 10.1.1.0/30 | R0‑0 Gi2 | R4 G? | OSPF **Area 0** |
| R4 ↔ R5 | 10.0.6.0/30 | R4 G2 | R5 G0/1 | OSPF **Area 1** |
| R4 ↔ R7 | 10.0.7.0/30 | R4 G3 | R7 G0/0 | OSPF **Area 1** |
| R4 ↔ R6 | 10.0.8.0/30 | R4 G4 | R6 G0/0 | OSPF **Area 1** |
| R5 ↔ R6 | 10.0.9.0/30 | R5 G0/2 | R6 G0/1 | OSPF **Area 1** |
| R6 ↔ R7 | 10.0.10.0/30 | R6 G0/0 or G0/1 | R7 G0/1 | OSPF **Area 1** |

> **Note:** Interface letters match the diagram labels. If your device slot/port numbers differ, keep subnets the same and adjust interface IDs in configs.

### 3.2 User VLANs / Access Networks
| Site | VLAN/Interface | Subnet | Device |
|---|---|---|---|
| SW1 LAN (behind R1) | G0/0 (or subifs) | 192.168.10.0/26, /64, /128, /192 | R1 ↔ SW1 |
| **Summary** | — | **192.168.10.0/24** | Summarized at R1 toward core |
| SW2 LAN (behind R2) | G0/0 (or subifs) | 192.168.20.0/26, /64, /128, /192 | R2 ↔ SW2 |
| **Summary** | — | **192.168.20.0/24** | Summarized at R2 toward core |
| SW3 LAN (behind R5) | subifs on R5 G0/1 | 192.168.40.0/26, /64, /128, /192 | R5 ↔ SW3 |
| **Summary** | — | **192.168.40.0/24** | Summarized by R4 (area range) into Area 0 |
| SW4 LAN (behind R7) | subifs on R7 G0/2.{10,20,30,40} | 192.168.30.0/26, /64, /128, /192 | R7 ↔ SW4 |
| **Summary** | — | **192.168.30.0/24** | Summarized by R4 (area range) into Area 0 |

### 3.3 Router IDs (OSPF/EIGRP)
- **R0‑0:** 10.255.0.0 (seen in outputs)  
- **R7:** 10.255.7.1; **R6:** 10.255.6.1 (from screenshots)  
- Others may use `10.255.x.1` loopbacks; exact values are flexible—ensure uniqueness.

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
 match tag 200                    ! block anything that came from OSPF
!
route-map EIGRP-TO-OSPF permit 10
 match ip address prefix-list E2O_ALLOWED
 set tag 100
 set metric 20
 set metric-type type-2           ! if supported; E2 is default
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
 match tag 100                    ! block anything that came from EIGRP
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

> On some IOS, `subnets` keyword is not used for OSPF redistribution—classless networks are redistributed by default.

---

## 5) Verification & Evidence
Use these sanity checks and capture outputs for the repo.

### 5.1 Neighbor & Topology
```
! EIGRP
show ip eigrp neighbors
show ip eigrp topology
show ip protocols

! OSPF
show ip ospf neighbor
show ip ospf interface brief
show ip ospf database summary
```

### 5.2 Routing Table Snapshots (R0‑0, R1, R2, R4, R7)
```
show ip route | include 192.168.(10|20|30|40)
show ip route eigrp
show ip route ospf
show ip ospf database external | include 192.168.10.0|192.168.20.0
```
Expected highlights:
- **R0‑0:** `D 192.168.10.0/24`, `D 192.168.20.0/24`, `O IA 192.168.30.0/24`, `O IA 192.168.40.0/24`.
- **R1/R2:** `D EX 192.168.30.0/24`, `D EX 192.168.40.0/24` (from OSPF via R0‑0).
- **R4/R7:** `O E2 192.168.10.0/24`, `O E2 192.168.20.0/24` (from EIGRP via R0‑0).

### 5.3 End‑to‑End Tests (use valid sources)
```
! EIGRP → OSPF
R1# ping 192.168.30.1 source 192.168.10.1
R2# ping 192.168.40.1 source 192.168.20.1
R1# traceroute 192.168.30.1 source 192.168.10.1

! OSPF → EIGRP
R7# ping 192.168.10.1 source 192.168.30.1
R5# ping 192.168.20.1 source 192.168.40.1
R7# traceroute 192.168.10.1 source 192.168.30.1
```

### 5.4 Tag Validation
```
R0-0# show ip ospf database external | section 192.168.10.0
   ... Tag: 100
R0-0# show ip eigrp topology 192.168.30.0/24 detail
   External data:
   Tag is 200
```

---

## 6) Security & Hygiene
- **Obscurity vs Security:** Summaries and filtering prevent information leakage and reduce attack surface, but they’re not security controls by themselves.
- **Recommended hardening (next steps):**
  - **EIGRP MD5** on p2p links (R0‑0↔R3, core links).  
  - **OSPF MD5** on Area 0 link (R0‑0↔R4) and optionally Area 1 links.  
  - **iACLs** on access-facing interfaces to block routing protocols from host VLANs (IP proto 88/89, etc.).  
  - **Passive** routing on loopbacks and user-facing SVIs.

---

## 7) Troubleshooting Playbook
1. **Routing present?** `show ip route <prefix>` on both sides.  
2. **Right source?** Ping from an address that’s actually redistributed (avoid transit /30s unless explicitly leaked).  
3. **Neighbors stable?** Check OSPF/EIGRP neighbors & timers.  
4. **Filters/tags correct?** `show route-map`, `show ip prefix-list`, DB/tag checks above.  
5. **Next hop reachable?** `show ip cef exact-route <src> <dst>` (if supported).  
6. **Masks & SVIs up?** `show ip int br` on SW1/SW2/SW3/SW4.  

Common fixes: add missing `area 1 range` on R4, add EIGRP seed metric for OSPF→EIGRP, ensure route‑map sequence order, remove unintended `redistribute connected`.

---

## 8) Repository Structure (Suggested)
```
EIGRP-OSPF-Redistribution-Lab/
├─ README.md                      # This document (trimmed for GH)
├─ diagrams/
│  ├─ topology-full.png
│  ├─ eigrp-domain.png
│  └─ ospf-domain.png
├─ configs/
│  ├─ R0.txt   R1.txt  R2.txt  R3.txt
│  ├─ R0-0.txt R4.txt  R5.txt  R6.txt  R7.txt
│  └─ SW1.txt SW2.txt SW3.txt SW4.txt
├─ verifications/
│  ├─ r0-0_show_ip_route.txt
│  ├─ r4_show_ip_route.txt
│  ├─ r1_show_ip_route.txt
│  ├─ r7_show_ip_route.txt
│  └─ ping_traceroute_tests.txt
├─ notes/
│  ├─ lessons-learned.md
│  └─ hardening-checklist.md
└─ extras/
   ├─ nssa-variant.md
   └─ e1-vs-e2-notes.md
```

---

## 9) Recruiter‑Focused Skills Demonstrated
- **IGP Design & Hierarchy:** Built OSPF backbone/area architecture; identified and used an **ABR** correctly.  
- **Route Summarization:** Implemented **EIGRP interface summaries** and **OSPF inter‑area summaries** to reduce LSDB size and routing churn.  
- **Controlled Redistribution:** Centralized at a single border (**R0‑0**), filtered to business‑relevant **/24s**, and applied **administrative tags** to prevent loops—mirrors enterprise change‑control standards.  
- **Metric & Path Control:** Used OSPF E2 metrics and EIGRP seed metrics; ready to demonstrate E1 behavior for multi‑ASBR scenarios.  
- **Operational Verification:** Wrote and executed a concise test plan (neighbors, DB, RIB, CEF, end‑to‑end pings/traces) and captured artifacts.  
- **Security Hygiene:** Planned authentication (OSPF/EIGRP MD5), iACLs, and passive interfaces; reduced attack surface by not leaking transit links.  
- **Documentation & Reproducibility:** Produced repo‑ready configs, diagrams, and a troubleshooting playbook suitable for peer review or handoff.

**Short resume line (use in projects section):**  
> Designed and implemented dual‑domain routing lab (EIGRP & OSPF) with ABR summarization and controlled bidirectional redistribution at a single border router using route‑maps/tags; validated end‑to‑end reachability and minimized route churn via /24 summaries. Deliverables include configs, verification artifacts, and hardening plan.

---

## 10) Optional Enhancements (for follow‑up commits)
- Convert **Area 1 to NSSA** and compare Type‑7 vs Type‑5 behavior.  
- Flip EIGRP→OSPF to **E1** and introduce a second ASBR to showcase internal cost influence.  
- Add **MD5** auth on EIGRP/OSPF adjacencies and document key‑rollover procedure.  
- Script verifications (Python/Netmiko) to capture `show` outputs automatically.  
- Introduce link failures and measure convergence (EIGRP DUAL vs OSPF SPF timers).

---

## 11) Appendix – Sample Device Snippets
> **R4 (ABR) – OSPF area range**
```cisco
router ospf 1
 router-id 10.255.4.1
 network 10.1.1.0 0.0.0.3 area 0
 network 10.0.6.0 0.0.0.3 area 1
 network 10.0.7.0 0.0.0.3 area 1
 network 10.0.8.0 0.0.0.3 area 1
 area 1 range 192.168.30.0 255.255.255.0
 area 1 range 192.168.40.0 255.255.255.0
```

> **R1 – EIGRP summary toward SW1**
```cisco
router eigrp MAIN
 address-family ipv4 unicast autonomous-system 100
  af-interface GigabitEthernet0/0
   summary-address 192.168.10.0 255.255.255.0
```

> **R0‑0 – Redistribution both ways**
```cisco
ip prefix-list E2O_ALLOWED seq 5  permit 192.168.10.0/24
ip prefix-list E2O_ALLOWED seq 10 permit 192.168.20.0/24
ip prefix-list O2E_ALLOWED seq 5  permit 192.168.30.0/24
ip prefix-list O2E_ALLOWED seq 10 permit 192.168.40.0/24
!
route-map EIGRP-TO-OSPF deny 5
 match tag 200
route-map EIGRP-TO-OSPF permit 10
 match ip address prefix-list E2O_ALLOWED
 set tag 100
 set metric 20
 set metric-type type-2
!
route-map OSPF-TO-EIGRP deny 5
 match tag 100
route-map OSPF-TO-EIGRP permit 10
 match ip address prefix-list O2E_ALLOWED
 set tag 200
 set metric 100000 100 255 1 1500
!
router ospf 1
 router-id 10.255.0.0
 redistribute eigrp 100 route-map EIGRP-TO-OSPF
!
router eigrp MAIN
 address-family ipv4 unicast autonomous-system 100
  redistribute ospf 1 route-map OSPF-TO-EIGRP
```

---

## 12) Credits
Topology and implementation by **Omar D. Sanchez**. Screenshots/diagrams to be added in `/diagrams`. All configs and `show` outputs captured from the live lab.

