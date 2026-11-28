# Dual-ISP Enterprise Network Lab

A dual-homed, two-site enterprise network built in Cisco Packet Tracer. An HQ and a branch connect to two simulated ISPs over eBGP, with HSRP gateway redundancy, OSPF internally, LACP EtherChannel, ACL-based segmentation, SSH-only management, and (planned) an IPsec site-to-site VPN and branch WLAN. Every feature is verified with saved command output, and failures are tested and written up as NOC-style incident reports and runbooks.

![Topology](docs/images/topology.png)

> **Status:** Phase 3 complete (28 Nov 2026). Phase 4 is next. Update the table below as each phase is finished.

## Project status

| Phase | Scope | Status |
|---|---|---|
| 0 | IP plan, topology, design decisions, feature tests | Complete |
| 1 | VLANs, trunks, LACP EtherChannel, RSTP | Complete |
| 2 | HSRP gateways, OSPF | Complete |
| 3 | ISP edge, eBGP, NAT, DHCP, DNS | Complete |
| 4 | SSH hardening, ACL policy, port security, syslog/NTP/TFTP | Not started |
| 5 | Failover tests, incident reports, runbooks | Not started |
| 6 | IPsec site-to-site VPN (HQ to branch) | Not started |
| 7 | Branch wireless (WPA2, guest network), optional | Not started |
| 8 | README polish, resume | Not started |

## Skills demonstrated

| Area | What was built |
|---|---|
| Addressing | VLSM plan: HQ 10.10.0.0/23, branch 10.10.2.0/24, infrastructure 10.10.255.0/24, loopbacks and point-to-point links |
| Layer 2 | VLANs, 802.1Q trunks with native VLAN 998, RSTP root placement, PortFast and BPDU Guard, LACP |
| First-hop redundancy | HSRP with preempt on every HQ VLAN, aligned with the STP root |
| Interior routing | Single-area OSPF with passive interfaces |
| Exterior routing | eBGP to two ISPs; inbound path steered by AS-path length (ISP-B-UP upstream), outbound by HSRP plus a floating static default |
| Services | DHCP with relay, DNS, PAT on both edges |
| Security | SSH-only management, named extended ACLs, port security, unused ports parked in VLAN 999 |
| VPN and wireless | IPsec site-to-site, WPA2 branch WLAN (planned) |
| Operations | Failover tests, incident reports, runbooks |

## Design summary

- **Company:** fictional logistics firm with an HQ in Cebu and one branch, dual-homed to ISP-A (primary) and ISP-B (backup).
- **Autonomous systems (documentation range):** company 64500, ISP-A 64496, ISP-B 64497, simulated Internet 64498, ISP-B upstream 64499.
- **Public block:** 192.0.2.0/24, advertised to both ISPs. Outbound uses ISP-A because HQ-R1 is the HSRP active router; a floating static default (AD 250) covers an ISP-A failure. Inbound prefers ISP-A because the ISP-B path crosses one extra AS (ISP-B-UP).
- **Internal addressing:** 10.10.0.0/16 allocated by VLSM (HQ /23, branch /24, infrastructure /24). See [`docs/ip-plan.md`](docs/ip-plan.md).
- **OSPF scope:** internal reachability only (transit link, loopbacks, LAN prefixes). BGP carries everything external and is not redistributed into OSPF.
- **Branch:** single uplink to ISP-B. Private traffic to HQ will use the IPsec tunnel (Phase 6).

Details and trade-offs: [`docs/design-decisions.md`](docs/design-decisions.md). Topology and link table: [`docs/topology.md`](docs/topology.md). Phase 0 feature tests: [`docs/phase0-feature-tests.md`](docs/phase0-feature-tests.md).

## Key results

<!-- TODO (Phase 5): replace each "TBD" with a number you measured, using a continuous ping. -->

| # | Test | Fault | Result |
|---|---|---|---|
| 01 | ISP-A failure | Shut HQ-R1 link to ISP-A | TBD replies lost; traffic shifted to ISP-B; restored on recovery |
| 02 | HSRP active failure | Shut HQ-R1 LAN interface | TBD replies lost; HQ-R2 became Active |
| 03 | EtherChannel member failure | Shut one member port | TBD replies lost |
| 04 | STP root failure | Power off HQ-SW1 | TBD seconds to reconverge; HQ-R2 became HSRP Active |
| 05 | Inbound path check | Shut ISP-A to INET link | INET moved to the ISP-B path and back (TBD) |
| 06 | Branch VPN (Phase 6) | Send traffic branch to HQ | TBD: traffic encrypted |
| 07 | ISP-B failure (Phase 6) | Shut HQ-R2 link to ISP-B | HQ unaffected; branch VPN down (documented limit) |

Test procedures and evidence: [`failover-tests/`](failover-tests/).

## Repository map

```
.
├── docs/              architecture, IP plan, topology (draw.io + PNG), design decisions, feature tests
├── pkt/               latest Packet Tracer file (includes all earlier phases)
├── configs/           one saved config per device
├── verification/      show-command evidence per feature
├── failover-tests/    fault injection with measured recovery
├── incidents/         NOC-style incident reports (index in incidents/README.md)
└── runbooks/          BGP, OSPF, and interface-flapping procedures
```

## How to open the lab

1. Install Cisco Packet Tracer (built with version 9.0.1.0858).
2. Open the latest `.pkt` file in [`pkt/`](pkt/) (currently `v3-bgp-dual-isp.pkt`, Phase 3: ISPs, BGP, NAT, DHCP, DNS). It includes everything built in earlier phases. Each later phase replaces it with the updated file.

3. Device configs are in [`configs/`](configs/) if you prefer reading to opening the lab.
4. Credentials in published configs are placeholders only.

## Known limitations

- **NAT with dual-homing:** replies to traffic that left via ISP-B can arrive via ISP-A, at a router with no translation. Normal traffic follows ISP-A, so this appears only in edge cases. Production designs handle this differently.
- **Packet Tracer BGP is basic:** Packet Tracer 9.0.1.0858 has no route-map, weight, local-pref, update-source, default-originate or iBGP (confirmed in Phase 0 tests). Inbound steering uses AS-path length through an extra upstream router; outbound uses HSRP plus a floating static default. A real network would use iBGP between the edge routers and local preference. Measured times are lab figures, not production figures.
- **ISP-B carries the branch VPN:** an ISP-B outage takes down branch connectivity to HQ.
- **Wireless uses two APs:** Packet Tracer access points support one SSID each. A production design would use one AP or a controller with VLAN-mapped SSIDs.
- **Simulator quirks:** occasional slow ping replies, settings not showing in `show` output, and pings failing despite correct configuration until devices are power-cycled. Save the running config before power-cycling.

## Troubleshooting and operations

- Incident reports: [`incidents/README.md`](incidents/README.md) (added in Phase 5)
- Runbooks (added in Phase 5): [`runbooks/bgp-session-down.md`](runbooks/bgp-session-down.md), [`runbooks/ospf-neighbor-down.md`](runbooks/ospf-neighbor-down.md), [`runbooks/interface-flapping.md`](runbooks/interface-flapping.md)

## What I learned

<!-- TODO (Phase 8): 3 to 5 sentences in your own words: what was hardest, what you would change, what you would add next (for example, route filtering, BGP communities, or automation of config backups). -->

## Author

Pol, licensed electrical engineer (Philippines) working toward a network engineering and NOC career. [https://www.linkedin.com/in/john-paul-villaban-387429190/](https://www.linkedin.com/in/john-paul-villaban-387429190/)
