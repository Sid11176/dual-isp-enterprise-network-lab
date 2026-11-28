# Architecture

## Purpose

A portfolio project for network and NOC engineering roles. It shows a network that is built, operated, measured and documented: every phase has saved configs and `show` output, every failure is tested with a measured result, and every limitation is stated.

## Scope

A two-site enterprise network in Cisco Packet Tracer 9.0.1.0858.

- **HQ (Cebu):** two edge routers (HQ-R1, HQ-R2), two access switches (HQ-SW1, HQ-SW2) joined by a LACP EtherChannel, six VLANs, HSRP gateways, single-area OSPF between the edge routers, PAT on both edges, DHCP and DNS servers.
- **Branch:** one router (BR-R1) and one switch (BR-SW), three VLANs, local DHCP and PAT, single uplink to ISP-B.
- **Provider side:** ISP-A, ISP-B, ISP-B-UP (an extra upstream that lengthens the ISP-B path) and INET, a simulated internet with a public web server. All routers are Cisco 2911 (the only model that passed the IPsec and BGP tests).
- **Routing design:** eBGP only (AS 64500 to ISP-A and ISP-B). Outbound uses ISP-A because HQ-R1 is HSRP active, with a floating static default as backup. Inbound prefers ISP-A because the ISP-B path is one AS longer. Reasons and Packet Tracer limits: [`design-decisions.md`](design-decisions.md).

Addressing is in [`ip-plan.md`](ip-plan.md), links and interfaces in [`topology.md`](topology.md).

## Tiers

| Tier | Phases | Status |
|---|---|---|
| Core | 0 to 5 | Committed. A complete, shippable repo at the end of Phase 5 |
| Upgrade | 6 (IPsec branch VPN) | Committed |
| Stretch | 7 (branch wireless) | Optional, only if time remains |
| Finish | 8 (README, resume) | Committed |

## Success criteria

- Every phase has exported configs and a `verification/` file. The latest `.pkt` is kept in `pkt/`; earlier stages are recorded in `configs/`, `verification/` and the commit history.
- Five core failover tests (01 to 05) ship with Tier 1; tests 06 and 07 are added after Phase 6. Each records baseline, fault, lost replies and recovery.
- Six to eight incident reports follow one format and cite real captured output.
- The README opens with the diagram and a results table, and lists known limitations honestly.

## Phase gates

A phase is finished only when its gate passes and its evidence is committed.

| Phase | Scope | Gate (summary) |
|---|---|---|
| 0 | Plan and tool check | All feature tests pass or are documented as limitations |
| 1 | VLANs, trunks, LACP, RSTP | VLANs and trunks correct on all three switches; Po1 `(SU)`; HQ-SW1 root; PortFast and BPDU guard on access ports; unused ports parked and shut; same-VLAN ping across Po1 |
| 2 | HSRP, OSPF | R1 Active and R2 Standby on every VLAN; failover to the VIP survives; OSPF FULL; cross-VLAN ping works |
| 3 | eBGP, NAT, DHCP, DNS | All BGP sessions Established; INET prefers ISP-A for 192.0.2.0/24; DHCP, DNS and web work; NAT translations present; branch client online |
| 4 | SSH, ACLs, port security, syslog/NTP/TFTP | ACL test matrix passes; SSH v2 only; syslog, NTP and TFTP work |
| 5 | Failover tests, incidents, runbooks | Five core failover tests recorded with measured loss; incident reports and runbooks written |
| 6 | IPsec branch VPN | `QM_IDLE`; encaps and decaps counters rise; branch PC reaches HQ servers |
| 7 | Branch wireless (optional) | APs visible; laptops associate, get DHCP and ping a wired PC |
| 8 | Polish | README finished; resume and LinkedIn updated |

Full checklists and expected outputs live in the project roadmap; this table is the summary the repo carries.

## Out of scope

iBGP, route-maps, BGP communities and local preference (not supported in Packet Tracer 9), real timers and hardware behavior, and a wireless controller. These are listed as known limitations in the README.
