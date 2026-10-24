# 04 - Verification and Troubleshooting

> The expected results below describe the intended behavior of the configs. They are verified statically by `scripts/validate_lab.py` but are not captured from a live run. If your image differs, note the difference in an issue.

## Verify phase by phase

| After phase | Run | Expect |
| :--- | :--- | :--- |
| 1 (addressing) | `show ip interface brief`, ping each neighbor | Every link up/up; neighbors reachable |
| 2 (OSPF baseline) | `show ip ospf neighbor`, `show ip ospf interface brief` | All adjacencies `FULL`, no DR/BDR; every router pings every loopback |
| 3 (area types) | `show ip ospf \| include Area\|stub\|NSSA` | Correct type on every router; adjacencies back to `FULL` |
| 4 (summarization) | `show ip route ospf` on R1 | One `O IA` per area block |
| 5 (EIGRP) | `show ip eigrp neighbors` on R8 | R11 as a neighbor; `O N2 10.10.160.0/19` on R10 |
| 6 (Internet edge) | `show ip route 0.0.0.0` on R1 | `O*E2 0.0.0.0/0` via R5 |

## Neighbor counts

| Router | OSPF neighbors | Neighbors |
| :--- | :--- | :--- |
| R1 | 4 | R2, R3, R4, R5 |
| R2 | 2 | R1, R6 |
| R3 | 2 | R1, R7 |
| R4 | 2 | R1, R8 |
| R5 | 2 | R1, R9 |
| R6 | 1 | R2 |
| R7 | 1 | R3 |
| R8 | 2 (+1 EIGRP) | R4, R10 (+ R11 over EIGRP) |
| R9 | 1 | R5 |
| R10 | 1 | R8 |
| R11 | 0 (+1 EIGRP) | R8 over EIGRP |

## Expected routing tables

| Router | Expected |
| :--- | :--- |
| R7 (totally stubby) | Connected routes plus `O*IA 0.0.0.0/0` only |
| R6 (stub) | `O IA` for Area 0 prefixes and the Area 2/3/4 summaries, plus `O*IA 0.0.0.0/0`; no `E` routes |
| R10 (NSSA) | `O IA` routes, `O N2 10.10.160.0/19`, `O*N2 0.0.0.0/0` |
| R9 (totally NSSA) | `S* 0.0.0.0/0` via Net and `S 10.10.0.0/16` via R5 (the OSPF default is in the database but the static wins) |
| R1 | `O IA` summaries for Areas 1-4, `O E2 10.10.160.0/19` via R4, `O*E2 0.0.0.0/0` via R5 |
| R11 | `D* 0.0.0.0/0` via R8 |

## Expected LSDB (`show ip ospf database`)

| Area | Expected LSAs |
| :--- | :--- |
| 0 | Type 1 (R1-R5), Type 3 (four area ranges), Type 5 (`10.10.160.0/19`, `0.0.0.0/0`); no Type 2, no Type 4 |
| 1 | Type 1 (R2, R6), Type 3 (inter-area routes plus default) |
| 2 | Type 1 (R3, R7), **one** Type 3: the default |
| 3 | Type 1 (R4, R8, R10), Type 3 (inter-area routes), Type 7 (`10.10.160.0/19`, `0.0.0.0/0`) |
| 4 | Type 1 (R5, R9), **one** Type 3 default, Type 7 (`0.0.0.0/0`) |

Useful commands: `show ip ospf database nssa-external`, `show ip ospf database external`, `show ip ospf border-routers`, `show ip ospf | include translat`.

## End-to-end tests

| From | Command | Path |
| :--- | :--- | :--- |
| R7 | `ping 10.10.96.10 source Loopback0` | R3, R1, R4, R8, R10 |
| R6 | `ping 10.10.162.1 source Loopback0` | R2, R1, R4, R8, R11 |
| R10 | `ping 8.8.8.8 source Loopback0` | R8, R4, R1, R5, R9, Net |
| R11 | `ping 8.8.8.8 source Loopback1` | R8, R4, R1, R5, R9, Net |

## Failure tests

| Action | Expected result | Lesson |
| :--- | :--- | :--- |
| `shut` on R1 Gi0/2 | Area 3 and the EIGRP domain are cut off | R4 has no backup path (single-homing) |
| `shut` on R9 Gi0/1 | R9 withdraws its default; R1 loses the E2 default. R2/R3 still advertise a default into Areas 1 and 2 | ABR-generated defaults do not track the Internet exit: they point at a blackhole |
| `shut` on R8 Gi0/2 | The `10.10.160.0/19` summary is withdrawn once all EIGRP specifics are gone | Summaries are tied to specifics |
| Remove `area 1 stub` on R6 | The R2 - R6 adjacency drops | Stub flag mismatch |
| Remove `no-summary` on R3 | R7 receives all inter-area Type 3 LSAs | Totally stubby depends on the ABR flag |

## Troubleshooting

| Symptom | Likely cause |
| :--- | :--- |
| Neighbor stuck in INIT or EXSTART | MTU mismatch, or mask mismatch on a `/31` |
| Neighbor drops after Phase 3 | Area-type mismatch between the two ends |
| No neighbor at all | Interface still passive, wrong area, or authentication mismatch |
| R7 has no default route | ABR missing `no-summary`, or R7 not actually stub |
| E2 routes missing in Area 3 or 4 | Expected: those areas are NSSA, externals arrive as Type 7 |
| Type 7 routes not translated | Check the translator with `show ip ospf \| include translat`; the forwarding address must be reachable |
| R1 has no `0.0.0.0/0` | Type 7 to Type 5 translation of the default did not occur: use the fallback on R5 |
| Traffic loops between R9 and R5 | The `/16` static on R9 or the Null0 route on R5 is missing |
| `D*EX` expected on R11 but `D*` seen | Normal: a default from `ip summary-address` is an internal route |

## Wireshark tasks (store captures in `captures/`)

Capture on the named link and apply the display filter. Compare options bits relative to a normal Area 0 link, because exact values can vary with LLS. Wireshark field names differ between versions: if a filter is rejected, use plain `ospf` and read the LSA type in the packet tree.

| Task | Capture on | Filter | Observe |
| :--- | :--- | :--- | :--- |
| Hello options in Area 0 | R1 - R2 | `ospf.msg.hello` | E bit set |
| Hello options in a stub area | R2 - R6 | `ospf.msg.hello` | E bit cleared |
| Hello options in an NSSA | R4 - R8 | `ospf.msg.hello` | E bit cleared, N bit set |
| Totally stubby LSU | R3 - R7 | `ospf.msg.lsupdate` | Only one Type 3 (the default); no other summaries |
| Type 7 origination | R8 - R4 | `ospf.lsa == 7` | `10.10.160.0/19` from R8, P bit set |
| Translation to Type 5 | R1 - R4 | `ospf.lsa == 5` | `10.10.160.0/19` advertised by R4, forwarding address present |
| Internet default | R1 - R5 | `ospf.lsa == 5` | `0.0.0.0` advertised by R5 |
| Point-to-point vs broadcast | any link | `ospf` | Hello destination `224.0.0.5`; no `224.0.0.6` traffic |
