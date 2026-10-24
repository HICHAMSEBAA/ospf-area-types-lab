# 02 - IP Addressing Plan

All lab addressing comes from **10.10.0.0/16**, split into aligned `/19` blocks so that each area, and the EIGRP domain, can be advertised as one prefix.

## Block allocation

| Block | Third octet | Use |
| :--- | :--- | :--- |
| `10.10.0.0/19` | 0-31 | Area 0 (backbone) |
| `10.10.32.0/19` | 32-63 | Area 1 (stub) |
| `10.10.64.0/19` | 64-95 | Area 2 (totally stubby) |
| `10.10.96.0/19` | 96-127 | Area 3 (NSSA) |
| `10.10.128.0/19` | 128-159 | Area 4 (totally NSSA) |
| `10.10.160.0/19` | 160-191 | EIGRP 100 domain |
| `10.10.192.0/19` | 192-223 | Reserved: future area |
| `10.10.224.0/19` | 224-255 | Reserved: management / services |
| `203.0.113.0/30` | n/a | R9 - Net link (documentation range, outside the lab block) |

Because every block is a power-of-two aligned `/19`, summarizing is a single `area X range` line and the summary does not overlap any other block.

## Inside a block

| Pattern | Rule |
| :--- | :--- |
| Loopback0 | `/32`, last octet = router number (R6 = `10.10.32.6`). ABR loopbacks live in Area 0 (`10.10.0.2` to `10.10.0.5`) |
| Point-to-point links | `/31` (RFC 3021). The lower address goes to the upstream router |
| Backbone links | `10.10.1.0/24` (four `/31`s used) |
| Reserved ABR-to-ABR links | `10.10.2.0/24` |
| Simulated LAN | `10.10.162.0/24` on R11 Loopback1 |

> ABR loopbacks are in Area 0 on purpose: each ABR has one router-ID that is stable and reachable from the backbone, while the area-facing link belongs to the area block.

## Loopbacks

| Router | Loopback0 | Block |
| :--- | :--- | :--- |
| R1 | `10.10.0.1/32` | Area 0 |
| R2 | `10.10.0.2/32` | Area 0 |
| R3 | `10.10.0.3/32` | Area 0 |
| R4 | `10.10.0.4/32` | Area 0 |
| R5 | `10.10.0.5/32` | Area 0 |
| R6 | `10.10.32.6/32` | Area 1 |
| R7 | `10.10.64.7/32` | Area 2 |
| R8 | `10.10.96.8/32` | Area 3 |
| R9 | `10.10.128.9/32` | Area 4 |
| R10 | `10.10.96.10/32` | Area 3 |
| R11 | `10.10.160.11/32` | EIGRP |
| Net | `8.8.8.8/32` | Simulated Internet host |

## Links

| Link | Subnet | Lower / upper address |
| :--- | :--- | :--- |
| R1 - R2 | `10.10.1.0/31` | R1 `.0`, R2 `.1` |
| R1 - R3 | `10.10.1.2/31` | R1 `.2`, R3 `.3` |
| R1 - R4 | `10.10.1.4/31` | R1 `.4`, R4 `.5` |
| R1 - R5 | `10.10.1.6/31` | R1 `.6`, R5 `.7` |
| R2 - R6 | `10.10.33.0/31` | R2 `.0`, R6 `.1` |
| R3 - R7 | `10.10.65.0/31` | R3 `.0`, R7 `.1` |
| R4 - R8 | `10.10.97.0/31` | R4 `.0`, R8 `.1` |
| R8 - R10 | `10.10.97.2/31` | R8 `.2`, R10 `.3` |
| R5 - R9 | `10.10.129.0/31` | R5 `.0`, R9 `.1` |
| R8 - R11 | `10.10.161.0/31` | R8 `.0`, R11 `.1` |
| R9 - Net | `203.0.113.0/30` | R9 `.1`, Net `.2` |

## Summarization map

| Where | Command | Summary | LSA type |
| :--- | :--- | :--- | :--- |
| R2 | `area 1 range 10.10.32.0 255.255.224.0` | `10.10.32.0/19` | Type 3 into Area 0 |
| R3 | `area 2 range 10.10.64.0 255.255.224.0` | `10.10.64.0/19` | Type 3 into Area 0 |
| R4 | `area 3 range 10.10.96.0 255.255.224.0` | `10.10.96.0/19` | Type 3 into Area 0 |
| R5 | `area 4 range 10.10.128.0 255.255.224.0` | `10.10.128.0/19` | Type 3 into Area 0 |
| R8 | `summary-address 10.10.160.0 255.255.224.0` | `10.10.160.0/19` | Type 7, translated to Type 5 at R4 |

Each `area range` and `summary-address` also installs a Null0 discard route on the router that generates it, which prevents loops when only part of a block is reachable.

## Why these choices

* **`/31` links** halve address use on point-to-point links and remove broadcast and DR/BDR semantics; combine with `ip ospf network point-to-point`.
* **One `/19` per area** gives 8,190 usable addresses of growth per area while the whole plan still fits one `/16` supernet.
* **Reserved blocks** (`10.10.192.0/19`, `10.10.224.0/19`) mean new areas or management networks do not force a re-address.
* **Loopback convention** makes router-IDs readable in `show ip ospf database` output.
