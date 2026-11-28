# BGP Policy

**Lab:** Dual-ISP Enterprise Network Lab
**Scope:** eBGP design, path selection, failover, and limitations for Phase 3
**Platform:** Cisco Packet Tracer 9.0.1.0858, Cisco 2911 routers

## 1. Design summary

HQ (AS 64500) is dual-homed to two ISPs. Each HQ router runs one eBGP session to its own ISP. There is no iBGP between HQ-R1 and HQ-R2 and no BGP policy configuration (route-maps, weight, local preference), because Packet Tracer 9.0.1.0858 does not support them. See section 6.

Path preference comes from AS-path length, using a deliberately built topology:

- ISP-A connects to INET directly.
- ISP-B reaches INET only through an extra upstream router (ISP-B-UP, AS 64499), so every path through ISP-B has one more AS hop.

## 2. Autonomous systems and sessions

| Device | AS | Role |
|---|---|---|
| HQ-R1, HQ-R2 | 64500 | Customer (HQ edge) |
| ISP-A | 64496 | Primary provider |
| ISP-B | 64497 | Secondary provider |
| INET | 64498 | Simulated Internet core (`8.8.8.8/32`) |
| ISP-B-UP | 64499 | Upstream of ISP-B (adds AS-path length) |

| Link | Subnet | Side A | Side B |
|---|---|---|---|
| HQ-R1 to ISP-A | `198.51.100.0/30` | HQ-R1 `.2` | ISP-A `.1` |
| HQ-R2 to ISP-B | `198.51.100.4/30` | HQ-R2 `.6` | ISP-B `.5` |
| BR-R1 to ISP-B | `198.51.100.8/30` | BR-R1 `.10` | ISP-B `.9` |
| ISP-A to INET | `198.51.100.12/30` | ISP-A `.13` | INET `.14` |
| ISP-B to ISP-B-UP | `198.51.100.16/30` | ISP-B `.17` | ISP-B-UP `.18` |
| ISP-B-UP to INET | `198.51.100.20/30` | ISP-B-UP `.21` | INET `.22` |

eBGP sessions: HQ-R1 to ISP-A, HQ-R2 to ISP-B, ISP-A to INET, ISP-B to ISP-B-UP, and ISP-B-UP to INET.

## 3. Prefixes advertised

| Origin | Prefix | Purpose |
|---|---|---|
| AS 64500 (both HQ routers) | `192.0.2.0/24` | HQ public block, also the NAT pool range (`192.0.2.1` on R1, `192.0.2.2` on R2). Anchored by `ip route 192.0.2.0 255.255.255.0 Null0` on each router. |
| AS 64497 (ISP-B) | `198.51.100.8/30` | Lets INET learn the branch PAT address (`198.51.100.10`). |
| AS 64498 (INET) | `8.8.8.8/32`, `203.0.113.16/30` | Simulated Internet host and the public web server segment. |

## 4. Path selection

### Inbound (Internet to HQ)

INET sees two paths to `192.0.2.0/24`:

| Path | AS path | Length | Selected |
|---|---|---|---|
| Via ISP-A (`198.51.100.13`) | `64496 64500` | 2 | **Best** |
| Via ISP-B-UP (`198.51.100.21`) | `64499 64497 64500` | 3 | Backup |

BGP chooses the shorter AS path, so inbound traffic prefers ISP-A. If the ISP-A path is withdrawn, INET switches to the ISP-B path automatically.

### Outbound (HQ to Internet)

| Router | Path to `8.8.8.8/32` | AS path |
|---|---|---|
| HQ-R1 | Via ISP-A (`198.51.100.1`) | `64496 64498` |
| HQ-R2 | Via ISP-B (`198.51.100.5`) | `64497 64499 64498` |

Users' default gateway is the HSRP virtual IP, and HQ-R1 is the active router (priority 110 with preempt), so normal user traffic leaves through ISP-A.

### Failover

Each HQ router has a floating static default pointing at the other router across the OSPF transit link (`10.10.255.0/30`):

```
HQ-R1: ip route 0.0.0.0 0.0.0.0 10.10.255.66 250
HQ-R2: ip route 0.0.0.0 0.0.0.0 10.10.255.65 250
```

If HQ-R1 loses its ISP-A routes, traffic follows the default to HQ-R2, which still holds its ISP-B routes and translates with its own NAT pool (`192.0.2.2`). Captured evidence of traffic translated on HQ-R2 is in `verification/nat-dhcp-dns.md`. The formal failover tests are in Phase 5.

## 5. Known limitations of this design

1. **Routing loop for unknown destinations.** Because the ISPs cannot send a default route, each HQ router's only default points at the other. A packet for a destination in neither BGP table (for example `1.1.1.1`) bounces between HQ-R1 and HQ-R2 until TTL expires (`tracert 1.1.1.1` in `verification/bgp.md`). The lab's reachable destinations (`8.8.8.8`, `203.0.113.16/30`, `198.51.100.8/30`) are all in the BGP tables, so they are not affected. In production an ISP default route or a discard route would remove the loop.
2. **Inbound preference is topology-driven.** HQ does not control which ISP the Internet uses to reach it. The preference for ISP-A exists only because INET's ISP-B path has an extra AS hop. Real-world inbound traffic engineering would use AS-path prepending or communities, which Packet Tracer cannot do (see section 6).
3. **No shared NAT state between the HQ routers.** If return traffic arrives through the ISP link of the router that did not create the translation, that router cannot translate it. Normal traffic uses ISP-A, so this is uncommon. The design is documented as two independent PAT instances.
4. **Branch private addresses are not reachable across the ISP path.** The branch reaches the Internet through PAT but cannot reach HQ private addresses (including DNS `10.10.1.36`) until the IPsec tunnel is built.

## 6. Packet Tracer 9.0.1.0858 constraints

Tested in Phase 0. These are unsupported, which is why they are not in the design:

| Feature | Status |
|---|---|
| Route-maps (local preference, AS-path prepend) | Not supported |
| `neighbor weight` | Not supported |
| `bgp default local-preference` | Not supported |
| iBGP between HQ-R1 and HQ-R2 | Not supported ("Only external neighbors are supported") |
| `neighbor update-source` | Not supported |
| `neighbor default-originate` | Not supported |
| `show ip bgp <prefix>` | Not supported (use the full `show ip bgp` table) |
| `show ip bgp summary` network count | Under-reports entries (display bug only) |

Supported and used: `bgp router-id`, `neighbor ... remote-as`, `network ... mask`, and `neighbor next-hop-self`.

## 7. Configuration reference

```
HQ-R1                                   HQ-R2
router bgp 64500                        router bgp 64500
 neighbor 198.51.100.1 remote-as 64496   neighbor 198.51.100.5 remote-as 64497
 network 192.0.2.0 mask 255.255.255.0    network 192.0.2.0 mask 255.255.255.0
```

Router IDs are `10.10.255.1` (HQ-R1) and `10.10.255.2` (HQ-R2), matching their loopbacks, as shown in `show ip bgp summary`. `bgp router-id` was entered on both routers, but Packet Tracer does not write it to the exported startup config when it equals the ID the router already uses.

Full device configurations are in `configs/`.
