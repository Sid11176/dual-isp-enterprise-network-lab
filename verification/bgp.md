# Verification: eBGP (Phase 3)

**Lab:** Dual-ISP Enterprise Network Lab
**Phase:** 3 (ISP connectivity, eBGP, NAT, DHCP, DNS)
**Platform:** Cisco Packet Tracer 9.0.1.0858, Cisco 2911 routers
**Design:** eBGP only (no iBGP, no route-maps). See `docs/bgp-policy.md`.

## Tool notes

- Packet Tracer has no `show ip bgp <prefix>` form, so the full `show ip bgp` table is used as evidence for the path checks below.
- Packet Tracer's `show ip bgp summary` under-reports the number of network entries. This is a display bug and does not affect the sessions or the routing tables.

## Gate results

| # | Check | Expected | Result |
|---|---|---|---|
| 1 | `show ip bgp summary` on all six routers | Every session Established, numeric `State/PfxRcd` | PASS |
| 2 | INET: `show ip bgp` (prefix `192.0.2.0/24`) | Best path via `198.51.100.13`, AS path `64496 64500` | PASS |
| 3 | HQ-R1: `show ip bgp` (prefix `8.8.8.8/32`) | Best path via `198.51.100.1`, AS path `64496 64498` | PASS |
| 4 | HQ-R2: `show ip bgp` (prefix `8.8.8.8/32`) | Best path via `198.51.100.5`, AS path `64497 64499 64498` | PASS |
| 5 | HQ-R1: `show ip route` (route to `8.8.8.8`) | BGP route via `198.51.100.1` | PASS |

## 1. BGP session summary

Only the neighbor tables are shown. Each session is Established (numeric prefix count in `State/PfxRcd`).

### HQ-R1 (AS 64500)

```
HQ-R1#show ip bgp summary
BGP router identifier 10.10.255.1, local AS number 64500
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
198.51.100.1    4 64496      89      68        7    0    0 00:03:19        4
```

### HQ-R2 (AS 64500)

```
HQ-R2#show ip bgp summary
BGP router identifier 10.10.255.2, local AS number 64500
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
198.51.100.5    4 64497     135      81       40    0    0 00:04:48        4
```

### ISP-A (AS 64496)

```
ISP-A#show ip bgp summary
BGP router identifier 198.51.100.13, local AS number 64496
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
198.51.100.2    4 64500      69      70       15    0    0 00:04:19        4
198.51.100.14   4 64498      64      40       15    0    0 00:38:09        4
```

### ISP-B (AS 64497)

```
ISP-B#show ip bgp summary
BGP router identifier 198.51.100.17, local AS number 64497
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
198.51.100.18   4 64499      95      41       29    0    0 00:39:54        4
198.51.100.6    4 64500      53      98       29    0    0 00:05:39        4
```

### ISP-B-UP (AS 64499)

```
ISP-B-UP#show ip bgp summary
BGP router identifier 198.51.100.21, local AS number 64499
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
198.51.100.17   4 64497      65      42       24    0    0 00:40:18        4
198.51.100.22   4 64498      77      41       24    0    0 00:39:17        4
```

### INET (AS 64498)

```
INET#show ip bgp summary
BGP router identifier 8.8.8.8, local AS number 64498
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
198.51.100.21   4 64499      65      41       24    0    0 00:39:32        4
198.51.100.13   4 64496      76      41       24    0    0 00:39:07        4
```

## 2. Inbound path to HQ (INET view of `192.0.2.0/24`)

Command run: `show ip bgp` (full table).

```
INET#show ip bgp
   Network          Next Hop            Metric LocPrf Weight Path
*  192.0.2.0/24      198.51.100.21            0     0     0 64499 64497 64500 i
*>                   198.51.100.13            0     0     0 64496 64500 i
*> 198.51.100.8/30   198.51.100.21            0     0     0 64499 64497 i
*> 203.0.113.16/30   0.0.0.0                  0     0 32768 i
```

**Result:** The best path (`>`) to `192.0.2.0/24` is via ISP-A (`198.51.100.13`), AS path `64496 64500` (length 2). The ISP-B path (`64499 64497 64500`, length 3) is valid but not best because of the extra AS hop through ISP-B-UP.

## 3. Outbound path from HQ-R1 to `8.8.8.8`

```
HQ-R1#show ip bgp
   Network          Next Hop            Metric LocPrf Weight Path
*> 8.8.8.8/32        198.51.100.1             0     0     0 64496 64498 i
*> 192.0.2.0/24      0.0.0.0                  0     0     0 64500 i
*> 198.51.100.8/30   198.51.100.1             0     0     0 64496 64498 64499 64497 i
*> 203.0.113.16/30   198.51.100.1             0     0     0 64496 64498 i
```

**Result:** `8.8.8.8/32` is reached via ISP-A (`198.51.100.1`), AS path `64496 64498`.

## 4. Outbound path from HQ-R2 to `8.8.8.8`

```
HQ-R2#show ip bgp
   Network          Next Hop            Metric LocPrf Weight Path
*> 8.8.8.8/32        198.51.100.5             0     0     0 64497 64499 64498 i
*> 192.0.2.0/24      0.0.0.0                  0     0     0 64500 i
*> 198.51.100.8/30   198.51.100.5             0     0     0 64497 i
*> 203.0.113.16/30   198.51.100.5             0     0     0 64497 64499 64498 i
```

**Result:** `8.8.8.8/32` is reached via ISP-B (`198.51.100.5`), AS path `64497 64499 64498` (one extra AS hop through ISP-B-UP).

## 5. HQ-R1 routing table

```
HQ-R1#show ip route
Gateway of last resort is 10.10.255.66 to network 0.0.0.0

     8.0.0.0/32 is subnetted, 1 subnets
B       8.8.8.8/32 [20/0] via 198.51.100.1, 00:00:00
     10.0.0.0/8 is variably subnetted, 16 subnets, 6 masks
C       10.10.0.0/25 is directly connected, GigabitEthernet0/0.10
...
O       10.10.255.2/32 [110/2] via 10.10.255.66, 01:24:05, GigabitEthernet0/2
C       10.10.255.64/30 is directly connected, GigabitEthernet0/2
...
S       192.0.2.0/24 is directly connected, Null0
     198.51.100.0/24 is variably subnetted, 3 subnets, 2 masks
C       198.51.100.0/30 is directly connected, GigabitEthernet0/1
B       198.51.100.8/30 [20/0] via 198.51.100.1, 00:00:00
     203.0.113.0/30 is subnetted, 1 subnets
B       203.0.113.16/30 [20/0] via 198.51.100.1, 00:00:00
S*   0.0.0.0/0 [250/0] via 10.10.255.66
```

**Result:** The route to `8.8.8.8/32` points through `198.51.100.1` (ISP-A). The floating static default (AD 250) via the HQ-R2 transit address is installed as the only default route, because no ISP sends a default in Packet Tracer.

## 6. Routing-loop observation (unknown destination)

```
C:\>tracert 1.1.1.1
  1   0 ms   0 ms   0 ms   10.10.0.2
  2   0 ms   1 ms   0 ms   10.10.0.3
  3   0 ms   0 ms   1 ms   10.10.0.2
  4   0 ms   0 ms  10 ms   10.10.0.3
  ... (alternates between 10.10.0.2 and 10.10.0.3 until hop 30)
Trace complete.
```

**Result:** A destination that is in neither HQ router's BGP table bounces between HQ-R1 and HQ-R2 until TTL expires, because each router's floating default points at the other. This is a documented limitation of the eBGP-only design. See `docs/bgp-policy.md`.
