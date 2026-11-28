# Verification: NAT, DHCP, DNS, and Branch Connectivity (Phase 3)

**Lab:** Dual-ISP Enterprise Network Lab
**Phase:** 3 (ISP connectivity, eBGP, NAT, DHCP, DNS)
**Platform:** Cisco Packet Tracer 9.0.1.0858

## Gate results

| # | Check | Expected | Result |
|---|---|---|---|
| 1 | Client DHCP lease (all HQ VLANs) | Correct address, HSRP gateway, DNS `10.10.1.36` | PASS |
| 2 | `nslookup publicwebsite.com` | `203.0.113.18` | PASS |
| 3 | Public website access | Client loads `publicwebsite.com` | See section 3 |
| 4 | HQ-R1: `show ip nat translations` | Active translations during client traffic | PASS |
| 5 | Branch DHCP | Branch PC receives an address in `10.10.2.0/25` | PASS |
| 6 | Branch connectivity | Pings to `10.10.2.1`, `198.51.100.9`, `8.8.8.8` succeed | PASS |

## 1. DHCP leases at HQ

All leases come from the DNS-DHCP server (`10.10.1.36`) through `ip helper-address` on the HQ subinterfaces (VLAN 50 excluded). Each gateway is the HSRP virtual IP.

| Client | VLAN | Address | Mask | Gateway (HSRP VIP) | DNS |
|---|---|---|---|---|---|
| PC-USER1 | 10 | `10.10.0.11` | `255.255.255.128` | `10.10.0.1` | `10.10.1.36` |
| PC-USER2 | 10 | `10.10.0.13` | `255.255.255.128` | `10.10.0.1` | `10.10.1.36` |
| PC-ADMIN | 20 | `10.10.0.139` | `255.255.255.192` | `10.10.0.129` | `10.10.1.36` |
| PC-IT | 30 | `10.10.0.200` | `255.255.255.224` | `10.10.0.193` | `10.10.1.36` |
| PC-GUEST | 40 | `10.10.1.8` | `255.255.255.224` | `10.10.1.1` | `10.10.1.36` |

### HSRP state (gateways handed out by DHCP)

```
HQ-R1#show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gig         10   110 P Active   local           10.10.0.3       10.10.0.1
Gig         20   110 P Active   local           10.10.0.131     10.10.0.129
Gig         30   110 P Active   local           10.10.0.195     10.10.0.193
Gig         40   110 P Active   local           10.10.1.3       10.10.1.1
Gig         50   110 P Active   local           10.10.1.35      10.10.1.33
Gig         99   110 P Active   local           10.10.1.53      10.10.1.49

HQ-R2#show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gig         10   100   Standby  10.10.0.2       local           10.10.0.1
Gig         20   100   Standby  10.10.0.130     local           10.10.0.129
Gig         30   100   Standby  10.10.0.194     local           10.10.0.193
Gig         40   100   Standby  10.10.1.2       local           10.10.1.1
Gig         50   100   Standby  10.10.1.34      local           10.10.1.33
Gig         99   100   Standby  10.10.1.52      local           10.10.1.49
```

## 2. DNS resolution

```
C:\>nslookup publicwebsite.com
Server: [10.10.1.36]
Address:  10.10.1.36

Non-authoritative answer:
Name:   publicwebsite.com
Address:   203.0.113.18

C:\>nslookup intranet.local
Server: [10.10.1.36]
Address:  10.10.1.36

Non-authoritative answer:
Name:   intranet.local
Address:   10.10.1.37
```

**Result:** Both records resolve to the expected addresses. ("Non-authoritative answer" is how Packet Tracer labels answers from its DNS server.)

## 3. Public website access

![Public website access](../docs/images/public-website.png)

*Screenshot: PC-USER1 browser loading `publicwebsite.com`.*

## 4. NAT / PAT translations

### HQ-R1 (pool `192.0.2.1`)

```
HQ-R1#show ip nat translations
Pro  Inside global     Inside local       Outside local      Outside global
tcp 192.0.2.1:1025     10.10.0.11:1025    203.0.113.18:80    203.0.113.18:80
tcp 192.0.2.1:1026     10.10.0.11:1026    203.0.113.18:80    203.0.113.18:80
tcp 192.0.2.1:1027     10.10.0.11:1027    203.0.113.18:80    203.0.113.18:80
... (11 entries total, all 10.10.0.11 to 203.0.113.18:80)
```

**Result:** PC-USER1 (`10.10.0.11`) is translated to `192.0.2.1` on HQ-R1 on the normal path (ISP-A).

### HQ-R2 (pool `192.0.2.2`)

```
HQ-R2#show ip nat translations
Pro  Inside global     Inside local       Outside local      Outside global
icmp 192.0.2.2:91      10.10.0.11:91      203.0.113.18:91    203.0.113.18:91
icmp 192.0.2.2:92      10.10.0.11:92      203.0.113.18:92    203.0.113.18:92
icmp 192.0.2.2:93      10.10.0.11:93      203.0.113.18:93    203.0.113.18:93
icmp 192.0.2.2:94      10.10.0.11:94      203.0.113.18:94    203.0.113.18:94
tcp 192.0.2.2:1054     10.10.0.11:1054    203.0.113.18:80    203.0.113.18:80
```

**Result:** Traffic from the same client was also translated on HQ-R2 to `192.0.2.2`. This shows that the HQ-R1 to HQ-R2 transit path works and that the transit interface is correctly marked `ip nat inside` (the INC-004 condition). The formal failover procedure is covered in Phase 5.

## 5. Branch DHCP and connectivity

### Branch leases (BR-R1 local pools)

| Client | VLAN | Address | Mask | Gateway | DHCP server | DNS | Domain |
|---|---|---|---|---|---|---|---|
| PC-BR-STAFF | 110 | `10.10.2.11` | `255.255.255.128` | `10.10.2.1` | `10.10.2.1` | `10.10.1.36` | BRANCH_STAFF |
| PC-BR-GUEST | 140 | `10.10.2.135` | `255.255.255.192` | `10.10.2.129` | `10.10.2.129` | `10.10.1.36` | BRANCH_GUEST |

### Branch connectivity

```
C:\>ping 10.10.2.1        Reply from 10.10.2.1: TTL=255   (4/4, 0% loss)
C:\>ping 198.51.100.9     Reply from 198.51.100.9: TTL=254 (4/4, 0% loss)
C:\>ping 8.8.8.8          Reply from 8.8.8.8: TTL=252      (4/4, 0% loss)
```

**Result:** The branch PC reaches its gateway, the ISP-B interface, and the simulated Internet (`8.8.8.8`) through BR-R1 PAT on `Gi0/1`.

## 6. Expected failures (not defects)

| Test | Result | Reason |
|---|---|---|
| Branch `nslookup` to `10.10.1.36` | Times out | The HQ DNS server uses a private address that the ISP path does not route. Expected until the IPsec VPN (Phase 6). |
| HQ PC ping to `10.10.2.1` (branch) | Fails | HQ has no route to `10.10.2.0/24`. This is also resolved by IPsec. |
| `tracert 1.1.1.1` from HQ | Loops between `10.10.0.2` and `10.10.0.3` | Destination is in neither BGP table, and each HQ router's floating default points at the other. See `docs/bgp-policy.md`. |

## 7. Known limitation: dual-ISP NAT

NAT state is not shared between HQ-R1 and HQ-R2. If return traffic enters through the ISP connection of the router that does not own the translation, that router cannot translate the packet. Under normal operation traffic follows ISP-A, so this is uncommon. The design is therefore documented as two independent PAT instances, not a stateful or synchronized dual-ISP NAT.
