# Ticket AP-20431 – Post-Incident Report

| | |
|---|---|
| **Client** | Ahuriri Physio, Napier |
| **Reported by** | Mere Walker, Practice Manager |
| **Engineer** | Antonio Chadwick |
| **Status** | Resolved |
| **Hints used** | 0 |

---

## 1. Ticket summary

PC-RECEP failed to obtain an ip address and PC-STAFF was able to get an ip address but was unable to reach the public internet or dns. This was caused by three configureation errors on SW1 and R1 made during Friday's change. All three were corrected, and both VLANs now receive DHCP addresses and can reach the internet.

---

## 2. Symptoms observed (before any changes)

| Device | Test | Result |
|---|---|---|
| PC-RECEP | `udhcpc -i eth1` |  *Fail - with message "udhcpc failed to get a DHCP lease"* |
| PC-STAFF | `udhcpc -i eth1` | Lease obtained in 192.168.10.0/24 |
| PC-STAFF | `ping 192.168.10.1` (gateway) | Success |
| PC-STAFF | `ping 8.8.8.8` | *Fail – with message "From 192.168.10.1 icmp_seq=3 Destination Host Unreachable" * |

---

## 3. Steps taken and commands run

1. **SW1 – checked trunk to R1:** `show interfaces trunk`
   ```
   Port     Vlans allowed on trunk 
   Et0/1    10
   ```
   These lines indicated that only VLAN 10 is allowed on the trunk which definitely breaks the path between PC-RECEP and the DHCP server.”.
2. **R1 – checked routing table:** `show ip route`
   ```
   S*    0.0.0.0/0 [1/0] via 203.0.113.5
   ```
   This is misconfiguration route, the correct route should be `S*    0.0.0.0/0 [1/0] via 203.0.113.1`
3. **R1 – checked NAT ACL:** `show access-lists NAT-INSIDE`
   ```
   10 permit 192.168.10.0, wildcard bits 0.0.0.255
   20 permit 192.168.20.0          ← host match only
   ```
The wildcard 0.0.0.0 matches only the single address 192.168.20.0, so traffic from the reception PCs was not selected for NAT. It left R1 with a private source address, and the ISP had no route back to it.
---

## 4. Root causes

| # | Device | Bad config | Effect |
|---|---|---|---|
| 1 | SW1 e0/1 | `switchport trunk allowed vlan 10` | VLAN 20 was not carried on the trunk, so PC-RECEP's DHCP requests never reached R1 |
| 2 | R1 | `ip route 0.0.0.0 0.0.0.0 203.0.113.5` | Default route pointed to a next hop that does not exist (ISP is 203.0.113.1), so no traffic could leave the site |
| 3 | R1 | `permit 192.168.20.0 0.0.0.0` in ACL NAT-INSIDE | Wildcard 0.0.0.0 matched only the single address 192.168.20.0, so reception PCs were not translated by PAT and the ISP could not route replies back to private addresses |

*Sometimes the simplest misconfiguration can take down an entire network. In my case, a single wrong next-hop IP in the default route cut off the whole office.*

> **Not a fault:** PC-RECEP and PC-STAFF needed `udhcpc -i eth1` run manually. The containers do not start a DHCP client by themselves. The DHCP server on R1 was working the whole time.

---

## 5. Fixes applied

**SW1**
```
interface Ethernet0/1
 switchport trunk allowed vlan add 20
```

**R1 – default route**
```
no ip route 0.0.0.0 0.0.0.0 203.0.113.5
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

**R1 – NAT ACL**
```
ip access-list standard NAT-INSIDE
 no permit 192.168.20.0 0.0.0.0
 permit 192.168.20.0 0.0.0.255
```

**Saved:** `write memory` on SW1 and R1

---

## 6. Verification

**DHCP:**.

*PC-STAFF*
```
udhcpc: started, v1.31.1
udhcpc: sending discover
udhcpc: sending select for 192.168.10.11
udhcpc: lease of 192.168.10.11 obtained, lease time 86400
mv: can't rename '/etc/resolv.conf.50': Resource busy
```
*PC-RECEP*
```
udhcpc: started, v1.31.1
udhcpc: sending discover
udhcpc: sending select for 192.168.20.11
udhcpc: lease of 192.168.20.11 obtained, lease time 86400
mv: can't rename '/etc/resolv.conf.39': Resource busy
```
**Default route (R1):**
```
R1#show running-config | include ip route
ip route 0.0.0.0 0.0.0.0 203.0.113.1
ip route vrf clab-mgmt 0.0.0.0 0.0.0.0 Ethernet0/0 172.20.20.1
```
(The `vrf clab-mgmt` route is containerlab's management route and is not part of the lab network.)

**NAT ACL counters (R1):** both subnets now match.
```
Standard IP access list NAT-INSIDE
    10 permit 192.168.10.0, wildcard bits 0.0.0.255 (4 matches)
    20 permit 192.168.20.0, wildcard bits 0.0.0.255 (1 match)
```

**NAT translation for reception (R1)**, captured during a continuous ping from PC-RECEP:
```
Pro  Inside global      Inside local        Outside local    Outside global
icmp 203.0.113.2:1024   192.168.20.11:4501  8.8.8.8:4501     8.8.8.8:1024
```

**End-to-end tests**

| From | To | Result |
|---|---|---|
| PC-STAFF | 192.168.10.1 (gateway) | ✅ |
| PC-RECEP | 192.168.20.1 (gateway) | ✅ |
| PC-STAFF | 8.8.8.8 / 1.1.1.1 | ✅ |
| PC-RECEP | 8.8.8.8 / 1.1.1.1 | ✅ |

---

## 7. Follow-up recommendation for the client

*In my scenario, both vlans can reach each other. I would confirm with the client if this is what they want or they would prefer to have the two vlans isolated from eachother.*

---

## 8. What I learned / what I'd do differently

- *I learned that following the symptoms is better then reading every `show run` output, this only works for this setup as it only contains 2 network nodes that is easy to manage but when you scale that up to 30 routers and switches it will be time consuming*
- *At first I only added the correct static route and did not remove the wrong one. Now I always replace bad configuration instead of adding to it.*

---

## Appendix – Lab platform issue (not part of the ticket)

Cisco IOL startup configs must be named `*.partial.cfg` so containerlab adds them to its baseline config (management IP, SSH). A normal `.cfg` replaces the baseline, and the node loses management access. The first boot's config is saved in NVRAM and reused on redeploy, so after changing config files run `containerlab destroy --cleanup` before deploying again.