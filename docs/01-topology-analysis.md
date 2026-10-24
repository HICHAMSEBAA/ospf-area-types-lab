# 01 - Topology Analysis

Analysis of the lab diagram ([`topology.png`](topology.png)): links, router roles, area assignment, and the weaknesses that matter for design discussion.

## Link inventory

| Link | Interfaces | Zone |
| :--- | :--- | :--- |
| R1 - R2 | Gi0/0 - Gi0/0 | Area 0 |
| R1 - R3 | Gi0/1 - Gi0/0 | Area 0 |
| R1 - R4 | Gi0/2 - Gi0/0 | Area 0 |
| R1 - R5 | Gi0/3 - Gi0/0 | Area 0 |
| R2 - R6 | Gi0/1 - Gi0/0 | Area 1 |
| R3 - R7 | Gi0/1 - Gi0/0 | Area 2 |
| R4 - R8 | Gi0/1 - Gi0/0 | Area 3 |
| R8 - R10 | Gi0/1 - Gi0/0 | Area 3 |
| R5 - R9 | Gi0/1 - Gi0/0 | Area 4 |
| R8 - R11 | Gi0/2 - Gi0/0 | EIGRP 100 (not OSPF) |
| R9 - Net | Gi0/1 - Gi0/0 | Internet (not OSPF) |

The diagram shows no area numbers. The plan assigns them by colour: green = Area 0, red = Area 1, yellow = Area 2, grey = Area 3, purple = Area 4.

## Router roles

| Router | Role | Why |
| :--- | :--- | :--- |
| R1 | Backbone internal router (hub) | All four links are in Area 0 |
| R2, R3, R4, R5 | ABRs | One link in Area 0, one in their own area |
| R6, R7, R10 | Internal leaf routers | Single link inside one area |
| R8 | Internal router and **ASBR** | Redistributes the EIGRP 100 domain behind R11 |
| R9 | Internal router and **ASBR** | Originates the Internet default learned from Net |
| R11 | EIGRP leaf | Outside OSPF |
| Net | Simulated Internet | Outside OSPF |

## What the topology implies

1. **Area 0 is contiguous.** Every ABR attaches directly to R1, so no virtual links are needed.
2. **Area types are dictated by the routers inside them.**
   * An area that contains an ASBR cannot be stub or totally stubby, so Area 3 (R8) and Area 4 (R9) must be NSSA variants.
   * Areas 1 and 2 are single-exit leaf areas with no external routes, which makes them stub and totally stubby candidates.
3. **Areas 1 and 2 can be as small as possible.** One exit means a default route is enough for routing decisions, and there is no sub-optimal-routing risk from filtering Type 3 LSAs.
4. **Each NSSA has exactly one ABR.** That ABR (R4, R5) is automatically the Type 7 to Type 5 translator. A second ABR would force a translator election.

## Weaknesses (by design, worth discussing)

| Weakness | Impact | Mitigation in a production design |
| :--- | :--- | :--- |
| R1 is the only backbone hub | Its failure partitions every area | ABR-to-ABR links (`10.10.2.0/24` is reserved), or a second hub |
| Every area is single-homed | An ABR failure isolates its area | Dual ABRs per area (NSSA translator election applies) |
| R4 - R8 - R10 is a daisy chain | R8 failure isolates R10 and the EIGRP domain | Second path to R10 |
| Single Internet exit (R9) | Internet loss is total | Second ASBR with metric-based preference |

## Boundary routers and filtering points

* **R8** is the only EIGRP boundary, so one-way redistribution plus a default route into EIGRP is loop-free.
* **R9** is the only Internet exit. Its static default and `/16` static are explained in [`03-ospf-design-decisions.md`](03-ospf-design-decisions.md).
